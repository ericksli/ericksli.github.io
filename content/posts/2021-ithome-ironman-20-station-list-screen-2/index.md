---
title: "2021 iThome 鐵人賽 Day 20：Station list screen (2)"
tags:
  - 2021 iThome 鐵人賽
  - Android
date: 2021-10-05T00:00:00+08:00
canonicalURL: https://ithelp.ithome.com.tw/articles/10277771
---

> 本篇文章是 [2021 iThome 鐵人賽](https://ithelp.ithome.com.tw/2021ironman)參賽題目「[寫一個列車抵站時間 Android App](https://ithelp.ithome.com.tw/users/20139666/ironman/4661)」的第 20 篇，你可到 iThome [查看原文](https://ithelp.ithome.com.tw/articles/10277771)。
>
> [文章目錄]({{< ref "2021-ithome-ironman" >}})

上一篇我們完成了 `StationListAdapter`，我們現在會繼續車站列表的 UI 部分。

## `StationListViewModel`

首先我們要寫的 class 是 `StationListViewModel`。首先來看看它的基本骨架：

```kotlin
@HiltViewModel
class StationListViewModel @Inject constructor(
    getLinesAndStations: GetLinesAndStationsUseCase,
) : ViewModel(), StationListAdapter.Callback {

    val list: StateFlow<List<StationListItem>> = TODO()
    val launchEtaScreen: Flow<Pair<Line, Station>> = TODO()

    override fun toggleExpanded(line: Line) {
        TODO()
    }

    override fun onClickLineAndStation(line: Line, station: Station) {
        TODO()
    }
}
```

一開首就看到 Dagger Hilt 的 `@HiltViewModel` annotation，它是用來標記 `ViewModel`。如果你想用Dagger Hilt 為你的 `ViewModel` 做 constructor injection 的話，就要為 `ViewModel` 標註 `@HiltViewModel`。加了它就不用再自己寫 `ViewModelProvider.Factory`，Dagger Hilt 會自動為我們打點好。如果你需要在 `ViewModel` 用到 `Context` 的話，可以在 constructor 加上 `@ApplicationContext private val context: Context`，Dagger Hilt 就能為你提供 `Application` `Context`。換句話講，用了 Dagger Hilt 就不需要再用 [`AndroidViewModel`](https://developer.android.com/reference/androidx/lifecycle/AndroidViewModel)。

Constructor 會看到我們之前寫好的 `GetLinesAndStationsUseCase`，因為我們會由那個 use case 取得車站列表然後交予 `RecyclerView` 顯示。至於要 implement 上一篇的 `StationListAdapter.Callback` 是因為 `ViewModel` 的角色是負責接收用戶的輸入動作，經過處理後再以 observer pattern 通知 `Fragment` 改變 UI。而通知改變 UI 的形式我們會用 Kotlin Flow 而不是 `LiveData`。這是因為現在 data binding 已經支援 `StateFlow` 而且 Flow 提供了不少現成的 operator 讓我們可以直接使用，不用我們每次都要 override `[MediatorLiveData](https://developer.android.com/reference/androidx/lifecycle/MediatorLiveData)`。所以上面的 code 會看到我們外露了 `list` 和 `launchEtaScreen` 兩個 Flow 好讓 `Fragment` 接收。`list` 就是用來提交畫面需要顯示的車站列表；`launchEtaScreen` 就是通知 `Fragment` 開啟抵站時間頁。

而 implement `StationListAdapter.Callback` 要實作的 `toggleExpanded` 和 `onClickLineAndStation` 就是放一些 code 令 `list` 和 `launchEtaScreen` 兩個 Flow 能因應用戶的輸入向 `Fragment` 發送最新的狀態。

## 車站列表 flow

```kotlin
private val lineAndStations = flowOf(getLinesAndStations())
private val expandedGroups = MutableStateFlow<Set<Line>>(emptySet())
val list: StateFlow<List<StationListItem>> =
    combine(lineAndStations, expandedGroups) { lineAndStations, expandedGroups ->
        lineAndStations.flatMap { (line, stations) ->
            sequence {
                val isExpanded = expandedGroups.contains(line)
                yield(
                    StationListItem.Group(
                        line = line,
                        isExpanded = isExpanded,
                    )
                )
                if (isExpanded) {
                    yieldAll(stations.map { StationListItem.Child(line = line, station = it) })
                }
            }.toList()
        }
    }.stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(STATE_FLOW_STOP_TIMEOUT_MILLIS),
        initialValue = emptyList(),
    )
```

我們首先要從 `GetLinesAndStationsUseCase` 取得車站列表（以 `Map<Line, Set<Station>` 形式，key 是路綫而 value 是該路綫的車站）。由於 use case 的 `invoke` 只是 return `Map` 而不是 `Flow`，所以我們要先把它轉成 `Flow` (`lineAndStations`)。我們直接用 `flowOf` 就可以了，反正那個 `Map` 是寫死的，不會突然改變。另外，因為 use case 是用 `operator fun invoke()` 寫的，所以我們可以把 use case 的 variable 名後面加上括號就能執行那個 `invoke` function，令整段 code 更為簡潔。

由於我們的列表是有展合功能，所以要記錄各路綫是否展開了車站列表。我們會用一個 `Set` 去記錄那些路綫現在是展開狀態 (`expandedGroups`)。但由於我們最終是要以 `StateFlow` 的形式通知 `Fragment` 最新的 list item，所以這個 `Set` 需要放在 `MutableStateFlow` 內，這樣只要 `expandedGroups` 有變更的話就能觸發更新 `list`。在 `MutableStateFlow` constructor 我們交了這個 flow 的初始值 `emptySet()`，意思是一開始時所有路綫都不會顯示車站名。留意我們用 `MutableStateFlow<Set<Line>>`，意思是要改變那個 `Set` 的內容就要透過 `MutableStateFlow` 的機制去更新，不能直接拿到 `Set` 的 reference 直接改（因為它是 immutable）。我以前看過有人寫了這些東西：

1. `StateFlow<MutableSet<Line>>`
2. `MutableStateFlow<MutableSet<Line>>`

第 1 個就是拿到 `MutableSet` 的 reference 來改內容，但改完是不能向下游通知這個 `MutableSet` 改了；第 2 個是「進可攻退可守」，又可以私下拿 `MutableSet` 的 reference 來改內容，又可以經 `MutableStateFlow` 的機制向下游通知內容已被更改。千萬不要為了節省每次改動內容都要 instantiate 新 object 而寫成這樣，這個寫法會令人混淆，改了 `Set` 但下游又看不見，結果日後要花時間 debug，廢時失事。另外我亦見過有人會把 type 定義成 nullable (`MutableStateFlow<Set<Line>?>`)，這個寫法變相要處理 null 和 empty 兩個情況。如果可以的話不如由 empty 表達沒有東西的意思，不用再增加多個東西處理。而我們用 `StateFlow`/`MutableStateFlow` 而不是單純的 `Flow` 是因為我們想保存當前最新的值，普通的 `Flow` 就是發射了值之後就不會保存最新的值。

接着我們來看看 `list`。它那一大段 code 就是按照當前那些路綫是展開了車站列表而生成對應的 list item 供 `RecyclerView` 顯示，所以我們需要把 `lineAndStations` 和 `expandedGroups` 結合在一起（即是那句 `combine` 的意思）。只要兩者其中一方有變動，那 `combine` 的 lambda 都會被執行。在 lambda 入面我們會收到兩個參數：`lineAndStations` 和 `expandedGroups`。兩個參數雖然跟上面的 `Flow` 和 `MutableStateFlow` 撞名，但 lambda 參數是兩個 flow 當前最新的值，所以 data type 分是 `Map<Line, Set<Station>` 和 `Set<Line>`，不要弄錯。lambda 裏面就是走遍 `Map<Line, Set<Station>` 每一個 `Map.Entry`，看看 `Set<Line>` 是否有這條路綫，有的話就把該路綫的車站都塞進去，做成展開的效果。`flatMap` 的作用就是讓你逐一走進每個 `Map.Entry`，然後每次都 return 一個 `List`，`flatMap` 會將全部的 `List` 合併成一條 `List` 交予下游。而我們用了 `sequence { ... }.toList()` 是因為 [`buildList`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/build-list.html) 現在仍是 experimental。在 `sequence {}` 中如果要提交 item 給 `Sequence` 的話會用到 `yield` 或 `yieldAll`，`yield` 就是提交一個 item 而 `yieldAll` 就是提交多個 item。

`combine` 的下游駁住了 `stateIn` 就是要把 `combine` 生成的 `Flow` 轉換成 `StateFlow`。轉成 `StateFlow` 的原因是如果 `Fragment` 經歷 configuration change 的話就會重新 collect `list`。如果用了普通的 `Flow` 那 `combine` 的一大段 code 就會再次執行，但用了 `StateFlow` 就不會，除非 `lineAndStations` 和 `expandedGroups` 有改動。另外，如果 `combine` 計算出來的東西跟上一次的結果是一樣的話，`StateFlow` 就不會再通知,下游有改動，這是 `StateFlow` 另一大特色。這和 `LiveData` 效果差不多，可以說是為了取代 `LiveData` 而設，所以 data binding 現在支援 Flow 都是支援 `StateFlow`。`stateIn` 要有三樣東西：

1. `scope` 那個 `StateFlow` 值分享的範圍，由於這個 `StateFlow` 是放在 `ViewModel` 內，那它的生死都是跟 `ViewModel` 一致，所以填了 `viewModelScope`
2. `started` 我們填了 `SharingStarted.WhileSubscribed(STATE_FLOW_STOP_TIMEOUT_MILLIS)`，意思是如果一直有人 subscribe (collect) 住這個 `StateFlow` 的話，那 `StateFlow` 的值就能一直被共用，但當最後一個 subscriber 退訂的話，我們會多等 `STATE_FLOW_STOP_TIMEOUT_MILLIS` 的時間後就把 `StateFlow` 的值清除掉（那個 `STATE_FLOW_STOP_TIMEOUT_MILLIS` 的值其實是 `Duration.seconds(5)` 五秒鐘）
3. `initialValue` 初始值，由於這是一個 `List` 那我們就用 `emptyList()` 比較合適

那個 `SharingStarted.WhileSubscribed(STATE_FLOW_STOP_TIMEOUT_MILLIS)` [五秒鐘是 Android Developers 在 Medium 文章內建議的數值](https://medium.com/androiddevelopers/things-to-know-about-flows-sharein-and-statein-operators-20e6ccb2bc74)。它的意思是五秒鐘應該有足夠時間在 configuration change 後重新 subscribe 那個 `StateFlow` ，這樣就不用在每次 configuration change 後都要重新執行上游的 code 計算它的值。

## 按下路綫名稱

按下後，我們要把路綫從 `expandedGroups` 拿走或者是加進去，從而觸發重新計算 `list`。留意我們用了 `update` 而不是用 `value` 來更新 `MutableStateFlow` 的值。這是因為我們需要建基於當前的值才能得知最新的值，用 `update` 就能保障 concurrency。在 `update` lambda 最後 return 的值將會是 `MutableStateFlow` 最新的值。

```kotlin
override fun toggleExpanded(line: Line) {
    viewModelScope.launch {
        expandedGroups.update {
            val newSet = it.toHashSet()
            if (newSet.contains(line)) {
                newSet.remove(line)
            } else {
                newSet.add(line)
            }
            newSet
        }
    }
}
```

## 按下車站名稱

按下後，我們要通知 `Fragment` 開啟抵站時間頁。這次我們用 `Channel` 來做背後發射 data 的原理，然後把 `Channel` 轉換成 `Flow` 供 `Fragment` subscribe。`Channel` 是用來在兩個 coroutine 之間傳送資料，跟 `BlockingQueue` 差不多，我們借用它來表示轉頁動作。這次用 `Flow` 而不是 `StateFlow` 是因為開啟另一頁和顯示 toast 一樣不需要有初始值，亦不需要在 configuration change 後獲取之前的值（如果這樣做就會在 configuration change 後開啟另一頁或顯示 toast 多一次，這不是我們要的效果）。要發射資料到 `Channel` 要用到 `send` 這個 method，留意要在 coroutine scope 內執行。

```kotlin
private val _launchEtaScreen = Channel<Pair<Line, Station>>(Channel.BUFFERED)
val launchEtaScreen: Flow<Pair<Line, Station>> = _launchEtaScreen.receiveAsFlow()

override fun onClickLineAndStation(line: Line, station: Station) {
    viewModelScope.launch {
        _launchEtaScreen.send(line to station)
    }
}
```

現在 `StationListAdapter` 已經完成了。接下來就轉到 `StationListFragment`。

## Fragment layout XML

跟之前的差別就是多了 `RecyclerView` 和由 data binding 改回用 view binding，因為這次用不着。但抵站時間頁會用到 data binding，不用擔心。

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.coordinatorlayout.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <com.google.android.material.appbar.AppBarLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content">

        <com.google.android.material.appbar.MaterialToolbar
            android:id="@+id/topAppBar"
            style="@style/Widget.MaterialComponents.Toolbar.Primary"
            android:layout_width="match_parent"
            android:layout_height="?attr/actionBarSize"
            app:title="@string/app_name" />
    </com.google.android.material.appbar.AppBarLayout>

    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/recyclerView"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:layout_behavior="@string/appbar_scrolling_view_behavior"
        tools:listitem="@layout/station_list_station_item" />
</androidx.coordinatorlayout.widget.CoordinatorLayout>
```

## `StationListFragment`

由於 logic 都是放在 `ViewModel`，所以 `Fragment` 要寫的東西不多，主要都是設定 view binding 和 subscribe `ViewModel` 外露的 `Flow`。

```kotlin
@AndroidEntryPoint
class StationListFragment : Fragment() {
    private val viewModel by viewModels<StationListViewModel>()
    private var _binding: StationListFragmentBinding? = null
    private val binding: StationListFragmentBinding get() = _binding!!
    private var _adapter: StationListAdapter? = null
    private val adapter: StationListAdapter get() = _adapter!!

    @Inject
    lateinit var presenter: LineStationPresenter

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        _binding = StationListFragmentBinding.inflate(inflater, container, false)
        return binding.root
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        _adapter = StationListAdapter(
            lifecycleOwner = viewLifecycleOwner,
            callback = viewModel,
            presenter = this.presenter,
        )
        with(binding.recyclerView) {
            layoutManager = LinearLayoutManager(requireContext())
            adapter = this@StationListFragment.adapter
        }
        observeViewModel()
    }

    private fun observeViewModel() {
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.list.collect {
                    adapter.submitList(it)
                }
            }
        }
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.launchEtaScreen.collect { (line, station) ->
                    findNavController().safeNavigate(
                        StationListFragmentDirections.actionStationListFragmentToEtaFragment(
                            line,
                            station
                        )
                    )
                }
            }
        }
    }

    override fun onDestroyView() {
        super.onDestroyView()
        binding.recyclerView.adapter = null
        _adapter = null
        _binding = null
    }
}
```

在 `observeViewModel`，我們 observe 了 `list` 和 `launchEtaScreen`。留意我們用了 `viewLifecycleOwner.lifecycleScope.launch` 又用了 `viewLifecycleOwner.repeatOnLifecycle` 包住那句 `viewModel.someFlow.collect`：

```kotlin
viewLifecycleOwner.lifecycleScope.launch {
    viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.someFlow.collect { ... }
    }
}
```

這個寫法是按照 [Android 的建議](https://medium.com/androiddevelopers/a-safer-way-to-collect-flows-from-android-uis-23080b1f8bda)來寫。因為包住 `viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED)` 的 coroutine 會在 `onStop` 和 `onStart` 之間暫停接收，從而避免在不適當的時機接觸到 view。

`list` 的部分我們只需要 call `ListAdapter.submitList` 就可以了，它會計算那些 list item 需要更新。而 `launchEtaScreen` 就是 call `findNavController().navigate()` 跳去抵站時間頁。由於我們用了 Save Args，所以用了 `StationListFragmentDirections.actionStationListFragmentToEtaFragment` 來保證 type safe 和沒有遺漏 Fragment argument。但我們 code 用了 `safeNavigate` 而非 `navigate`，原因是避免用戶在按下轉頁按鈕後畫面尚未顯示到下一頁時用戶再次按動轉頁按鈕從而 app crash。因為 Navigation component 覺得 `findNavController().navigate()` 後就已經轉到新一頁，即使畫面尚未完成轉頁。所以用戶重按轉頁按鈕時 Navigation component 就會發現當前頁面並沒有這個導航方式，因而報錯。要避免這個情況我們可以參考 [Nnabueze Uhiara](https://nezspencer.medium.com/navigation-components-a-fix-for-navigation-action-cannot-be-found-in-the-current-destination-95b63e16152e) 提供的 `safeNavigate`：

```kotlin
fun NavController.safeNavigate(direction: NavDirections) {
    currentDestination?.getAction(direction.actionId)?.run {
        navigate(direction)
    }
}
```

## 小結

來到這裏車站列表頁已經完成了。本篇介紹了 `ViewModel` 的定位：提供 `Flow` 供 `Fragment` subscribe 來更新 UI 和提供 method 供 `Fragment` 通知 `ViewModel` 用戶做了甚麼動作，從而讓 `ViewModel` 執行適當的動作回應，例如用戶按下按鈕後會 call use case 並將新的狀態以 `Flow` 通知 `Fragment`。另外，我們用 `Channel` 做出 [`SingleLiveEvent` 的效果](https://medium.com/androiddevelopers/livedata-with-snackbar-navigation-and-other-events-the-singleliveevent-case-ac2622673150)。最後還介紹了 Navigation component 在轉頁時的陷阱。如果想對 `ViewModel` 的定位有更深入的了解可以看看「[Don't let ViewModel know about framework level dependencies](https://blog.shreyaspatil.dev/dont-let-viewmodel-knew-about-framework-level-dependencies)」一文。

完整的 code 可以到 [GitHub repo 查閱](https://github.com/ericksli/eta-demo/tree/main/app/src/main/java/net/swiftzer/etademo/presentation/stationlist)。下一篇我們會開始做抵站時間頁，屆時會有更多 `ViewModel` 和 `Flow` 的示範。

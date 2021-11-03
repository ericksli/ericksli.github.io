---
title: "2021 iThome 鐵人賽 Day 23：ETA screen (2)"
tags:
  - 2021 iThome 鐵人賽
  - Android
date: 2021-10-08T00:00:00+08:00
canonicalURL: https://ithelp.ithome.com.tw/articles/10279171
---

> 本篇文章是 [2021 iThome 鐵人賽](https://ithelp.ithome.com.tw/2021ironman)參賽題目「[寫一個列車抵站時間 Android App](https://ithelp.ithome.com.tw/users/20139666/ironman/4661)」的第 23 篇，你可到 iThome [查看原文](https://ithelp.ithome.com.tw/articles/10279171)。
>
> [文章目錄]({{< ref "2021-ithome-ironman" >}})

## `SavedStateHandle`

不知道大家有沒有發現在「ETA Screen (1)」貼出來的 `EtaViewModel` 的 constructor 有一個 `SavedStateHandle`？在繼續完成餘下的錯誤情景前，我們先看看 `SavedStateHandle` 是甚麼。

大家看過不少有關 Architecture Components 講關於 `ViewModel` 的特色時都一定會提到 `ViewModel` 內的 variable 能在 configuration change 後都能保持着，因為 `Activity` 或者 `Fragment` 在 configuration change 後經 `ViewModelProvider` 拿到的 `ViewModel` 是之前的 instance，不像其他 `View` 要重新 instantiate 過，用了它就好像解決到大部分 Android 開發麻煩的問題。但有沒有考慮到如果裝置記憶體有限時要 kill app 然後用戶從 recent screen 開啟之前用過的 app 又會怎樣？很多人都忽略了這個環節，可能現在的裝置比以前多很多 RAM，少了用戶明顯為意到的 kill app 情況。但有時在一些舊裝置仍有可能發生。例如你的 app 用 activity result 打開了預設的相機 app 拍照，拍完就返回你的 app。但有可能在返回你的 app 那時整個 app 已經被系統殺掉而重新啟動，但因為你沒有特別處理這個情況導致拍照後的流程中斷了。

要處理這個問題，以往都是建議大家用 `Activity`/`Fragment` 的 `onSaveInstanceState` callback 來儲存目前的 state 然後從 `onCreate` 或者 `onRestoreInstanceState`/`onViewStateRestored` callback 取回系統 kill app 前的 state。但現在有了 `ViewModel` 都會把 state 放入去而不是放在 `Activity`/`Fragment`，如果要把 `ViewModel` 的 state 交去 `Activity`/`Fragment` 路綫就會很迂迴。有見及此就出了 `SavedStateHandle`。`SavedStateHandle` 是從 `ViewModel` 的 constructor 取得，可以經它存取 key value 組合，就像 `Bundle` 一樣 (`savedStateHandle["xxx"]`)。但不同的是它除了取得 value 外，還可以取得 value 的 `LiveData` (`savedStateHandle.getLiveData("xxx", "default value")`)，好讓你把 state 直接放入去 `SavedStateHandle`。這做法有別於以往，因為 `ViewModel` 沒有那些 `onSaveInstanceState`、`onRestoreInstanceState` callback。由於系統能隨時 kill app，所以就要把 `SavedStateHandle` 視作儲存當前 state 的地方，而不是待系統 kill app 前一刻才放資料進去。

`SavedStateHandle` 另一個用途是用來取得 `Activity` 的 intent extras 和 `Fragment` argument。獲取方式跟之前的 `savedStateHandle["xxx"]` 一樣（`"xxx"` 是 intent extras/argument 的 key）。但我們已經用了 Navigation Component 的 Safe Args plugin，用 plugin 就是為了 `Bundle` 做到 type-safe，現在 `SavedStateHandle` 要走回頭路要自已寫 key 不覺得有點怪嗎？但其實是可以自己寫一個  delegate 將 `SavedStateHandle` 內儲存的 key-value pair 變成 Safe Args plugin 生成的 argument class 的 object。

```kotlin
@MainThread
inline fun <reified Args : NavArgs> navArgs(savedStateHandle: SavedStateHandle) =
    NavArgsLazy(Args::class) {
        val pairs = savedStateHandle.keys()
            .map { Pair<String, Any?>(it, savedStateHandle[it]) }
            .toTypedArray()
        bundleOf(*pairs)
    }
```

用法就是在「ETA Screen (1)」貼出來的 code 找到，以下是節錄：

```kotlin
private val args by navArgs<EtaFragmentArgs>(savedStateHandle)

// 直接用 argument 的值作為 StateFlow 的初始值
val line: StateFlow<Line> = MutableStateFlow(args.line)
val station: StateFlow<Station> = MutableStateFlow(args.station)
```

而我們有一個功能是讓用戶改變排序方式，這個設定我們會放在 `SavedStateHandle` 內：

```kotlin
// SavedStateHandle 放排序方式的 key
private const val SORT_BY = "sort_by"

// 把 SavedStateHandle 內的值以 LiveData 形式取出
// 但因為我們取得班次列表是用 Flow，所以要轉為 Flow 並由 Int 轉為 SortBy enum
private val sortedBy = savedStateHandle.getLiveData(SORT_BY, 0).asFlow()
        .map { GetEtaUseCase.SortBy.values()[it] }

// 用戶按下 MaterialToolbar 內的排序選單項目會觸發的 ViewModel method
fun toggleSorting() {
    val values = GetEtaUseCase.SortBy.values()
    val oldSortByOrdinal: Int = savedStateHandle.get<Int?>(SORT_BY) ?: 0
    savedStateHandle[SORT_BY] = (oldSortByOrdinal + 1) % values.size
}
```

`LiveData.asFlow` 這個 extension function 是由 [AndroidX Lifecycle 提供](https://developer.android.com/reference/kotlin/androidx/lifecycle/package-summary#asflow)，它還提供了 `Flow` 轉 `LiveData` 的 extension function。（那個五秒 timeout 就是由這裏起源的）

## 各式錯誤狀態

在前篇我們看過下面這張圖，入面有好幾個錯誤狀態，我們先做那三個全頁顯示的錯誤狀態。

為方便分辨錯誤的狀態，我們為 domain 的 `EtaResult` 內跟錯誤相關的 class 都幫它 implement 另一個 sealed interface `EtaFailResult`：

```kotlin
sealed interface EtaFailResult

sealed interface EtaResult {
    data class Success(
        val schedule: List<Eta> = emptyList(),
    ) : EtaResult {
        // ...
    }

    object Delay : EtaResult, EtaFailResult

    data class Incident(
        val message: String = "",
        val url: String = "",
    ) : EtaResult, EtaFailResult

    object TooManyRequests : EtaResult, EtaFailResult

    object InternalServerError : EtaResult, EtaFailResult

    data class Error(val e: Throwable?) : EtaResult, EtaFailResult
}
```

另外，由於三款錯誤都是一段文字再加一個按鈕，所以我們乾脆在 layout XML 共用這幾個元素。

接下來就回到 `EtaViewModel` 的部分。XML layout 內的 `NestedScrollView` 包含了顯示錯誤的 UI，我們已經用 data binding 跟 `EtaViewModel` 的 `showError` 綁定是否顯示。以下是 `showError` 的內容：

```kotlin
val showError = etaResult
    .map { it is Loadable.Loaded && it.value is EtaFailResult }
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(STATE_FLOW_STOP_TIMEOUT_MILLIS),
        initialValue = false,
    )
```

可以看到我們標註了 `EtaFailResult` 後整個寫法變得簡單了，毋須再把 `EtaResult` 的 class 逐一判斷。

然後是 `NestedScrollView` 內的 `TextView` 和 `Button` data binding 用到的 `Flow`。首先是用來決定顯示甚麼錯誤訊息的 `errorResult`：

```kotlin
val errorResult = loadedEtaResult
    .filterIsInstance<EtaFailResult>()
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(STATE_FLOW_STOP_TIMEOUT_MILLIS),
        initialValue = EtaResult.InternalServerError,
    )
```

至於實際顯示甚麼文字的部分我們會交由 `EtaPresenter` 負責控制：

```kotlin
@ActivityScoped
class EtaPresenter @Inject constructor(@ActivityContext context: Context) {
    private val res = context.resources

    fun mapErrorMessage(result: EtaFailResult): String = when (result) {
        EtaResult.Delay -> res.getString(R.string.delay)
        is EtaResult.Error,
        EtaResult.InternalServerError,
        EtaResult.TooManyRequests,
        -> res.getString(R.string.error)
        is EtaResult.Incident -> result.message
    }
}
```

因為 data binding 寫的 expression 是要用 Java、只可以單行再加上本身是 XML 檔就有一堆字符要 escape 過才可以寫到，所以遇上比較複雜的 expression 我都會另外找個地方寫個 function 讓 layout XML call，否則會比多層 Excel formula 包圍的 expression 更難看（人家還會把開關括號配上不同顏色方便你看）。只要 function 的參數有 `LiveData` 或者 `StateFlow` data binding 都能自動更新（緊記要設定好 data binding 的 `lifecycleOwner`）。如果不喜歡開新 class 放這些東西可以把它放去 `Activity` 或者 `Fragment` 內，然後在 layout XML 加上那個 `Activity` 或者 `Fragment` 的 `<variable>`。下面是這個 `TextView` 的 layout XML：

```xml
<com.google.android.material.textview.MaterialTextView
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:text="@{etaPresenter.mapErrorMessage(viewModel.errorResult)}"
    android:textAlignment="center"
    android:textAppearance="?textAppearanceBody1"
    tools:text="@string/delay" />
```

然後來到文字下面的按鈕，我們為了簡化寫法，所以分開「Try again」和「View detail」兩個按鈕。

```xml
<com.google.android.material.button.MaterialButton
    isVisible="@{viewModel.showViewDetail}"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:layout_marginTop="16dp"
    android:onClick="@{() -> viewModel.viewIncidentDetail()}"
    android:text="@string/incident_cta" />

<com.google.android.material.button.MaterialButton
    isVisible="@{viewModel.showTryAgain}"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:layout_marginTop="16dp"
    android:onClick="@{() -> viewModel.refresh()}"
    android:text="@string/try_again" />
```

下面是控制是否顯示那兩個按鈕的 `StateFlow`：

```kotlin
val showViewDetail = loadedEtaResult.map { it is EtaResult.Incident }.stateIn(
    scope = viewModelScope,
    started = SharingStarted.WhileSubscribed(STATE_FLOW_STOP_TIMEOUT_MILLIS),
    initialValue = false,
)
val showTryAgain = loadedEtaResult.map {
    when (it) {
        EtaResult.Delay,
        is EtaResult.Incident,
        is EtaResult.Success -> false
        is EtaResult.Error,
        EtaResult.InternalServerError,
        EtaResult.TooManyRequests -> true
    }
}.stateIn(
    scope = viewModelScope,
    started = SharingStarted.WhileSubscribed(STATE_FLOW_STOP_TIMEOUT_MILLIS),
    initialValue = false,
)
```

其實這些 `StateFlow` 的出現都是因為 data binding 不能寫太複雜的 code，所以就把這些 logic 放在 `ViewModel` 內，然後 data binding 只需要接到 Boolean 值來控制 visibility 是 `VISIBLE` 還是 `GONE`。

然後來到按下兩個按鈕的 click listener。「Try again」那個的做法非常簡單，只需向 `triggerRefresh` 發射東西就能踢起 `etaResult` 整串東西：

```kotlin
fun refresh() {
    viewModelScope.launch {
        triggerRefresh.send(Unit)
    }
}
```

而「View detail」要做的東西是開啟瀏覽器前往 API response 提供的網址。同樣地，因為開啟瀏覽器不應在 configuration change 後再次接收到上次的值，所以我們要用 `Channel` 來送知開啟的網址。

```kotlin
private val _viewIncidentDetail = Channel<String>(Channel.BUFFERED)
val viewIncidentDetail: Flow<String> = _viewIncidentDetail.receiveAsFlow()

fun viewIncidentDetail() {
    val result = etaResult.value
    if (result !is Loadable.Loaded) return
    if (result.value !is EtaResult.Incident) return
    viewModelScope.launch {
        _viewIncidentDetail.send(result.value.url)
    }
}
```

在 `EtaFragment` 我們會 collect `viewIncidentDetail`：

```kotlin
viewLifecycleOwner.lifecycleScope.launch {
    viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.viewIncidentDetail.collect {
            try {
                requireActivity().startActivity(Intent(Intent.ACTION_VIEW).apply {
                    data = Uri.parse(it)
                })
            } catch (e: ActivityNotFoundException) {
                Toast.makeText(
                    requireContext(),
                    R.string.cannot_launch_browser,
                    Toast.LENGTH_SHORT
                ).show()
            }
        }
    }
}
```

留意不是所有 Android 裝置都有內置瀏覽器，為謹慎起見我們要 catch `ActivityNotFoundException`，並顯示 toast 提示用戶不能開啟瀏覽器。

## 小結

來到這裏應該可以順利地執行 app 並運用上篇介紹的 [Whistle proxy server](https://github.com/avwo/whistle) 造出不同的 response 來測試這頁。這篇我們討論了 `SavedStateHandle` 和避免在 layout XML 寫複雜 binding expression 的方法。完整的 code 可以在 [GitHub repo](https://github.com/ericksli/eta-demo/tree/main/app/src/main/java/net/swiftzer/etademo/presentation/eta) 找到，下一篇我們會把這頁做成自動更新，不用先出去再進入班次頁才能看到最新內容。

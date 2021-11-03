---
title: "2021 iThome 鐵人賽 Day 25：ETA screen (4)"
tags:
  - 2021 iThome 鐵人賽
  - Android
date: 2021-10-10T00:00:00+08:00
canonicalURL: https://ithelp.ithome.com.tw/articles/10279995
---

> 本篇文章是 [2021 iThome 鐵人賽](https://ithelp.ithome.com.tw/2021ironman)參賽題目「[寫一個列車抵站時間 Android App](https://ithelp.ithome.com.tw/users/20139666/ironman/4661)」的第 25 篇，你可到 iThome [查看原文](https://ithelp.ithome.com.tw/articles/10279995)。
>
> [文章目錄]({{< ref "2021-ithome-ironman" >}})

現在來到整個 app 最後一個功能：錯誤 banner。這個 banner 出現的目的是因為鐵路隧道沿綫的電話上網訊號都接收得不太好（因為太多人同時在用），很容易出現錯誤。如果自動更新時有不能上網的錯誤會彈出全頁錯誤畫面的話效果就不太好。所以就設計了 banner 形式的顯示錯誤方式。

## Layout XML

現在先看看 `EtaFragment` layout XML 的改動：

```xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools">

    <!-- 略 -->

    <androidx.coordinatorlayout.widget.CoordinatorLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <com.google.android.material.appbar.AppBarLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content">

            <!-- 略 -->
        </com.google.android.material.appbar.AppBarLayout>

        <LinearLayout
            isVisible="@{viewModel.showEtaList}"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:divider="?android:listDivider"
            android:orientation="vertical"
            android:showDividers="middle"
            app:layout_behavior="@string/appbar_scrolling_view_behavior">

            <!-- banner 部分 -->
            <LinearLayout
                isVisible="@{viewModel.showErrorBanner}"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:gravity="center_vertical"
                android:orientation="horizontal"
                android:paddingStart="16dp"
                android:paddingTop="8dp"
                android:paddingEnd="16dp"
                android:paddingBottom="8dp"
                tools:visibility="visible">

                <com.google.android.material.textview.MaterialTextView
                    android:layout_width="0dp"
                    android:layout_height="wrap_content"
                    android:layout_marginEnd="16dp"
                    android:layout_weight="1"
                    android:text="@{etaPresenter.mapErrorMessage(viewModel.errorResult)}"
                    android:textAlignment="viewStart"
                    android:textAppearance="?textAppearanceBody1"
                    android:textColor="@color/design_default_color_error"
                    tools:text="@string/error" />

                <com.google.android.material.button.MaterialButton
                    style="@style/Widget.MaterialComponents.Button.TextButton"
                    android:layout_width="wrap_content"
                    android:layout_height="wrap_content"
                    android:onClick="@{() -> viewModel.refresh()}"
                    android:text="@string/try_again" />
            </LinearLayout>

            <!-- 原先的 RecyclerView -->
            <androidx.recyclerview.widget.RecyclerView
                android:id="@+id/recyclerView"
                android:layout_width="match_parent"
                android:layout_height="0dp"
                android:layout_weight="1"
                tools:listitem="@layout/eta_list_eta_item" />
        </LinearLayout>

        <!-- 原先的全頁錯誤顯示 -->
        <androidx.core.widget.NestedScrollView
            isVisible="@{viewModel.showFullScreenError}"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:fillViewport="true"
            app:layout_behavior="@string/appbar_scrolling_view_behavior">

            <!-- 略 -->
        </androidx.core.widget.NestedScrollView>

        <!-- 略 -->
    </androidx.coordinatorlayout.widget.CoordinatorLayout>
</layout>
```

主要是多了一個 `LinearLayout` 來放 banner 及原有的 `RecyclerView`。另外就是把 `viewModel.showError` 改名為更明顯的 `viewModel.showFullScreenError`。

## `EtaViewModel`

由於 banner 顯示的同時亦都要顯示上一次成功載入的班次，所以原先 `etaResult` 所提供的 `TimedValue<Loadable<EtaResult>>` 並不能同時提供現在的錯誤和上一次成功載入的結果。我們的目標是要把這兩樣資訊都由 `etaResult` 一併提供，因為這樣做的話比起分兩個 `StateFlow` 存放這兩樣東西更易控制。

由於要用一個 `StateFlow` 表達兩樣東西，我們需要造一個專門的 data class `CachedResult` 表達它。雖然 Kotlin Standard Library 有 `Pair` 可以用，但過幾個月再看這段 code 的話應該都忘了是甚麼意思。

```kotlin
private data class CachedResult(
    val lastSuccessResult: EtaResult.Success? = null,
    val lastFailResult: EtaFailResult? = null,
    val currentResult: TimedValue<Loadable<EtaResult>> = TimedValue(
        value = Loadable.Loading,
        updatedAt = Instant.EPOCH
    ),
)
```

而 `etaResult` 的 type 亦都改為 `StateFlow<CachedResult>`。

```kotlin
private val etaResult: StateFlow<CachedResult> = triggerRefresh
    .consumeAsFlow()
    .flatMapLatest {
        flowOf(
            flowOf(TimedValue(value = Loadable.Loading, updatedAt = clock.instant())),
            combine(
                language,
                line,
                station,
                sortedBy,
            ) { language, line, station, sortedBy ->
                TimedValue(
                    value = Loadable.Loaded(getEta(language, line, station, sortedBy)),
                    updatedAt = clock.instant(),
                )
            }.onEach {
                // schedule the next refresh after loading
                autoRefreshScope.launch {
                    delay(AUTO_REFRESH_INTERVAL)
                    triggerRefresh.send(Unit)
                }
            },
        ).flattenConcat()
    }
    .scan(CachedResult()) { acc, currentValue ->
        if (currentValue.value is Loadable.Loaded) {
            val currentResult = currentValue.value.value
            CachedResult(
                lastSuccessResult = if (currentResult is EtaResult.Success) currentResult else acc.lastSuccessResult,
                lastFailResult = if (currentResult is EtaFailResult) currentResult else null,
                currentResult = currentValue,
            )
        } else {
            acc.copy(currentResult = currentValue)
        }
    }
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(STATE_FLOW_STOP_TIMEOUT_MILLIS),
        initialValue = CachedResult(),
    )
```

前段的載入中和 call API 的部分是和上一篇沒分別，分別在於之後多了 `scan` 和 `stateIn` 的 `initialValue` 轉做 `CachedResult()`。

`scan` 就是本篇的關鍵所在。如果翻查 [KDoc](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/scan.html) 的話，會看到以下示例：

> `flowOf(1, 2, 3).scan(emptyList<Int>()) { acc, value -> acc + value }.toList()`
will produce `[], [1], [1, 2], [1, 2, 3]]`

這個 operator 有另一個別名叫 `runningFold`，看到 [`fold`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/fold.html) 就知道它是 [`reduce`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/reduce.html) 的意思。`reduce` 就是把一連串的元素壓成一個元素，Kotlin collection 的 `fold` 跟 `reduce` 差別就是 `fold` 可以提供 lambda 內 `accumulator` 的初始值，而 `reduce` lambda 在第一次 call 的時候 `accumulator` 和 `value` 參數會提供 collection 的第一二項元素。回到 `scan` 的部分，從上面的例子看到它的初始值是 empty list，每次都把 `acc`（上一次的結果）駁長，最終生成了一個包含所有元素的 list。我們可以借助這個 operator 把先前成功載入的結果和目前最新的結果整合成一個值交到下游。所以 `scan` 那個 lambda 如果現在是載入中的話那就用 data class 的 `copy` 保留 `CachedResult` 的 `lastSuccessResult` 和 `lastFailResult`。但當載入完成後就把 `CachedResult` 的所有內容換走。

由於我們這次改了 `etaResult` 的 type，所以又會像上一篇一樣把所有用到 `etaResult` 的地方修改一遍。但直接用 `etaResult` 會比較麻煩，故此引入了另一個中途 `StateFlow` 和 enum 來表達現在要顯示的畫面：

```kotlin
private enum class ScreenState {
    LOADING,
    ETA,
    FULL_SCREEN_ERROR,
    ETA_WITH_ERROR_BANNER,
}

private val screenState = etaResult.map {
    when (it.currentResult.value) {
        is Loadable.Loaded -> {
            when (it.currentResult.value.value) {
                EtaResult.Delay,
                is EtaResult.Incident -> ScreenState.FULL_SCREEN_ERROR
                is EtaResult.Error,
                EtaResult.InternalServerError,
                EtaResult.TooManyRequests -> if (it.lastSuccessResult != null) {
                    ScreenState.ETA_WITH_ERROR_BANNER
                } else {
                    ScreenState.FULL_SCREEN_ERROR
                }
                is EtaResult.Success -> ScreenState.ETA
            }
        }
        Loadable.Loading -> when (it.lastFailResult) {
            EtaResult.Delay,
            is EtaResult.Incident -> ScreenState.FULL_SCREEN_ERROR
            is EtaResult.Error,
            EtaResult.InternalServerError,
            EtaResult.TooManyRequests -> if (it.lastSuccessResult != null) {
                ScreenState.ETA_WITH_ERROR_BANNER
            } else {
                ScreenState.FULL_SCREEN_ERROR
            }
            null -> if (it.lastSuccessResult != null) {
                ScreenState.ETA
            } else {
                ScreenState.LOADING
            }
        }
    }
}
```

有了這個 `screenState` 判斷何時要顯示那種畫面我們就容易修改其餘的地方。

```kotlin
private val loadedEtaResult = etaResult
    .map { it.currentResult.value }
    .filterIsInstance<Loadable.Loaded<EtaResult>>()
    .map { it.value }
val showLoading: StateFlow<Boolean> = screenState.map { it == ScreenState.LOADING }
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(STATE_FLOW_STOP_TIMEOUT_MILLIS),
        initialValue = true,
    )
val showFullScreenError = screenState.map { it == ScreenState.FULL_SCREEN_ERROR }
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(STATE_FLOW_STOP_TIMEOUT_MILLIS),
        initialValue = false,
    )
val showErrorBanner = screenState.map { it == ScreenState.ETA_WITH_ERROR_BANNER }
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(STATE_FLOW_STOP_TIMEOUT_MILLIS),
        initialValue = false,
    )
val showEtaList = screenState
    .map { it == ScreenState.ETA || it == ScreenState.ETA_WITH_ERROR_BANNER }
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(STATE_FLOW_STOP_TIMEOUT_MILLIS),
        initialValue = false,
    )
val etaList = etaResult
    .map { it.lastSuccessResult?.schedule.orEmpty() }
    .combine(sortedBy) { schedule, sortedBy ->
        // 略
    }
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(STATE_FLOW_STOP_TIMEOUT_MILLIS),
        initialValue = emptyList(),
    )

fun startAutoRefresh() {
    autoRefreshScope.launch {
        // 略
    }
}
fun viewIncidentDetail() {
    val result = etaResult.value.currentResult.value
    if (result !is Loadable.Loaded) return
    if (result.value !is EtaResult.Incident) return
    // 略
}
```

## 小結

這樣我們就完成了顯示錯誤 banner 功能了。本篇主要是示範了用 `scan` operator 把當前和以前的值整合成一個 `Flow` 內，另外亦用 enum 表達目前整頁的狀態。其實我們不知不覺間已經將整頁的狀態由一個 `StateFlow` 表達出來（就是 `etaResult`），只不過我們另外衍生一堆零碎 `StateFlow` 供 data binding 用。其實我們可以將 `etaResult` 和 `screenState` 兩個 `Flow` 二合為一，屆時 `ScreenState` 不會是 enum 而是 sealed interface，原先的 `LOADING`、`ETA`、`FULL_SCREEN_ERROR`、`ETA_WITH_ERROR_BANNER` 就會變成一個個 object expression 和 data class。這樣就做到了 MVI (Model-View-Intent) 的 state reducer 風味的東西，而且又有類似 unidirectional data flow 的機制。現在那些 `showLoading`、`showFullScreenError`、`showEtaList` 之類的 `StateFlow` 都是為了避免在 layout XML 寫太複雜的邏輯（因為不方便做 unit testing 和它本身是 XML 所以某些字符要 escape）而造出來的。日後改用 Jetpack Compose 寫 UI 的話相信可以減省到只外露 `ScreenState` sealed interface 的 `Flow` 就足夠了。

完整的 code 可以到 [GitHub repo](https://github.com/ericksli/eta-demo/tree/main/app/src/main/java/net/swiftzer/etademo/presentation/eta) 查閱，下一篇我們會寫 `ViewModel` 的 unit test case。

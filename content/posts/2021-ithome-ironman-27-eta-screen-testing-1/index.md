---
title: "2021 iThome 鐵人賽 Day 27：ETA screen testing (1)"
tags:
  - 2021 iThome 鐵人賽
  - Android
date: 2021-10-12T00:00:00+08:00
canonicalURL: https://ithelp.ithome.com.tw/articles/10280812
---

> 本篇文章是 [2021 iThome 鐵人賽](https://ithelp.ithome.com.tw/2021ironman)參賽題目「[寫一個列車抵站時間 Android App](https://ithelp.ithome.com.tw/users/20139666/ironman/4661)」的第 27 篇，你可到 iThome [查看原文](https://ithelp.ithome.com.tw/articles/10280812)。
>
> [文章目錄]({{< ref "2021-ithome-ironman" >}})

上一篇我們完成了車站列表頁的 ViewModel 和 Presenter 的 unit test。現在轉過去寫班次頁的 unit test。

## `EtaPresenter`

首先我們寫 `EtaPresenter` 的 test。這次我們來點新意思：使用 JUnit 4 的 parameterized test，寫法跟之前 `LineStationPresenterTest` 很不同。Parameterized test 的基本格式是：

1. 提供一堆輸入和預期輸出值的 `Collection`（例如 `List`）
2. Constructor 的 parameter 會接收那些參數
3. 在 test method 可以拿 constructor 的 parameter 來做測試的輸入和預期輸出值

因為這次要用 Robolectric 取得 Android 的 string resource，我們要先在 *build.gradle* 加入以下的東西才能令 Robolectric 取得 Android resource：

```groovy
android {
    // 略……
    testOptions {
        unitTests {
            includeAndroidResources = true
        }
    }
}
```

但這次我們不需要刻意用 `@Config` 改變語言，因為 `mapErrorMessage` 只需要回傳 string resource，我們的 code 沒有 logic 決定輸出甚麼語言的文字。接下來就是完整的 `EtaPresenterTest`：

```kotlin
@RunWith(ParameterizedRobolectricTestRunner::class)
class EtaPresenterTest(
    private val result: EtaFailResult,
    private val expectedString: String?,
    @StringRes private val expectedResourceId: Int,
) {

    private lateinit var presenter: EtaPresenter
    private lateinit var res: Resources

    @Before
    fun setUp() {
        presenter = EtaPresenter(ApplicationProvider.getApplicationContext())
        res = ApplicationProvider.getApplicationContext<Context>().resources
    }

    @Test
    fun mapErrorMessage() {
        if (expectedString == null) {
            expectThat(presenter.mapErrorMessage(result)).isEqualTo(res.getString(expectedResourceId))
        } else {
            expectThat(presenter.mapErrorMessage(result)).isEqualTo(expectedString)
        }
    }

    companion object {
        @JvmStatic
        @get:ParameterizedRobolectricTestRunner.Parameters
        val data = listOf(
            arrayOf(EtaResult.Delay, null, R.string.delay),
            arrayOf(EtaResult.Incident("Incident", "https://example.com"), "Incident", 0),
            arrayOf(EtaResult.TooManyRequests, null, R.string.error),
            arrayOf(EtaResult.InternalServerError, null, R.string.error),
            arrayOf(EtaResult.Error(RuntimeException("Testing")), null, R.string.error),
        )
    }
}
```

先看 class 的 `@RunWith`，如果是單純的 JUnit 4 parameterized test 是應該用 JUnit 的 `Parameterized` runner。但因為我們要用 Robolectric 所以要改用 `ParameterizedRobolectricTestRunner`。之前用的那個 `AndroidJUnit4` runner 它有封裝 Robolectric runner，但不支援 parameterized test，惟有直接用 Robolectric 提供的 runner。

之後看看最底的 companion object，同樣因為要用 `ParameterizedRobolectricTestRunner`，所以要用對應的 `@get:ParameterizedRobolectricTestRunner.Parameters` 來標註提供予 test method 的參數。前面加了 `@get` 是因為我們要針對 property getter 來 annotation（因為 JUnit 是 Java 的東西，不會看懂 `val`，JUnit 本身的 `@Parameters` 是用來標註在 static method，所以要放在 companion object 內另加 `@JvmStatic`）。由於 `EtaFailResult` 有五款，我們在 `listOf` 就針對這五款情況提供了五組參數，每組都用 `arrayOf` 包住，它們分別對應 constructor 的三個參數。

之後跳到 `mapErrorMessage` 這個 test method。我們會在 test method 用到 constructor 的三個參數來完成 assertion。由於我不想日後改了 string resource 的文字後 test 會報錯，所以在使用 string resource 的情景就在參數交了 string resource ID，但缺點是 assertion 部分就像謄文般抄一次它背後的 code 一次。你可以視乎情況決定在測試時即場用 string resource ID 取得文字來做做比對還是在 test case 寫死它輸出的文字做比對。

順帶一提，如果想令 test case 的名不是 `mapErrorMessage[0]`、`mapErrorMessage[1]` 之類的話，可以在 `@get:ParameterizedRobolectricTestRunner.Parameters` 或者 `@get:Parameters` 加上 `name` 參數。例如 `@get:Parameters(name = "{0}")` 就是第一個參數的值 `toString` 後的文字，換做 `{1}` 就是第二個參數，如此類推。預設是用 `{index}` 即是參數序號。

## `EtaViewModel`

現在來到最後一個 ViewModel，最重要的部分當然是 `etaList`。由於 `EtaViewModel` 的 constructor 有 Java Time 的 `Clock`，為方便之後的測試，我們會用 [ThreeTen-Extra](https://www.threeten.org/threeten-extra/) 提供的 `MutableClock`：

```groovy
testImplementation "org.threeten:threeten-extra:$threeTenExtraVersion"
```

如果不需要在測試中途改變 `Clock` 輸出的時間，可以直接用 Java Time 的 `Clock.fixed()`，毋須另外安裝 ThreeTen-Extra。

但在寫跟 `etaList` 相關的 test case 之前我們先來測試一些簡單的東西。首先準備好 test class 的基本通用部分：

```kotlin
private val DEFAULT_LOCAL_DATE = LocalDate.of(2021, 9, 1)
private val DEFAULT_LOCAL_TIME = LocalTime.of(13, 0, 0)
private val DEFAULT_INSTANT =
    ZonedDateTime.of(DEFAULT_LOCAL_DATE, DEFAULT_LOCAL_TIME, DEFAULT_TIMEZONE).toInstant()

@RunWith(AndroidJUnit4::class)
class EtaViewModelTest {

    @get:Rule
    val coroutineScope = MainCoroutineScopeRule()

    @MockK
    private lateinit var getEtaUseCase: GetEtaUseCase

    private lateinit var clock: MutableClock

    @Before
    fun setUp() {
        MockKAnnotations.init(this)
        clock = MutableClock.of(DEFAULT_INSTANT, DEFAULT_TIMEZONE)
    }
}
```

雖然 `EtaViewModel` 沒有直接用到 Android SDK 的 class，但因為 constructor 的 `SavedStateHandle` 背後有用到 `Bundle` 所以還是要加上 `@RunWith(AndroidJUnit4::class)`。在 `setUp` 我們先設定 `MutableClock` 的時間做 2021 年 9 月 1 日下午 1 時正。這次特意用 `MutableClock` 是因為我們那個定時更新功能需要用 `Clock` 取得當前時間來決定下次 call API 的時間，如果時間是在 call `EtaViewModel` constructor 那時寫死的話就不能測試那個位置。這亦都是我們用 dependency injection library inject `Clock` 而不是用 `System.currentTimeMillis` 之類的方式取得當前時間的原因。

---

我們先來寫一些簡單的 test case：`line` 跟 `station`，這兩個 `StateFlow` 就是供 data binding 的 `MaterialToolbar` 顯示班次所屬於的路綫和車站。這兩個 `StateFlow` 的值其實是來自 `SavedStateHandle`（即是 `Fragment` 的 `arguments`），我們需要在 `EtaViewModel` 的 constructor 提供帶有這兩個參數的 `SavedStateHandle` 然後檢查這兩個 `StateFlow` 的值是不是跟我們放在 `SavedStateHandle` 的一樣。

```kotlin
@Test
fun line() = coroutineScope.runBlockingTest {
    val viewModel = EtaViewModel(
        savedStateHandle = SavedStateHandle(
            mapOf(
                "line" to Line.TCL,
                "station" to Station.TUC,
            )
        ),
        clock = clock,
        getEta = getEtaUseCase,
    )

    viewModel.line.test {
        expectThat(awaitItem()).isEqualTo(Line.TCL)
        expectNoEvents()
    }
}

@Test
fun station() = coroutineScope.runBlockingTest {
     val viewModel = EtaViewModel(
        savedStateHandle = SavedStateHandle(
            mapOf(
                "line" to Line.TCL,
                "station" to Station.TUC,
            )
        ),
        clock = clock,
        getEta = getEtaUseCase,
    )

    viewModel.station.test {
        expectThat(awaitItem()).isEqualTo(Station.TUC)
        expectNoEvents()
    }
}
```

兩個 test case 的內容基本上是一樣，首先是要建構 `EtaViewModel`。跟以前的 test case 寫法不同，我們不會在 `@Before` 預先建構 `EtaViewModel`，這是因為不同的 test case 需要傳入不同的 constructor 參數（其實是 `SavedStateHandle` 會因應 test case 不同）。之後就是用 [Turbine](https://github.com/cashapp/turbine) collect 我們要檢查的 `Flow`。由於路綫和車站在整頁的 lifecycle 都不會再改變，所以當初我們寫的時候就直接把 `MutableStateFlow` cast 成 `StateFlow`，並且在 `MutableStateFlow` 以 `SavedStateHandle` 取得的 argument 值作為它的初始值（下面就是它們的定義）。所以這兩個 `StateFlow` 只會發射一個值出去。

```kotlin
val line: StateFlow<Line> = MutableStateFlow(args.line)
val station: StateFlow<Station> = MutableStateFlow(args.station)
```

我們知道這兩個 `StateFlow` 只會發射一個值，那我們在測試時只需要 call 一次 `awaitItem()` 就可以了，當 assert 完第一個值後就可以 call `expectNoEvents()` 告訴 Turbine 之後應該不會再有新的值出現。

之後我們看看另一個 test case `navigateBack`。

```kotlin
@Test
fun navigateBack() = coroutineScope.runBlockingTest {
    val viewModel = EtaViewModel(
        savedStateHandle = SavedStateHandle(
            mapOf(
                "line" to Line.TCL,
                "station" to Station.TUC,
            )
        ),
        clock = clock,
        getEta = getEtaUseCase,
    )

    viewModel.navigateBack.test {
        viewModel.goBack()
        awaitEvent()
        expectNoEvents()
    }
}
```

這個又是比較簡單的，就是測試 call 了 `viewModel.goBack()` 後 `viewModel.navigateBack` 這個 `Flow` 有沒有發射訊號提示 `EtaFragment` 轉頁。由於這個 `Flow` 的 type 是 `Unit`，我們就不用 assert 它的值，只需要讓它消耗掉就可以了。

接下來我們會寫 `viewIncidentDetail` 的測試。由於它是看結果是不是 `EtaResult.Incident` 才向 `viewIncidentDetail` 發射要瀏覽的網址，所以我們先試試當 `getEtaUseCase` 輸出 `EtaResult.Incident` 的情況：

```kotlin
@Test
fun `viewIncidentDetail incident`() = coroutineScope.runBlockingTest {
    coEvery {
        getEtaUseCase(
            Language.ENGLISH,
            Line.TCL,
            Station.TUC,
            GetEtaUseCase.SortBy.DIRECTION,
        )
    } returns EtaResult.Incident("Message", "https://example.com")

    val viewModel = EtaViewModel(
        savedStateHandle = SavedStateHandle(
            mapOf(
                "line" to Line.TCL,
                "station" to Station.TUC,
            )
        ),
        clock = clock,
        getEta = getEtaUseCase,
    )

    viewModel.etaList.test {
        viewModel.viewIncidentDetail.test {
            viewModel.startAutoRefresh()
            viewModel.viewIncidentDetail()
            expectThat(awaitItem()).isEqualTo("https://example.com")
            expectNoEvents()
        }
        cancelAndIgnoreRemainingEvents()
    }
}
```

因為在 call 了 `EtaViewModel.startAutoRefresh` 才會 call use case，我們在 `viewModel.viewIncidentDetail.test` 內第一句就是 `viewModel.startAutoRefresh()`。在寫的時候我發現 `viewModel.viewIncidentDetail.test` 會有怪問題出現。原來是因為我們之前寫 `etaResult` 是 `StateFlow<CachedResult>`。如果外圍再包多一層 `viewModel.etaList.test` 的話才會正常，這是因為 `StateFlow` 是 cold flow。意思是如果沒有其他人 collect 這個 flow 的話，那在 `etaResult` 寫的一大串 `flatMapLatest` 和 `scan` 是不會執行。但 `etaResult` 是 private，所以我找了 `etaList` 來 collect（因為它的上游是 `etaResult`），這樣才不會卡死在 `Loading`。在實際執行其實看不到這個問題，因為 layout XML 的 data binding 會 collect 那一大堆跟 `etaResult` 相關的 `StateFlow`，所以 `etaResult` 內的東西一定會被執行。但感覺上還是不好吧，所以我們應該把 `etaResult` 改成即使沒有人 collect 仍會執行（即是 hot flow）。`SharedFlow` 就是 hot flow 的一種，以下是節錄自 `SharedFlow` 的 KDoc：

> A *hot* Flow that shares emitted values among all its collectors in a broadcast fashion, so that all collectors get all emitted values. A shared flow is called hot because its active instance exists independently of the presence of collectors. This is opposed to a regular `Flow`, such as defined by the `flow { ... }` function, which is cold and is started separately for each collector.

有一樣東西要留意是：如果連續有兩個相同的值發送到 `SharedFlow` 的話，那麼兩個值都能交到下游；但 `StateFlow` 就會吃掉第二個值，直至下一個值跟先前的值不相同才會交到下游。不過在我們這個情況因為下游都是 `StateFlow`，即使上游發射重覆的值對那些 `StateFlow` 的下游都沒有分別。

要改成 `SharedFlow`，只需把原先的 `stateIn` 換成 `shareIn`。 `replay` 設定 `1` 是為了其他人一開始訂閱時就能馬上收到 `SharedFlow` 在訂閱前所發射的最後一個值，這樣就不用讓下游在訂閱時乾等到下一次更新才能收到 `CachedResult`。 

改了 `etaResult` 後還是要改其他地方，因為 `SharedFlow` 是沒有 `value` 這個 property，要取新最新的值就要用 `first()`。所以我們需要一併修改 `startAutoRefresh` 和 `viewIncidentDetail`。我們亦順帶修正 `viewIncidentDetail` 只看 `currentResult` 的問題：如果當前是載入中但畫面仍是顯示事故畫面的話那按下「View detail」沒有反應。

```kotlin
private val etaResult: SharedFlow<CachedResult> = triggerRefresh
    .consumeAsFlow()
    .flatMapLatest { /* 略 */ }
    .scan(CachedResult()) { acc, currentValue -> /* 略 */ }
    .shareIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(STATE_FLOW_STOP_TIMEOUT_MILLIS),
        replay = 1,
    )

fun startAutoRefresh() {
    autoRefreshScope.launch {
        val delayDuration =
            JavaDuration.between(etaResult.first().currentResult.updatedAt, clock.instant())
        // 略
    }
}

fun viewIncidentDetail() {
    viewModelScope.launch {
        val result = etaResult.first().lastFailResult
        if (result !is EtaResult.Incident) return@launch
        _viewIncidentDetail.send(result.url)
    }
}
```

那剛才的 `viewIncidentDetail incident` test case 我們就可以拿掉 `viewModel.etaList.test` 的部分。

我們再試試當 `lastFailResult` 不是 `EtaResult.Incident` 的情況，為了令 test case 寫得短，我用了 `EtaResult.Delay` 來試。

```kotlin
@Test
fun `viewIncidentDetail delay`() = coroutineScope.runBlockingTest {
    coEvery {
        getEtaUseCase(
            Language.ENGLISH,
            Line.TCL,
            Station.TUC,
            GetEtaUseCase.SortBy.DIRECTION,
        )
    } returns EtaResult.Delay

    val viewModel = EtaViewModel(
        savedStateHandle = SavedStateHandle(
            mapOf(
                "line" to Line.TCL,
                "station" to Station.TUC,
            )
        ),
        clock = clock,
        getEta = getEtaUseCase,
    )

    viewModel.viewIncidentDetail.test {
        viewModel.startAutoRefresh()
        viewModel.viewIncidentDetail()
        expectNoEvents()
    }
}
```

這次我們期望 `viewIncidentDetail` 這個 `Flow` 不會發射訊號，所以用了 `expectNoEvents()`。

在本篇結束前我們再寫多一個 test case `showLoading`，它就是用來控制是否顯示 `CircularProgressIndicator`。`CircularProgressIndicator` 會在載入完成後消失，我們會檢查它是不是首先顯示然後轉為不顯示。

```kotlin
@Test
fun showLoading() = coroutineScope.runBlockingTest {
    coEvery {
        getEtaUseCase(
            Language.ENGLISH,
            Line.TCL,
            Station.TUC,
            GetEtaUseCase.SortBy.DIRECTION,
        )
    } returns EtaResult.InternalServerError

    val viewModel = EtaViewModel(
        savedStateHandle = SavedStateHandle(
            mapOf(
                "line" to Line.TCL,
                "station" to Station.TUC,
            )
        ),
        clock = clock,
        getEta = getEtaUseCase,
    )

    viewModel.showLoading.test {
        viewModel.startAutoRefresh()
        expectThat(awaitItem()).isEqualTo(true)
        expectThat(awaitItem()).isEqualTo(false)
        expectNoEvents()
    }
}
```

## 小結

現在我們已經寫了幾個 test case，了解到如何測試 `Flow`，順帶介紹了 `SharedFlow`。另外亦在寫測試時發現先前寫的 code 有 bug，這其實是正常的，因為靠實機人手體驗可能會看不到一些問題，換了另一個角度又會看得到之前不為意的問題。下一篇我們會寫一些跟時間相關的 test case，完整的 code 可以在 [GitHub repo](https://github.com/ericksli/eta-demo/tree/main/app/src/test/java/net/swiftzer/etademo/presentation/eta) 找到。

## 參考

- [Testing Coroutines on Android (Android Dev Summit '19)](https://www.youtube.com/watch?v=KMb0Fs8rCRs)
- [KotlinConf 2019: Testing with Coroutines by Sean McQuillan](https://www.youtube.com/watch?v=hMFwNLVK8HU)
- [StateFlow and SharedFlow](https://developer.android.com/kotlin/flow/stateflow-and-sharedflow)

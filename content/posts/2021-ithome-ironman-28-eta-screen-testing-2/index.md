---
title: "2021 iThome 鐵人賽 Day 28：ETA screen testing (2)"
tags:
  - 2021 iThome 鐵人賽
  - Android
date: 2021-10-13T00:00:00+08:00
canonicalURL: https://ithelp.ithome.com.tw/articles/10281130
---

> 本篇文章是 [2021 iThome 鐵人賽](https://ithelp.ithome.com.tw/2021ironman)參賽題目「[寫一個列車抵站時間 Android App](https://ithelp.ithome.com.tw/users/20139666/ironman/4661)」的第 28 篇，你可到 iThome [查看原文](https://ithelp.ithome.com.tw/articles/10281130)。
>
> [文章目錄]({{< ref "2021-ithome-ironman" >}})

上一篇我們寫了一些 `EtaViewModel` 的測試，這一篇會集中寫跟時間相關的測試。

之前在 `EtaViewModel` 我們定義了更新一次的間距常數 `AUTO_REFRESH_INTERVAL`，現在我們要在 `EtaViewModelTest` 用到它，所以要把它改成 public：

```kotlin
import kotlin.time.Duration as KotlinDuration

val AUTO_REFRESH_INTERVAL = KotlinDuration.seconds(10)
```

在看 test case 的 code 之前我們先了解 Kotlin Coroutine 如何處理時間相關的測試。如果有接觸過 RxJava 的話，要測試跟時間相關的 operator 就要用到 `TestScheduler` 的 `advanceTimeBy` method 把時間快進。Kotlin Coroutine 的作法都是差不多，寫法是在 `runBlockingTest` 內 call `advanceTimeBy`。之前我們在寫 Ktor client 的測試時因為找不到 Ktor client 如何自訂 `Executor` 或者  `Dispatcher`，所以惟有用了 `runBlocking` 而不是 `runBlockingTest`。`runBlocking` 跟 `runBlockingTest` 的分別是 `runBlocking` 內如果加了 `delay(Duration.seconds(10))` 的話那個 test case 就真的會在等十秒才執行 `delay` 的下一句；但 `runBlockingTest` 就會自動把這些 `delay` 快進，直到它發現已經進入閒置狀態。這樣就可以令 test case 執行速度加快，不用再乾等十秒。

回到我們的 `EtaViewModel`，我們在每次收到 `getEtaUseCase` 的回傳值就會在 `onEach` 內執行一句 `delay`，`delay` 之後就向 `triggerRefresh` `Channel` 發訊號觸發整串 `etaResult` 執行一遍，那之後又會再執行多次 `onEach` 的東西。整串 `etaResult` 就是一個無限循環，不會有閒置狀態。所以我們不能簡單地靠 `runBlockingTest` 自動快進功能來寫跟自動更新相關的 test case。那我們要做的就是手動把時間快進，然後在快進後檢查 `Flow` 的值。另一樣東西要留意的是我們在 `startAutoRefresh` 會比對上次 `getEtaUseCase` 回傳的時間和現在時間來決定要 `delay` 多久才 call 另一次 `getEtaUseCase`。這個部分牽涉到 `EtaViewModel` constructor 的 `Clock`。所以除了用 Coroutine test 的 `advanceTimeBy` 外，我們亦需要把 `Clock` 的時間同時快進，這樣才能正確地模擬現實情景。這亦都是我在上一篇特意引入 [ThreeTen Extra](https://www.threeten.org/threeten-extra/) 的原因。

為了更簡單地快進兩邊的時間，我們先準備一個 extension function：

```kotlin
import kotlin.time.Duration as KotlinDuration

private fun DelayController.advanceTimeBy(amount: KotlinDuration) {
    clock.add(amount.toJavaDuration())
    advanceTimeBy(amount.inWholeMilliseconds)
}
```

接下來我們先來看看 `showFullScreenError` 的 test case，`showFullScreenError` 就是控制是否顯示全頁式的錯誤畫面：

```kotlin
@Test
fun showFullScreenError() = coroutineScope.runBlockingTest {
    coEvery {
        getEtaUseCase(
            Language.ENGLISH,
            Line.TCL,
            Station.TUC,
            GetEtaUseCase.SortBy.DIRECTION,
        )
    }.returnsMany(
        EtaResult.InternalServerError,
        EtaResult.Success(),
        EtaResult.TooManyRequests,
        EtaResult.Delay,
    )

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

    viewModel.showFullScreenError.test {
        viewModel.startAutoRefresh()
        expectThat(awaitItem()).isEqualTo(false) // Loading/ScreenState.LOADING
        advanceTimeBy(AUTO_REFRESH_INTERVAL)
        expectThat(awaitItem()).isEqualTo(true) // InternalServerError/ScreenState.FULL_SCREEN_ERROR
        advanceTimeBy(AUTO_REFRESH_INTERVAL)
        expectThat(awaitItem()).isEqualTo(false) // Success/ScreenState.ETA, TooManyRequests/ScreenState.ETA_WITH_ERROR_BANNER
        advanceTimeBy(AUTO_REFRESH_INTERVAL)
        expectThat(awaitItem()).isEqualTo(true) // Delay/ScreenState.FULL_SCREEN_ERROR
        expectNoEvents()
    }
}
```

看到我們用了 `returnsMany` 來一次過定義好幾個 `getEtaUseCase` 會回傳的值，它會順序地回傳。例如第一次 call 會回傳 `EtaResult.InternalServerError`、第二次 call 會回傳 `EtaResult.Success()`……到最底的部分，我們檢查當 `getEtaUseCase` 回傳了不同的結果時 `showFullScreenError` 會發射的 `Boolean` 值。留意是由 `EtaResult.Success()` 轉到 `EtaResult.TooManyRequests` 時介面並不需要顯示全頁錯誤畫面，只需要顯示錯誤 banner。因為 `showFullScreenError` 是 `StateFlow`，所以並沒有連續發射出兩個 `false` 出來而是一個 `false`。

我們再看另一個類似的 test case `showEtaList`：

```kotlin
@Test
fun showEtaList() = coroutineScope.runBlockingTest {
    coEvery {
        getEtaUseCase(
            Language.ENGLISH,
            Line.TCL,
            Station.TUC,
            GetEtaUseCase.SortBy.DIRECTION,
        )
    }.returnsMany(
        EtaResult.InternalServerError,
        EtaResult.Success(),
        EtaResult.TooManyRequests,
        EtaResult.Delay,
    )

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

    viewModel.showEtaList.test {
        viewModel.startAutoRefresh()
        expectThat(awaitItem()).isEqualTo(false) // ScreenState.LOADING
        advanceTimeBy(AUTO_REFRESH_INTERVAL)
        expectThat(awaitItem()).isEqualTo(true) // ScreenState.ETA, ScreenState.ETA_WITH_ERROR_BANNER
        advanceTimeBy(AUTO_REFRESH_INTERVAL)
        expectNoEvents()
        advanceTimeBy(AUTO_REFRESH_INTERVAL)
        expectThat(awaitItem()).isEqualTo(false) // ScreenState.FULL_SCREEN_ERROR
        expectNoEvents()
    }
}
```

現在掌握到如何操縱時間後，我們就可以寫針對 `etaList` 的 test case。但首先要準備一下 custom assertion：

```kotlin
private fun Assertion.Builder<EtaListItem>.assertHeader(
    direction: EtaResult.Success.Eta.Direction,
) = isA<EtaListItem.Header>().and {
    get(EtaListItem.Header::direction).isEqualTo(direction)
}

private fun Assertion.Builder<EtaListItem>.assertEta(
    direction: EtaResult.Success.Eta.Direction,
    destination: Station,
    platform: String,
    minuteCountdown: Int,
) = isA<EtaListItem.Eta>().and {
    get(EtaListItem.Eta::direction).isEqualTo(direction)
    get(EtaListItem.Eta::destination).isEqualTo(destination)
    get(EtaListItem.Eta::platform).isEqualTo(platform)
    get(EtaListItem.Eta::minuteCountdown).isEqualTo(minuteCountdown)
}
```

現在先檢查排序切換，看看 `etaList` 寫的 header 加插是否正確。

```kotlin
@Test
fun `etaList sorting`() = coroutineScope.runBlockingTest {
    coEvery {
        getEtaUseCase(
            Language.ENGLISH,
            Line.TML,
            Station.KSR,
            any(),
        )
    } returns EtaResult.Success(
        schedule = listOf(
            EtaResult.Success.Eta(
                direction = EtaResult.Success.Eta.Direction.UP,
                destination = Station.TUM,
                platform = "1",
                time = ZonedDateTime.of(
                    DEFAULT_LOCAL_DATE,
                    LocalTime.of(13, 1, 1),
                    DEFAULT_TIMEZONE
                ).toInstant()
            ),
            EtaResult.Success.Eta(
                direction = EtaResult.Success.Eta.Direction.UP,
                destination = Station.SIH,
                platform = "1",
                time = ZonedDateTime.of(
                    DEFAULT_LOCAL_DATE,
                    LocalTime.of(13, 7, 59),
                    DEFAULT_TIMEZONE
                ).toInstant()
            ),
            EtaResult.Success.Eta(
                direction = EtaResult.Success.Eta.Direction.DOWN,
                destination = Station.HUH,
                platform = "2",
                time = ZonedDateTime.of(
                    DEFAULT_LOCAL_DATE,
                    LocalTime.of(13, 2, 2),
                    DEFAULT_TIMEZONE
                ).toInstant()
            ),
        ),
    )

    val viewModel = EtaViewModel(
        savedStateHandle = SavedStateHandle(
            mapOf(
                "line" to Line.TML,
                "station" to Station.KSR,
            )
        ),
        clock = clock,
        getEta = getEtaUseCase,
    )

    viewModel.etaList.test {
        expectThat(awaitItem()).isEmpty()
        viewModel.startAutoRefresh()
        expectThat(awaitItem()).hasSize(5).and {
            get(0).assertHeader(EtaResult.Success.Eta.Direction.UP)
            get(1).assertEta(
                direction = EtaResult.Success.Eta.Direction.UP,
                destination = Station.TUM,
                platform = "1",
                minuteCountdown = 1,
            )
            get(2).assertEta(
                direction = EtaResult.Success.Eta.Direction.UP,
                destination = Station.SIH,
                platform = "1",
                minuteCountdown = 7,
            )
            get(3).assertHeader(EtaResult.Success.Eta.Direction.DOWN)
            get(4).assertEta(
                direction = EtaResult.Success.Eta.Direction.DOWN,
                destination = Station.HUH,
                platform = "2",
                minuteCountdown = 2,
            )
        }
        viewModel.toggleSorting()
        expectThat(awaitItem()).hasSize(3).and {
            get(0).assertEta(
                direction = EtaResult.Success.Eta.Direction.UP,
                destination = Station.TUM,
                platform = "1",
                minuteCountdown = 1,
            )
            get(1).assertEta(
                direction = EtaResult.Success.Eta.Direction.UP,
                destination = Station.SIH,
                platform = "1",
                minuteCountdown = 7,
            )
            get(2).assertEta(
                direction = EtaResult.Success.Eta.Direction.DOWN,
                destination = Station.HUH,
                platform = "2",
                minuteCountdown = 2,
            )
        }
        viewModel.toggleSorting()
        expectThat(awaitItem()).hasSize(5).and {
            get(0).assertHeader(EtaResult.Success.Eta.Direction.UP)
            get(1).assertEta(
                direction = EtaResult.Success.Eta.Direction.UP,
                destination = Station.TUM,
                platform = "1",
                minuteCountdown = 1,
            )
            get(2).assertEta(
                direction = EtaResult.Success.Eta.Direction.UP,
                destination = Station.SIH,
                platform = "1",
                minuteCountdown = 7,
            )
            get(3).assertHeader(EtaResult.Success.Eta.Direction.DOWN)
            get(4).assertEta(
                direction = EtaResult.Success.Eta.Direction.DOWN,
                destination = Station.HUH,
                platform = "2",
                minuteCountdown = 2,
            )
        }
        expectNoEvents()
    }
    coVerify(exactly = 2) { getEtaUseCase(any(), any(), any(), GetEtaUseCase.SortBy.DIRECTION) }
    coVerify(exactly = 1) { getEtaUseCase(any(), any(), any(), GetEtaUseCase.SortBy.TIME) }
}
```

看起來很長，但其實很簡單。我們這次沒有快進時間，主要是看它發射出來的班次是否正確。首先在 `startAutoRefresh` 之前我們先檢查 `StateFlow` 的初始值 empty list。然後當第一次載入時預設是按方向排序，所以會有 header。之後我們改變排序，於事就變了按時間排序。但因為實際的排序是在 `GetEtaUseCaseImpl` 做，我們又沒特別 mock 第二次 call `getEtaUseCase` 的 return value，所以實際結果的排序看起來不合理，但 header 就正如我們的期望被拿走。最後試試切換排序一次，看看是不是跟第一次的結果一樣。最尾的 `coVerify` 是用來檢查 `getEtaUseCase` 是不是被執行了兩次按方向排序和一次按時間排序，那些 `any()` 就是說我們不在乎那些參數的值是甚麼。當然你可以寫明參數的值來確保我們寫的 code 的確合符預期。

其實如果不用 [Turbine](https://github.com/cashapp/turbine) 的話，`viewModel.etaList.test` 的部分可以寫成這樣：

```kotlin
val results = mutableListOf<List<EtaListItem>>()
val job = launch {
    viewModel.etaList.toList(results)
}
viewModel.startAutoRefresh()
viewModel.toggleSorting()
viewModel.toggleSorting()
job.cancel()
expectThat(results).hasSize(4).and { /* 針對每個元素做檢查 */ }
```

有時候在寫 test case 時發覺結果不似預期，或許可以用這個寫法看看它的結果是甚麼然後才想想那裏出現問題。

接下來是另一個 test，這是為了測試當首次載入後能否在十秒後自動 call `getEtaUseCase` 一次取得最新班次。

```kotlin
@Test
fun `etaList auto refresh automatically after loaded`() = coroutineScope.runBlockingTest {
    coEvery {
        getEtaUseCase(
            Language.ENGLISH,
            Line.TML,
            Station.KSR,
            GetEtaUseCase.SortBy.DIRECTION,
        )
    }.returnsMany(
        EtaResult.Success(
            schedule = listOf(
                EtaResult.Success.Eta(
                    direction = EtaResult.Success.Eta.Direction.UP,
                    destination = Station.TUM,
                    platform = "1",
                    time = ZonedDateTime.of(
                        DEFAULT_LOCAL_DATE,
                        LocalTime.of(13, 1, 1),
                        DEFAULT_TIMEZONE
                    ).toInstant()
                ),
            ),
        ),
        EtaResult.Success(
            schedule = listOf(
                EtaResult.Success.Eta(
                    direction = EtaResult.Success.Eta.Direction.DOWN,
                    destination = Station.TIS,
                    platform = "8",
                    time = ZonedDateTime.of(
                        DEFAULT_LOCAL_DATE,
                        LocalTime.of(13, 14, 0),
                        DEFAULT_TIMEZONE
                    ).toInstant()
                ),
            ),
        )
    )

    val viewModel = EtaViewModel(
        savedStateHandle = SavedStateHandle(
            mapOf(
                "line" to Line.TML,
                "station" to Station.KSR,
            )
        ),
        clock = clock,
        getEta = getEtaUseCase,
    )

    viewModel.etaList.test {
        viewModel.startAutoRefresh()
        // StateFlow 初始值
        expectThat(awaitItem()).isEmpty()
        // 第一次 getEtaUseCase
        expectThat(awaitItem()).hasSize(2).and {
            get(0).assertHeader(EtaResult.Success.Eta.Direction.UP)
            get(1).assertEta(
                direction = EtaResult.Success.Eta.Direction.UP,
                destination = Station.TUM,
                platform = "1",
                minuteCountdown = 1,
            )
        }
        // 快進到下一次更新時間
        advanceTimeBy(AUTO_REFRESH_INTERVAL)
        // 第二次 getEtaUseCase 執行中，目前仍然是用第一次 getEtaUseCase 的結果，
        // 但因為重新執行 etaList 內的 combine 所以會重新計算倒數分鐘
        expectThat(awaitItem()).hasSize(2).and {
            get(0).assertHeader(EtaResult.Success.Eta.Direction.UP)
            get(1).assertEta(
                direction = EtaResult.Success.Eta.Direction.UP,
                destination = Station.TUM,
                platform = "1",
                minuteCountdown = 0,
            )
        }
        // 第二次 getEtaUseCase
        expectThat(awaitItem()).hasSize(2).and {
            get(0).assertHeader(EtaResult.Success.Eta.Direction.DOWN)
            get(1).assertEta(
                direction = EtaResult.Success.Eta.Direction.DOWN,
                destination = Station.TIS,
                platform = "8",
                minuteCountdown = 13,
            )
        }
        expectNoEvents()
    }
    coVerify(exactly = 2) {
        getEtaUseCase(
            Language.ENGLISH,
            Line.TML,
            Station.KSR,
            GetEtaUseCase.SortBy.DIRECTION,
        )
    }
}
```

基本上寫法都是大同小異，只是要留意我們之前寫的 logic 是有載入中這個過程，當 `etaResult` 發射載入中的時候 `etaList` 還是會沿用上一次的結果來輸出，然後當新的 `getEtaUseCase` 結果來到後就用新的結果轉化出供 `RecyclerView` 顯示的內容。

最後來多一個 test case 測試當 `Fragment` `onPause` 再 `onResume` 的情景。這個重新返回班次頁的情景可以分為兩個：在十秒內返回和過十秒後返回。

```kotlin
@Test
fun `etaList stop and resume auto refresh`() =
    coroutineScope.runBlockingTest {
        coEvery {
            getEtaUseCase(
                Language.ENGLISH,
                Line.TKL,
                Station.QUB,
                GetEtaUseCase.SortBy.DIRECTION,
            )
        }.returnsMany(
            EtaResult.Success(
                schedule = listOf(
                    EtaResult.Success.Eta(
                        direction = EtaResult.Success.Eta.Direction.UP,
                        destination = Station.LHP,
                        platform = "1",
                        time = ZonedDateTime.of(
                            DEFAULT_LOCAL_DATE,
                            LocalTime.of(13, 20, 30),
                            DEFAULT_TIMEZONE
                        ).toInstant(),
                    ),
                ),
            ),
            EtaResult.Success(
                schedule = listOf(
                    EtaResult.Success.Eta(
                        direction = EtaResult.Success.Eta.Direction.UP,
                        destination = Station.LHP,
                        platform = "2",
                        time = ZonedDateTime.of(
                            DEFAULT_LOCAL_DATE,
                            LocalTime.of(13, 30, 0),
                            DEFAULT_TIMEZONE
                        ).toInstant(),
                    ),
                ),
            ),
            EtaResult.Success(
                schedule = listOf(
                    EtaResult.Success.Eta(
                        direction = EtaResult.Success.Eta.Direction.UP,
                        destination = Station.LHP,
                        platform = "3",
                        time = ZonedDateTime.of(
                            DEFAULT_LOCAL_DATE,
                            LocalTime.of(13, 30, 0),
                            DEFAULT_TIMEZONE
                        ).toInstant(),
                    ),
                ),
            ),
        )

        val viewModel = EtaViewModel(
            savedStateHandle = SavedStateHandle(
                mapOf(
                    "line" to Line.TKL,
                    "station" to Station.QUB,
                )
            ),
            clock = clock,
            getEta = getEtaUseCase,
        )

        viewModel.etaList.test {
            viewModel.startAutoRefresh()
            // StateFlow 初始值
            expectThat(awaitItem()).isEmpty()
            // 第一次 getEtaUseCase
            expectThat(awaitItem()).hasSize(2).and {
                get(0).assertHeader(EtaResult.Success.Eta.Direction.UP)
                get(1).assertEta(
                    direction = EtaResult.Success.Eta.Direction.UP,
                    destination = Station.LHP,
                    platform = "1",
                    minuteCountdown = 20,
                )
            }
            viewModel.stopAutoRefresh()
            advanceTimeBy(KotlinDuration.seconds(5))
            expectNoEvents()
            viewModel.startAutoRefresh()
            // 未夠十秒，上游不會有新的值
            expectNoEvents()
            advanceTimeBy(KotlinDuration.seconds(5))
            // 第二次 getEtaUseCase
            expectThat(awaitItem()).hasSize(2).and {
                get(0).assertHeader(EtaResult.Success.Eta.Direction.UP)
                get(1).assertEta(
                    direction = EtaResult.Success.Eta.Direction.UP,
                    destination = Station.LHP,
                    platform = "2",
                    minuteCountdown = 29,
                )
            }
            expectNoEvents()
            viewModel.stopAutoRefresh()
            advanceTimeBy(KotlinDuration.minutes(20))
            viewModel.startAutoRefresh()
            // 第三次 getEtaUseCase 執行中，仍然是用第二次 getEtaUseCase
            expectThat(awaitItem()).hasSize(2).and {
                get(0).assertHeader(EtaResult.Success.Eta.Direction.UP)
                get(1).assertEta(
                    direction = EtaResult.Success.Eta.Direction.UP,
                    destination = Station.LHP,
                    platform = "2",
                    minuteCountdown = 9,
                )
            }
						// 第三次 getEtaUseCase
            expectThat(awaitItem()).hasSize(2).and {
                get(0).assertHeader(EtaResult.Success.Eta.Direction.UP)
                get(1).assertEta(
                    direction = EtaResult.Success.Eta.Direction.UP,
                    destination = Station.LHP,
                    platform = "3",
                    minuteCountdown = 9,
                )
            }
            expectNoEvents()
        }
        coVerify(exactly = 3) {
            getEtaUseCase(
                Language.ENGLISH,
                Line.TKL,
                Station.QUB,
                GetEtaUseCase.SortBy.DIRECTION,
            )
        }
    }
```

## 小結

這次的 code 比較長，這是因為那些 use case 的 return value 本身都很長，加上每個值都要做 assertion，但 test case 的寫法來來去去都是差不多。本篇主要介紹了 Kotlin Coroutine 測試時如何快進時間，另外亦實際示範了為甚麼我們要用 `Clock` 來獲取當前時間。其餘的 test case 因為性質相近所以我就不再寫了，因為現在都可以示範到那些手法。而我們的班次示範 app 來到現在都大致上完結，下篇會再抽一些題目再討論一下。這次的 code 可以在 [GitHub repo](https://github.com/ericksli/eta-demo/blob/main/app/src/test/java/net/swiftzer/etademo/presentation/eta/EtaViewModelTest.kt) 找到。

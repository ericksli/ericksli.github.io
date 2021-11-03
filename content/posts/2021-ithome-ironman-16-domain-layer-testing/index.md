---
title: "2021 iThome 鐵人賽 Day 16：Domain layer testing"
tags:
  - 2021 iThome 鐵人賽
  - Android
date: 2021-10-01T00:00:00+08:00
canonicalURL: https://ithelp.ithome.com.tw/articles/10275721
---

> 本篇文章是 [2021 iThome 鐵人賽](https://ithelp.ithome.com.tw/2021ironman)參賽題目「[寫一個列車抵站時間 Android App](https://ithelp.ithome.com.tw/users/20139666/ironman/4661)」的第 16 篇，你可到 iThome [查看原文](https://ithelp.ithome.com.tw/articles/10275721)。
>
> [文章目錄]({{< ref "2021-ithome-ironman" >}})

今天會為上一篇所寫的兩個 use case 加上 unit test。

## `GetLinesAndStationsUseCaseImplTest`

這個 test 其實很簡單，因為本身就是直接把 `EtaRepository` 拿到的 `Map` return 出去，所以 unit test 我們只需 mock `EtaRepository` 的 `getLinesAndStations` 然後檢查 use case return 出來的 map 是不是跟我們 mock 出來的 `getLinesAndStations` return value 是否一致即可。

```kotlin
private val DUMMY_DATA = mapOf(
    Line.AEL to setOf(Station.HOK, Station.KOW),
    Line.TKL to setOf(Station.NOP, Station.YAT, Station.LHP),
)

class GetLinesAndStationsUseCaseImplTest {

    private lateinit var useCase: GetLinesAndStationsUseCase

    @MockK
    private lateinit var repository: EtaRepository

    @Before
    fun setUp() {
        MockKAnnotations.init(this)
        useCase = GetLinesAndStationsUseCaseImpl(repository)
        every { repository.getLinesAndStations() } returns DUMMY_DATA

    }

    @Test
    fun invoke() {
        expectThat(useCase()).isEqualTo(DUMMY_DATA)
    }
}
```

## `GetEtaUseCaseImplTest`

另一個 unit test 是關於車站班次，基本上都是左手交右手（把 `EtaRepository` return 出來的東西直接 return 給 ViewModel 用），唯獨是在 `EtaResult.Success` 時我們會再排序。所以我們這次測試的重點會落在 `EtaResult.Success` 的情景。首先是測試按行車方向排序：

```kotlin
class GetEtaUseCaseImplTest {

    private lateinit var useCase: GetEtaUseCase

    @MockK
    private lateinit var repository: EtaRepository

    @Before
    fun setUp() {
        MockKAnnotations.init(this)
        useCase = GetEtaUseCaseImpl(repository)
    }

    @Test
    fun `success, sort by direction`() = runBlockingTest {
        coEvery {
            repository.getEta(
                Language.ENGLISH,
                Line.TCL,
                Station.OLY
            )
        } returns EtaResult.Success(
            schedule = listOf(
                EtaResult.Success.Eta(
                    direction = EtaResult.Success.Eta.Direction.DOWN,
                    platform = "1",
                    time = Instant.ofEpochSecond(1630700001),
                    destination = Station.HOK,
                    sequence = 2,
                ),
                EtaResult.Success.Eta(
                    direction = EtaResult.Success.Eta.Direction.UP,
                    platform = "1",
                    time = Instant.ofEpochSecond(1630700002),
                    destination = Station.TSY,
                    sequence = 2,
                ),
                EtaResult.Success.Eta(
                    direction = EtaResult.Success.Eta.Direction.UP,
                    platform = "1",
                    time = Instant.ofEpochSecond(1630700002),
                    destination = Station.TUC,
                    sequence = 1,
                ),
                EtaResult.Success.Eta(
                    direction = EtaResult.Success.Eta.Direction.DOWN,
                    platform = "1",
                    time = Instant.ofEpochSecond(1630700004),
                    destination = Station.KOW,
                    sequence = 1,
                ),
            ),
        )
        val result =
            useCase.invoke(Language.ENGLISH, Line.TCL, Station.OLY, GetEtaUseCase.SortBy.DIRECTION)
        expectThat(result).isA<EtaResult.Success>().get(EtaResult.Success::schedule).hasSize(4)
            .and {
                get(0).get(EtaResult.Success.Eta::destination).isEqualTo(Station.TUC)
                get(1).get(EtaResult.Success.Eta::destination).isEqualTo(Station.TSY)
                get(2).get(EtaResult.Success.Eta::destination).isEqualTo(Station.KOW)
                get(3).get(EtaResult.Success.Eta::destination).isEqualTo(Station.HOK)
            }
    }
}
```

這次我們偷懶，我們把四筆班次的目的地 `destination` 都設定成不同車站，然後只檢查車站就知道順序是否正確。第二和第三筆班次是用來測試當時間相同時會不會按序號 `sequence` 排列。

另一個 test case 是測試按時間排序。跟剛才的 `success, sort by direction` test case 一樣，我們都是借 `destination` 判斷排序是否正確。留意第一及第二筆的時間 `time` 都是相同，目的是檢查結果是否有按序號 `sequence` 排列。

```kotlin
@Test
fun `success, sort by time`() = runBlockingTest {
    coEvery {
        repository.getEta(
            Language.ENGLISH,
            Line.TCL,
            Station.OLY
        )
    } returns EtaResult.Success(
        schedule = listOf(
            EtaResult.Success.Eta(
                direction = EtaResult.Success.Eta.Direction.DOWN,
                platform = "1",
                time = Instant.ofEpochSecond(1630700001),
                destination = Station.HOK,
                sequence = 2,
            ),
            EtaResult.Success.Eta(
                direction = EtaResult.Success.Eta.Direction.UP,
                platform = "1",
                time = Instant.ofEpochSecond(1630700001),
                destination = Station.TUC,
                sequence = 1,
            ),
            EtaResult.Success.Eta(
                direction = EtaResult.Success.Eta.Direction.UP,
                platform = "1",
                time = Instant.ofEpochSecond(1630700004),
                destination = Station.TSY,
                sequence = 2,
            ),
            EtaResult.Success.Eta(
                direction = EtaResult.Success.Eta.Direction.DOWN,
                platform = "1",
                time = Instant.ofEpochSecond(1630700003),
                destination = Station.KOW,
                sequence = 1,
            ),
        ),
    )
    val result =
        useCase.invoke(Language.ENGLISH, Line.TCL, Station.OLY, GetEtaUseCase.SortBy.TIME)
    expectThat(result).isA<EtaResult.Success>().get(EtaResult.Success::schedule).hasSize(4)
        .and {
            get(0).get(EtaResult.Success.Eta::destination).isEqualTo(Station.TUC)
            get(1).get(EtaResult.Success.Eta::destination).isEqualTo(Station.HOK)
            get(2).get(EtaResult.Success.Eta::destination).isEqualTo(Station.KOW)
            get(3).get(EtaResult.Success.Eta::destination).isEqualTo(Station.TSY)
        }
}
```

除了 `EtaResult.Success` 外，我們還會為其他情況寫 test case，下面是 `EtaResult.Incident` 的 test case：

```kotlin
@Test
fun incident() = runBlockingTest {
    val incident = EtaResult.Incident("Out of order!", "http://example.com")
    coEvery {
        repository.getEta(
            Language.ENGLISH,
            Line.TCL,
            Station.OLY
        )
    } returns incident
    val result =
        useCase.invoke(Language.ENGLISH, Line.TCL, Station.OLY, GetEtaUseCase.SortBy.DIRECTION)
    expectThat(result).isEqualTo(incident)
}
```

基本上都是比對 return value 是不是跟 `EtaRepository` return 出來的一樣，其餘的 test case 都是同樣寫法，所以我就不逐一貼出來。完整的 test class 可以到 [GitHub repo](https://github.com/ericksli/eta-demo/tree/main/app/src/test/java/net/swiftzer/etademo/domain) 查看。

下一篇我們會開始做 presentation layer。

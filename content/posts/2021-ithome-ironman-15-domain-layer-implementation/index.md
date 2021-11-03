---
title: "2021 iThome 鐵人賽 Day 15：Domain layer implementation"
tags:
  - 2021 iThome 鐵人賽
  - Android
date: 2021-09-30T00:00:00+08:00
canonicalURL: https://ithelp.ithome.com.tw/articles/10275114
---

> 本篇文章是 [2021 iThome 鐵人賽](https://ithelp.ithome.com.tw/2021ironman)參賽題目「[寫一個列車抵站時間 Android App](https://ithelp.ithome.com.tw/users/20139666/ironman/4661)」的第 15 篇，你可到 iThome [查看原文](https://ithelp.ithome.com.tw/articles/10275114)。
>
> [文章目錄]({{< ref "2021-ithome-ironman" >}})

經過這麼多集的 data layer 後，我們來到 domain layer。Domain layer 的用途是用來放 business logic，並向 presentation layer（即是 `Activity`、`Fragment`、`ViewModel`、layout XML 這層）提供一個表象 (façade) 去用跟 data 互動。你或許會問為甚麼我們不在 `ViewModel` 直接 call 之前寫好的 repository 而要經 domain layer，這是為了日後功能變更提供彈性。例如一個新聞 app 會有新聞列表頁跟新聞正文頁，當按下新聞列表的項目時就會進入正文頁。在正文頁按了收藏後返回列表頁就會發現剛才那個項目出現了一個收藏 icon。如果在 data layer 跟 presentation layer 之間直接接駁的話，那個列表頁在收藏狀態變動後自動更新的 logic 就會放入 `ViewModel` 內。（可能是正文頁在按下收藏 icon 時用 event bus 通知其他 `ViewModel` 去更新 UI。）如果再多幾個地方跟那個收藏狀態有關連的話那些觸發檢查更新的 code 就會放在各個 `ViewModel` 之中，日後收藏功能再有改動就很麻煩。另外，一些複雜的東西例如之前提及過的車費計算都不是單純在網路上 call API 然後稍加修飾就輸出去 UI 上，而是真的有 business logic 在 mobile app 內進行。那些 business logic 都是會放在 domain layer 入面。可能車費計算背後有很多的 class，但我們只外露幾個 use case 或者 interactor 讓 presentation layer call，這樣就把背後複雜的東西隱藏起來。

其實叫 interactor 或者 use case 我覺得沒有太大分別，反正它們都是把背後的東西（例如與 backend API、本地 database 互動、business logic）隱藏起來，並且將「收藏」功能這樣改一處地方就能通知另一處要更新的 logic 移離 presentation layer（另一個例子是收到 push notification 後要更新資料）。通常一個 use case 只會做一個動作，如果是要做齊[增刪查改](https://zh.wikipedia.org/wiki/%E5%A2%9E%E5%88%AA%E6%9F%A5%E6%94%B9) (create, read, update, delete) 的話那就會有四個 use case。有些人會偏好做一個通用的 use case interface（就像我們做 mapper 要有個 mapper interface 般要所有 mapper 都 implement 同一個 interface），但缺點是如果要傳遞參數的話就會變得很麻煩，每個 use case 都要做一個 data class 去載住參數（或者是用 `Map` 裝着）。所以我們這次是每一個 use case 都做一個專門的 interface。

## 車站列表

示範 app 會有兩個 use case，第一個是提供車站列表供用戶選取查閱班次的車站。首先準備它的 interface class `GetLinesAndStationsUseCase`：

```kotlin
interface GetLinesAndStationsUseCase {
    operator fun invoke(): Map<Line, Set<Station>>
}
```

用了 `operator fun invoke()` 是因為之後 ViewModel 可以把 variable 當 method call，這樣看起上來更簡潔。這個 syntax 在 Kotlin 叫 [operator overloading](https://kotlinlang.org/docs/operator-overloading.html)。以下就是例子：

```kotlin
val getLinesAndStations: GetLinesAndStationsUseCase

val result: Map<Line, Set<Station>> = getLinesAndStations()
```

然後是它的實作：

```kotlin
class GetLinesAndStationsUseCaseImpl @Inject constructor(
    private val repository: EtaRepository,
) : GetLinesAndStationsUseCase {
    override fun invoke(): Map<Line, Set<Station>> = repository.getLinesAndStations()
}
```

因為這個 use case 沒有特別的 logic 要處理，其實就是把 `EtaRepository` 拿到的東西左手交右手給 ViewModel 用。

## 抵站時間

另一個 use case 是取得車站的抵站時間，背後就是 call 港鐵的 API。首先我們都是要準備 `GetEtaUseCase` interface：

```kotlin
interface GetEtaUseCase {
    suspend operator fun invoke(
        language: Language,
        line: Line,
        station: Station,
        sortBy: SortBy,
    ): EtaResult

    enum class SortBy { DIRECTION, TIME }
}
```

這次 `operator fun` 我們加了 `suspend` keyword 和多了參數。Kotlin operator overloading 是容許這樣做的。

由於我們的示範 app 沒有做到強行更改 app locale，app 的界面語言是依據系統語言來顯示，所以 call API 時語言參數就由 presentation layer 提供。如果你的 app 有強行更改 app locale 的話，語言部分其實可以在 use case 實作內向查閱 shared preferences 或 data store 之類的儲存位置而無需由 presentation layer 提供。下面是它的實作：

```kotlin
class GetEtaUseCaseImpl @Inject constructor(
    private val repository: EtaRepository,
) : GetEtaUseCase {
    override suspend fun invoke(
        language: Language,
        line: Line,
        station: Station,
        sortBy: GetEtaUseCase.SortBy,
    ): EtaResult = when (val result = repository.getEta(language, line, station)) {
        is EtaResult.Success -> {
            val comparator: Comparator<EtaResult.Success.Eta> = when (sortBy) {
                GetEtaUseCase.SortBy.DIRECTION -> compareBy({ it.direction }, { it.sequence })
                GetEtaUseCase.SortBy.TIME -> compareBy({ it.time }, { it.sequence })
            }
            result.copy(schedule = result.schedule.sortedWith(comparator))
        }
        else -> result
    }
}
```

這次為了增加一些「business logic」，我們就加了一個排序功能。用戶可以把班次按行車方向（上行和下行）或者按時間排列。排序是用 `Comparator` 來做，用法就是在 `compareBy` 提供一個或多個 selector lambda。以 `SortBy.DIRECTION` 為例，`compareBy({ it.direction }, { it.sequence )}` 的意思是先按 `EtaResult.Success.Eta.direction` 排，然後再按 `EtaResult.Success.Eta.sequence` 排。排完之後我們直接把原先的 `EtaResult.Success` 內的 `schedule` 換成重新排序過的 list。

如果項目數量多的話，我們應該叫 backend 負責排；如果是由本地的 database 提供的話就應該由 database 負責排序。這次是因為港鐵 API 沒有提供這個功能而且項目數量少才會這樣寫。

## Dagger

最後，不要忘記加入 Dagger binding。我們這次開一個全新的 Dagger module，同樣地都是 `@InstallIn(SingletonComponent::class)`。

```kotlin
@Module
@InstallIn(SingletonComponent::class)
interface DomainModule {

    @Binds
    fun bindGetEtaUseCase(impl: GetEtaUseCaseImpl): GetEtaUseCase

    @Binds
    fun bindGetLinesAndStationsUseCase(impl: GetLinesAndStationsUseCaseImpl): GetLinesAndStationsUseCase
}
```

---

完整的 code 可以到 [GitHub repo](https://github.com/ericksli/eta-demo/tree/main/app/src/main/java/net/swiftzer/etademo/domain) 查閱，下一篇我們會幫這兩個 use case implementation 寫 unit test。

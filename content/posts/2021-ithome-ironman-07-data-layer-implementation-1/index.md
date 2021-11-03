---
title: "2021 iThome 鐵人賽 Day 7：Data layer implementation (1)"
tags:
  - 2021 iThome 鐵人賽
  - Android
date: 2021-09-22T00:00:00+08:00
canonicalURL: https://ithelp.ithome.com.tw/articles/10269457
---

> 本篇文章是 [2021 iThome 鐵人賽](https://ithelp.ithome.com.tw/2021ironman)參賽題目「[寫一個列車抵站時間 Android App](https://ithelp.ithome.com.tw/users/20139666/ironman/4661)」的第 7 篇，你可到 iThome [查看原文](https://ithelp.ithome.com.tw/articles/10269457)。
>
> [文章目錄]({{< ref "2021-ithome-ironman" >}})

在上一篇，我們把 Ktor client 加到 Dagger 的 object graph 內。現在我們就繼續寫 data layer 部分。

## 跨 layer 共用部分

不過在繼續之前，我們要先準備一些通用的 enum。這些 enum 分別是用來表示路綫、車站和語言。因為這次的示範 app 所用到的車站和路綫的數量有限，我們就簡化用 enum 寫死在 app 入面就算了，但如果數量多的話或許會改用 SQLite database 之類去儲存它們。在這種情況下，我們一般都會每個 layer 都有對應的 data class 表示，而不會好像現在把 data class/enum 放在 common  的地方。

```kotlin
enum class Line(val zh: String, val en: String) {
    AEL("機場快綫", "Airport Express"),
    TCL("東涌綫", "Tung Chung Line"),
    TML("屯馬綫", "Tuen Ma Line"),
    TKL("將軍澳綫", "Tseung Kwan O Line"),
}
```

```kotlin
enum class Station(val zh: String, val en: String) {
    UNKNOWN("", ""),

    HOK("香港", "Hong Kong"),
    KOW("九龍", "Kowloon"),
    OLY("奧運", "Olympic"),
    NAC("南昌", "Nam Cheong"),
    LAK("茘景", "Lai King"),
    TSY("青衣", "Tsing Yi"),
    SUN("欣澳", "Sunny Bay"),
    TUC("東涌", "Tung Chung"),
    AIR("機場", "Airport"),
    AWE("博覽館", "AsiaWorld-Expo"),

    WKS("烏溪沙", "Wu Kai Sha"),
    MOS("馬鞍山", "Ma On Shan"),
    HEO("恆安", "Heng On"),
    TSH("大水坑", "Tai Shui Hang"),
    SHM("石門", "Shek Mun"),
    CIO("第一城", "City One"),
    STW("沙田圍", "Sha Tin Wai"),
    CKT("車公廟", "Che Kung Temple"),
    TAW("大圍", "Tai Wai"),
    HIK("顯徑", "Hin Keng"),
    DIH("鑽石山", "Diamond Hill"),
    KAT("啟德", "Kai Tak"),
    SUW("宋皇臺", "Sung Wong Toi"),
    TKW("土瓜灣", "To Kwa Wan"),
    HOM("何文田", "Ho Man Tin"),
    HUH("紅磡", "Hung Hom"),
    ETS("尖東", "East Tsim Sha Tsui"),
    AUS("柯士甸", "Austin"),
    MEF("美孚", "Mei Foo"),
    TWW("荃灣西", "Tsuen Wan West"),
    KSR("錦上路", "Kam Sheung Road"),
    YUL("元朗", "Yuen Long"),
    LOP("朗屏", "Long Ping"),
    TIS("天水圍", "Tin Shui Wai"),
    SIH("兆康", "Siu Hong"),
    TUM("屯門", "Tuen Mun"),

    NOP("北角", "North Point"),
    QUB("鰂魚涌", "Quarry Bay"),
    YAT("油塘", "Yau Tong"),
    TIK("調景嶺", "Tiu Keng Leng"),
    TKO("將軍澳", "Tseung Kwan O"),
    HAH("坑口", "Hang Hau"),
    POA("寶琳", "Po Lam"),
    LHP("康城", "LOHAS Park"),
}
```

```kotlin
enum class Language {
    CHINESE,
    ENGLISH,
}
```

## Domain layer 準備工作

雖然本篇題目是 data layer，但因為 data layer 需要實作 domain layer 的 interface。所以我們要先定義好那些東西才能回到 data layer。

首先是 `EtaRepository` 的部分，裏面有兩個 function。一個是提供路綫和車站的關連，用來做揀選車站的 UI；另一個是 call API 取得某路綫的某車站的抵站時間。

```kotlin
interface EtaRepository {
    fun getLinesAndStations(): Map<Line, Set<Station>>
    suspend fun getEta(language: Language, line: Line, station: Station): EtaResult
}
```

`getLinesAndStations` 是普通 function 但 `getEta` 是 suspending function 是因為 `getLinesAndStations` return value 就是一個寫死的 `Map`，不會有耗時的工作。而 `getEta` 會 call API endpoint（非同步），所以就用 suspending function。

由於我們目標是顯示班次的目的地名稱、月台編號和倒數分鐘，所以不能直接拿 response 來用，要經過處理才可以拿去 UI 那邊用。

如果 response 的 `status` 是 `0` 的話，我們就用另一個 data class 表示目前綫路有事故。

至於 HTTP 429、HTTP 500、其餘的例外情況我們會另外用 object 和 data class 表示，這樣就可以不用 throw exception 去調用那一邊。

我們會把這幾款 class 都繼承同一個 sealed interface `EtaResult`，用來表達這個 API 所有可能輸出的結果。而交給 domain layer 調用的 method 會是一個 suspend fun 並回傳那個 sealed interface，那樣之後就算找不到 API 文檔只看 code 大概都會知道有甚麼情景。

```kotlin
sealed interface EtaResult {
    data class Success(
        val schedule: List<Eta> = emptyList(),
    ) : EtaResult {
        data class Eta(
            val direction: Direction = Direction.UP,
            val platform: String = "",
            val time: Instant = Instant.EPOCH,
            val destination: Station = Station.UNKNOWN,
            val sequence: Int = 0,
        ) {
            enum class Direction { UP, DOWN }
        }
    }

    object Delay : EtaResult

    data class Incident(
        val message: String = "",
        val url: String = "",
    ) : EtaResult

    object TooManyRequests : EtaResult

    object InternalServerError : EtaResult

    data class Error(val e: Throwable?) : EtaResult
}
```

有部分 class 用了 `object` 是因為它們沒有 field 在儲東西，所以全個 app 只用同一個 instance 都沒問題。

## 實作接駁 API endpoint

由於先前已經準備了 `EtaResponse` data class，我們可以把上一篇的 Ktor client 稍為改小許就可以完成今天要做的部分。

```kotlin
class EtaRepositoryImpl @Inject constructor(
    private val httpClient: HttpClient,
    private val etaResponseMapper: Mapper<HttpResponse, EtaResult>,
) : EtaRepository {
    override fun getLinesAndStations(): Map<Line, Set<Station>> = linkedMapOf(
        AEL to linkedSetOf(HOK, KOW, TSY, AIR, AWE),
        TCL to linkedSetOf(HOK, KOW, OLY, NAC, LAK, TSY, SUN, TUC),
        TML to linkedSetOf(
            WKS, MOS, HEO, TSH, SHM, CIO, STW, CKT, TAW, HIK, DIH, KAT, SUW, TKW,
            HOM, HUH, ETS, AUS, MEF, TWW, KSR, YUL, LOP, TIS, SIH, TUM,
        ),
        TKL to linkedSetOf(NOP, QUB, YAT, TIK, TKO, HAH, POA, LHP),
    )

    override suspend fun getEta(language: Language, line: Line, station: Station) = try {
        val response =
            httpClient.get<HttpResponse>("https://rt.data.gov.hk/v1/transport/mtr/getSchedule.php") {
                parameter("line", line.name)
                parameter("sta", station.name)
                parameter(
                    "lang", when (language) {
                        Language.CHINESE -> "TC"
                        Language.ENGLISH -> "EN"
                    }
                )
            }
        // 這部分我們下一篇會做
        etaResponseMapper.map(response)
    } catch (e: Throwable) {
        EtaResult.Error(e)
    }
}
```

`getLinesAndStations` 的實作沒甚麼特別，就是一個 immutable 的 `Map`。至於 `getEta` 其實就分為兩個部分：用 Ktor client call API 並取得 response 和把 API response 變成 domain layer 的 `EtaResult`（亦即是這個 function 的 return type）。

第一部分我們會用到上一篇所準備的 `DataModule` 入面的 `HttpClient`，我們只需要在 `EtaRepositoryImpl` constructor 加上 `@Inject` 並在 constructor 加上 `HttpClient` 的 parameter 就能讓 Dagger 幫我們取得 dependency。`etaResponseMapper` 我們會在下一篇處理。回到 `getEta` 裏面，第一句那個 `get` 基本上跟上一篇的寫法一樣，只是這次 `get` 那個 generic type 用了 `HttpResponse` 而不是 `EtaResponse`。這是因為我們要取得 HTTP response 的 status code。取得 response 後就交去 mapper 轉換成 `EtaResult`。另外，在整句調用 Ktor client 的部分包裹着 try-catch block，為的就是把不能連接網路之類的 exception 接住，那調用 repository 的一方不用再包 try-catch block，而且用了 sealed interface 亦能保證對方知悉這個情景從而寫對應的 code 處理。

在完成 `EtaRepositoryImpl` 後，我們要把 `EtaRepository` 和 `EtaRepositoryImpl` 的關連告訴給 Dagger 知道，否則 Dagger 看到有其他地方依賴 `EtaRepository` 時就不知道要給它甚麼實作。我們會用 `@Binds` 向 Dagger 指明凡是其他地方依賴 `EtaRepository` 我們都會給它 instantiate 一個新的 `EtaRepositoryImpl`。`@Binds` 跟 `@Provides` 不同的地方是 `@Binds` 是 `@Provides` 的簡化版，如果要指明依賴某個 abstract class 或 interface 時要 instantiate 那個 concrete implementation 的話，我們就可以用 `@Binds` 寫一個 abstract function，Dagger 就會幫我們補上那個 abstract function 的實作。注意的是如果那個 concrete implementation 的 class 的 constructor 也是需要用到其他 dependency，你的 constructor 亦都需要加上 `@Inject`，這樣 Dagger 才能替你自動替你做 dependency injection。但如果那個 class 是人家寫的（即是 library 提供你又不能幫它加 `@Inject`）的話，你就只能用 `@Provides` 並在 method parameter 自行補上要用到的 dependency，然後在 method body 用那些 method parameter 來 instantiate 那個 class 並將它 return 出去，就像之前準備 Ktor client 般的做法。

```kotlin
@Module
@InstallIn(SingletonComponent::class)
interface DataModule {

    @BindsOptionalOf
    fun bindLogging(): HttpClientFeature<Logging.Config, Logging>

    // 本節新加的部分
    @Binds
    fun bindEtaRepository(impl: EtaRepositoryImpl): EtaRepository

    // 之前準備 Ktor client/OkHttp 所寫的東西
    companion object {
        // ...
    }
}
```

## 跟 Retrofit 的比較

如果是用 OkHttp 及 Retrofit 的話，寫法會比較簡潔一點。因為那些 parameter 和 API endpoint 的 URL 都是寫在 interface function 的 Retrofit 專門 annotation。現在我們用 Ktor client 感覺上會像直接用 OkHttp 發送 HTTP request 般，但有一點不同是 Ktor client 還是有跟 JSON deserialization library 有整合，只需一句 `httpClient.get<EtaResponse>` 就能拿到 deserialize 後的 object。如果你的 app 要駁不同的 API，每個的 base URL 都不相同的話或許用 Ktor client 這種寫法比起 Retrofit 要寫 interface 還要方便。

下一篇我們會實作 `EtaResponseMapper`。

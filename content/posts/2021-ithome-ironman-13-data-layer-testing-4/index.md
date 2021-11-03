---
title: "2021 iThome 鐵人賽 Day 13：Data layer testing (4)"
tags:
  - 2021 iThome 鐵人賽
  - Android
date: 2021-09-28T00:00:00+08:00
canonicalURL: https://ithelp.ithome.com.tw/articles/10273796
---

> 本篇文章是 [2021 iThome 鐵人賽](https://ithelp.ithome.com.tw/2021ironman)參賽題目「[寫一個列車抵站時間 Android App](https://ithelp.ithome.com.tw/users/20139666/ironman/4661)」的第 13 篇，你可到 iThome [查看原文](https://ithelp.ithome.com.tw/articles/10273796)。
>
> [文章目錄]({{< ref "2021-ithome-ironman" >}})

上一篇示範了 Ktor mock engine 的設定和測試了如果出現 exception 時能否順利地處理。現在就測試 `getEta` 輸出班次的情景。

Test case 的目標是：

1. 檢查交去 Ktor client 的 request parameter（即是語言、路綫、車站）是否正確
2. 看看加在 Ktor client 的 Kotlinx serialization 能否正常地把我們提供的 response JSON 轉成 `EtaResponse`

由於我們已針對 `EtaResponseMapper` 寫了 unit test，我們就乾脆 mock 那個 mapper 然後隨便 return 一個 `EtaResult.Success` 就算了。當然你亦可以 instantiate 一個真的 mapper 來做轉換，因為本身有 mapper 的 unit test，所以即使 repository test 有錯都可以容易剔除 mapper 有錯這個因素。

同樣地，我們都是用之前寫的 `mockHttpClient` 來假裝 server response。首先我們會檢查 HTTP request 的 query parameter 是否正確。

```kotlin
@Test
fun `getEta normal`() {
    runBlocking {
        val (client, requestSlot) = mockHttpClient(
            HttpStatusCode.OK,
            "api/schedule_tkl_tko_normal.json"
        )
        val repository = EtaRepositoryImpl(client, etaResponseMapper)
        val responseSlot = slot<HttpResponse>()
        coEvery { etaResponseMapper.map(capture(responseSlot)) } returns EtaResult.Success()
        val result = repository.getEta(Language.ENGLISH, Line.TKL, Station.TKO)
        expectThat(requestSlot).assert("EN", "TKL", "TKO")

        // 檢查 deserialize 後的 EtaResponse 會在稍後提供

        expectThat(result).isA<EtaResult.Success>()
    }
}
```

那句 `expectThat(requestSlot).assert("EN", "TKL", "TKO")` 就是檢查 query parameter。但那個 `assert("EN", "TKL", "TKO")` 是甚麼來的？我在 Strikt 找不到呢。其實這是我另外寫的 function。

之前我們示範了用 Strikt 寫巢狀的 assertion（就是一層層 object 走入去檢查），雖然在 fail 時能得到清晰的錯誤訊息，但如果每個 test case 都這樣寫就變得很長。Strikt 其實是可以寫[自定義的 assertion](https://strikt.io/wiki/custom-assertions/)。本身 assertion function 就是 `Assertion.Builder<T>` 的 [extension function](https://kotlinlang.org/docs/extensions.html)，只要我們學它寫 extension function 就能做到類似 Strikt 提供的 assertion function。那個 `assert("EN", "TKL", "TKO")` 其實是這樣寫的：

```kotlin
@JvmName("assert_CapturingSlot_HttpRequestData")
private fun Assertion.Builder<CapturingSlot<HttpRequestData>>.assert(
    lang: String,
    line: String,
    sta: String,
): Assertion.Builder<CapturingSlot<HttpRequestData>> = withCaptured {
    get { url.parameters }.and {
        get(Parameters::names).containsExactlyInAnyOrder("lang", "line", "sta")
        get { get("lang") }.isEqualTo(lang)
        get { get("line") }.isEqualTo(line)
        get { get("sta") }.isEqualTo(sta)
    }
}
```

這段 code 的大意是檢查 `HttpRequestData.url.parameters` 的 `names` 是不是有齊 `lang`、`line`、 `sta`。而它們三個 parameter 的值分別是參數指明的值。它的寫法基本上和在 test case 直接寫那些 assertion 一樣，只是把它們抽出來放在一個 function 裏面，那下次在其他 test case 寫同樣的 assertion 就不用寫那麼長。

而 function 的 `@JvmName("assert_CapturingSlot_HttpRequestData")` 是用來告訴 compiler 要把這個 function 在 JVM bytecode 更名為 `assert_CapturingSlot_HttpRequestData`。原因是我們之後會再有其他的 `assert` function，但 `Assertion.Builder` 的 generic type 不同。因為 Java 有 type erasure 的特性，compile 出來的 JVM bytecode return type 只會是 `Assertion.Builder` 而不是 `Assertion.Builder<CapturingSlot<HttpRequestData>>`。所以再寫多幾個 `assert` function 就很容易撞名（跟其他 method signature 相同）。所以 Kotlin 有 `@JvmName` annotation 讓你改變 function 名來避免撞名問題。其實 type erasure 是因為令在 generic 功能出現前所 compile 出來的 bytecode 向後相容，因為 generic 不是 Java「自古以來」就有的功能，所以 generic type 只會在 compile 時檢查一下，但 bytecode 是不會記錄那個 generic type。

然後我們就可以用同樣方法做 `EtaResponse` 的 custom assertion function。我們先寫班次那個 assertion function（即是 `EtaResponse.Eta`）：

```kotlin
@JvmName("assert_EtaResponse_Eta")
private fun Assertion.Builder<EtaResponse.Eta>.assert(
    plat: String,
    time: String,
    dest: String,
    seq: String,
): Assertion.Builder<EtaResponse.Eta> = and {
    get(EtaResponse.Eta::plat).isEqualTo(plat)
    get(EtaResponse.Eta::time).isEqualTo(time)
    get(EtaResponse.Eta::dest).isEqualTo(dest)
    get(EtaResponse.Eta::seq).isEqualTo(seq)
}
```

這個應該沒甚麼特別，之後再看看上一層 `EtaResponse` 的 custom assertion：

```kotlin
@JvmName("assert_EtaResponse")
private fun Assertion.Builder<EtaResponse>.assert(
    status: Int,
    message: String,
    url: String,
    isDelay: String,
    dataKey: String? = null,
    upAssertionBlock: Assertion.Builder<List<EtaResponse.Eta>>.() -> Unit = {},
    downAssertionBlock: Assertion.Builder<List<EtaResponse.Eta>>.() -> Unit = {},
): Assertion.Builder<EtaResponse> = and {
    get(EtaResponse::status).isEqualTo(status)
    get(EtaResponse::message).isEqualTo(message)
    get(EtaResponse::url).isEqualTo(url)
    get(EtaResponse::isDelay).isEqualTo(isDelay)
    if (dataKey == null) {
        get(EtaResponse::data).isEmpty()
    } else {
        get(EtaResponse::data).hasSize(1).and {
            get(dataKey).isNotNull().and {
                upAssertionBlock(get(EtaResponse.Data::up))
                downAssertionBlock(get(EtaResponse.Data::down))
            }
        }
    }
}
```

大部分的 code 都是拿某個 property 再做 assertion。不過特別的地方是 function parameter 的 `upAssertionBlock` 和 `downAssertionBlock`。`EtaResponse` 有兩個 `List<EtaResponse.Eta>` 的 property（用來表示上行和下行的班次），我們打算用剛才的 custom assertion 來做 assertion。`upAssertionBlock` 和 `downAssertionBlock` 兩個 lambda 就是用來放針對 `List<EtaResponse.Eta>` 的 assertion（我們會在 lambda 入面 call 剛才那個 custom assertion）。

至於 `dataKey` 要檢查是否 null 是因為有時候 response 的 `data: Map<String, Data>` 可以是 empty map，如果我們交了 null 的 `dataKey` 那就檢查 `data` 是不是 empty map，否則就檢查那個 map 是不是有那一個 entry 然後就調用 `upAssertionBlock` 和 `downAssertionBlock` 兩個 lambda，把 `Assertion.Builder<List<EtaResponse.Eta>>` 交去 lambda 內。

其實這個 custom assertion 有 lambda 的 parameter，如果[考慮到效能問題](https://www.baeldung.com/kotlin/inline-functions)的話可以改成 [inline function](https://kotlinlang.org/docs/inline-functions.html)。

所以最後 `EtaResponse` 的 assertion 會是這樣：

```kotlin
expectThat(responseSlot.captured.receive<EtaResponse>()).assert(
    status = EtaResponse.STATUS_NORMAL,
    message = "successful",
    url = "",
    isDelay = EtaResponse.IS_DELAY_FALSE,
    dataKey = "TKL-TKO",
    upAssertionBlock = {
        hasSize(4).and {
            get(0).assert("1", "2020-01-11 14:28:00", "POA", "1")
            get(1).assert("1", "2020-01-11 14:32:00", "POA", "2")
            get(2).assert("1", "2020-01-11 14:36:00", "LHP", "3")
            get(3).assert("1", "2020-01-11 14:38:00", "POA", "4")
        }
    },
    downAssertionBlock = {
        hasSize(4).and {
            get(0).assert("2", "2020-01-11 14:26:00", "NOP", "1")
            get(1).assert("2", "2020-01-11 14:29:00", "NOP", "2")
            get(2).assert("2", "2020-01-11 14:35:00", "NOP", "3")
            get(3).assert("2", "2020-01-11 14:37:00", "TIK", "4")
        }
    }
)
```

因為寫法都是大同小異，所以就不再逐一介紹。其餘的 [test case](https://github.com/ericksli/eta-demo/blob/main/app/src/test/java/net/swiftzer/etademo/data/EtaRepositoryImplTest.kt) 和 [JSON response 檔](https://github.com/ericksli/eta-demo/tree/main/app/src/test/resources/api)可以到 GitHub repo 查閱，而 data layer 的 unit test 部分亦告一段落。

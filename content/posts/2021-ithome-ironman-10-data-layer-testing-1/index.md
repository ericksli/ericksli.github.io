---
title: "2021 iThome 鐵人賽 Day 10：Data layer testing (1)"
tags:
  - 2021 iThome 鐵人賽
  - Android
date: 2021-09-25T00:00:00+08:00
canonicalURL: https://ithelp.ithome.com.tw/articles/10271848
---

> 本篇文章是 [2021 iThome 鐵人賽](https://ithelp.ithome.com.tw/2021ironman)參賽題目「[寫一個列車抵站時間 Android App](https://ithelp.ithome.com.tw/users/20139666/ironman/4661)」的第 10 篇，你可到 iThome [查看原文](https://ithelp.ithome.com.tw/articles/10271848)。
>
> [文章目錄]({{< ref "2021-ithome-ironman" >}})

在切回去寫 domain layer 之前，我們先把之前寫好的 data layer class 補回 unit test。在開始寫之前，我們要先加入一些 testing 會用到的 dependency（[Strikt](https://strikt.io/) 和 [MockK](https://mockk.io/)）：

```groovy
dependencies {
    testImplementation platform("io.strikt:strikt-bom:$striktVersion")
    testImplementation "io.strikt:strikt-core"
    testImplementation "io.strikt:strikt-mockk"

    testImplementation "io.mockk:mockk:$mockkVersion"
    androidTestImplementation "io.mockk:mockk-android:$mockkVersion"
}
```

Strikt 是 assertion library，就是用來檢驗那個 variable 是不是 null、等於甚麼、如果是 collection 的話亦可以檢驗裏面是不是有這些項目，是不是完全按照那個順序等等的 library。類似的 library 有 [kotlin.test](https://kotlinlang.org/api/latest/kotlin.test/)、[Truth](https://truth.dev/)、[Hamcrest](http://hamcrest.org/) 和 [AssertJ](https://joel-costigliola.github.io/assertj/)。其實用甚麼 assertion library 都是個人喜好，只要做到 assertion 而且有錯時能指出清晰的錯處就可以了。至於 MockK 是 Kotlin 的 mock library，它就是用來把 class 或 interface 做一個假的版本，那你就可以控制它回傳甚麼，這種做法在 unit testing 時很常用到。

現在我們就寫第一個 unit test：`EtaResponseMapperTest`。

`EtaResponseMapper` 只有一個 public 的 function：`map`。那我們的目標是通過傳入不同的 `HttpResponse` 從而把所有使用 `EtaResponseMapper` 的情景都試一次。另一個講法是要把整個 `EtaResponseMapper` 的所有分支都走遍一次，亦即是指 branch coverage。

首先我們先寫 unit test class 最基本的東西：

```kotlin
class EtaResponseMapperTest {

    private lateinit var mapper: Mapper<HttpResponse, EtaResult>

    @Before
    fun setUp() {
        mapper = EtaResponseMapper()
    }

    @Test
    fun `internal server error`() {
    }
}
```

我們第一個 test 就試 Internal Server Error 的情景。Function 名為方便閱讀我用了 backtick 包住，這樣 function 名就可以有 space character。

```kotlin
@Test
fun `internal server error`() = runBlockingTest {
    val response = mockk<HttpResponse>()
    val httpClientCall = mockk<HttpClientCall>()
    every { response.status } returns HttpStatusCode.InternalServerError
    every { response.call } returns httpClientCall
    coEvery { httpClientCall.receive<EtaResponse>() } returns EtaResponse(
        status = EtaResponse.STATUS_ERROR_OR_ALERT,
        message = "Error",
    )
    expectThat(mapper.map(response)).isA<EtaResult.InternalServerError>()
}
```

包住 `runBlockingTest` 是因為我們會在入面 call suspended function。

其實 mapper 會用到 `HttpResponse` 的 `status` 和 `receive`。所以我們就要針對這兩個東西來換成自己想要的東西，令到我們可以讓程式是做到 Internal Server Error 的情景。

首先 `mockk<HttpResponse>()` 的意思是做一個假的 `HttpResponse`。然後 `every { response.status } returns HttpStatusCode.InternalServerError` 就是說凡是執行 `response.status` 都會回傳 `HttpStatusCode.InternalServerError`。那就是滿足 `EtaResponseMapper` 入面的 `when (o.status) { ... }` 能進去 `HttpStatusCode.InternalServerError` 的部分。

其實要試 `InternalServerError` 的話，是不用再 mock 其他東西。但為了其他 test case，我們會示範 mock 拿 response data class 的部分。

看看 `HttpStatusCode.OK` 的部分，它會 call `receive`。一般來說我們會 mock 那個 receive 讓它回傳我們想看到的東西。不過，當我們看一看那個 `receive` 的話，就會發現它是一個 inline function：

```kotlin
public suspend inline fun <reified T> HttpResponse.receive(): T = call.receive(typeInfo<T>()) as T
```

Inline function 的意思是 compile 時 Kotlin compiler 會將那個 function 內容抄到 call 那個 function 的位置，之後就沒有那個 function 的㾗跡。所以我們再追蹤那個 `call` 和 `receive`。

```kotlin
public abstract val call: HttpClientCall
```

```kotlin
public suspend fun receive(info: TypeInfo): Any
```

這次不是 inline function，那我們可以 mock 了。首先是要 mock `HttpClientCall` 讓 `response.call` 回傳我們另一個假的 `HttpClientCall`。之後因為之前的 inline function 會 call `receive` 取得 response data class，而 `receive` 是一個 suspended function，我們要用 MockK 的 `coEvery` 控制它回傳我們想要的 object。

```kotlin
val httpClientCall = mockk<HttpClientCall>()
every { response.call } returns httpClientCall
coEvery { httpClientCall.receive<EtaResponse>() } returns EtaResponse(
    status = EtaResponse.STATUS_ERROR_OR_ALERT,
    message = "Error",
)
```

因為控制 HTTP client 的 response 是每一個 test 都會做的東西，我們就把這幾句抽取成為一個 function：

```kotlin
private fun mockHttpResponse(
    statusCode: HttpStatusCode,
    etaResponse: EtaResponse
): HttpResponse {
    val response = mockk<HttpResponse>()
    val httpClientCall = mockk<HttpClientCall>()
    every { response.status } returns statusCode
    every { response.call } returns httpClientCall
    coEvery { httpClientCall.receive<EtaResponse>() } returns etaResponse
    return response
}
```

最後先前那個 test 就可以變成這樣：

```kotlin
@Test
fun `internal server error`() = runBlockingTest {
    val response = mockHttpResponse(
        statusCode = HttpStatusCode.InternalServerError,
        etaResponse = EtaResponse(
            status = EtaResponse.STATUS_ERROR_OR_ALERT,
            message = "Error",
        ),
    )
    expectThat(mapper.map(response)).isA<EtaResult.InternalServerError>()
}
```

最後一句 `expectThat` 是 Strikt 的寫法，`expectThat` 入面放的是要檢驗的項目，然後就可以繼續串接 Strikt 的 method 就能針對它做檢驗。

因為篇幅有點長，我們在下一篇示範正常輸出班次的情景。

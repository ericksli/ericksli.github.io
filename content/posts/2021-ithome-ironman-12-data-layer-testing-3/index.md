---
title: "2021 iThome 鐵人賽 Day 12：Data layer testing (3)"
tags:
  - 2021 iThome 鐵人賽
  - Android
date: 2021-09-27T00:00:00+08:00
canonicalURL: https://ithelp.ithome.com.tw/articles/10273081
---

> 本篇文章是 [2021 iThome 鐵人賽](https://ithelp.ithome.com.tw/2021ironman)參賽題目「[寫一個列車抵站時間 Android App](https://ithelp.ithome.com.tw/users/20139666/ironman/4661)」的第 12 篇，你可到 iThome [查看原文](https://ithelp.ithome.com.tw/articles/10273081)。
>
> [文章目錄]({{< ref "2021-ithome-ironman" >}})

上一篇我們寫好了 `EtaResponseMapper` 的 unit test。但 data layer 還有 `EtaResponseMapper` 未寫 unit test。今天我們就寫這一個 class 的 unit test。

## Logback 的特別設定

我們先前在設定 Ktor client 時幫它加了 logging 功能，這樣我們就可以在 Logcat 看到 request 和 response 的資訊，方便 debug。而它所用的 logger 是按照 Simple Logging Facade for Java (SLF4J) 規格。SLF4J 其實是一個 Java 有名的 logging interface，如果一些組件或 library 想用 logging 功能的話，它們可以用 SLF4J 的 interface 發送 log 到 logger，但最終所用的 logger 是由用那些組件的一方控制。這樣就不會把 log 亂射和可以把 log 集中處理。在 Android 的話，我們會用 [logback-android](https://github.com/tony19/logback-android)。

把這段內容放到本篇才說是因為我們會用到 Ktor 的 mock client。如果我們只加了 logback-android 的話，在執行 unit test 時就會出現以下錯誤（節錄）：

```text
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/C:/Users/Eric/.gradle/caches/transforms-3/ff97545d615cededbad0c653ea1a09c7/transformed/jetified-logback-android-2.0.0-runtime.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/C:/Users/Eric/.gradle/caches/transforms-3/afd61d02052ffb4feaec186ea7b45062/transformed/jetified-logback-android-2.0.0/jars/classes.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
Failed to instantiate [ch.qos.logback.classic.LoggerContext]
Reported exception:
java.lang.RuntimeException: Method getExternalStorageState in android.os.Environment not mocked. See http://g.co/androidstudio/not-mocked for details.
    at android.os.Environment.getExternalStorageState(Environment.java)
    at ch.qos.logback.core.android.AndroidContextUtil.getMountedExternalStorageDirectoryPath(Unknown Source)
    at ch.qos.logback.core.android.AndroidContextUtil.setupProperties(Unknown Source)
    at ch.qos.logback.classic.util.ContextInitializer.autoConfig(Unknown Source)
    at org.slf4j.impl.StaticLoggerBinder.init(Unknown Source)
    at org.slf4j.impl.StaticLoggerBinder.<clinit>(Unknown Source)
    at org.slf4j.LoggerFactory.bind(LoggerFactory.java:150)
    at org.slf4j.LoggerFactory.performInitialization(LoggerFactory.java:124)
    at org.slf4j.LoggerFactory.getILoggerFactory(LoggerFactory.java:417)
    at org.slf4j.LoggerFactory.getLogger(LoggerFactory.java:362)
    at org.slf4j.LoggerFactory.getLogger(LoggerFactory.java:388)
    at io.ktor.client.features.logging.LoggerJvmKt$DEFAULT$1.<init>(LoggerJvm.kt:13)
...
SLF4J: Actual binding is of type [ch.qos.logback.classic.util.ContextSelectorStaticBinder]
```

這是因為 logback-android 會嘗試找 Android app 的 assets 目錄內的 *logback.xml*，這個 XML 檔是用來設定 logger。在執行 app 時是完全正常，但在執行 unit test 時就因為當前執行環境沒有 Android SDK（就是那個 `android.os.Environment.getExternalStorageState`）而彈出錯誤訊息。要解決它其中一個方法是用 [Robolectric](http://robolectric.org/)，因為 Robolectric 可以幫我們補回 `getExternalStorageState`。但為了一個 logger 而要在一個本身不會接觸到 Android SDK 的地方的 unit test 加 Robolectric 實在是太大陣仗，執行 unit test 時只是下載 Robolectric 的 JAR 都用了好幾分鐘。所以我們用另一個方法：就是在 unit test 是[把 logback-android 換成 logback-classic](https://github.com/tony19/logback-android/issues/151)，那就不會在 unit test 時 call 了 Android SDK 的 method。按照 GitHub issue 的建議，我們把 *build.gradle* 改成這樣：

```groovy
dependencies {
    implementation "com.github.tony19:logback-android:$logbackAndroidVersion"
}

configurations.all { config ->
    if (config.name.toLowerCase().contains('test')) {
        config.resolutionStrategy.dependencySubstitution {
            substitute module("com.github.tony19:logback-android:$logbackAndroidVersion") with module("ch.qos.logback:logback-classic:$logbackVersion")
        }
    }
}
```

這樣就不會出現錯誤訊息。

## `getStations`

這個 test 寫法很簡單，因為它就是回傳一個寫死的 `Map`。

```kotlin
class EtaRepositoryImplTest {

    @MockK
    private lateinit var etaResponseMapper: Mapper<HttpResponse, EtaResult>

    @Before
    fun setUp() {
        MockKAnnotations.init(this)
    }

    @Test
    fun getStations() {
        val client = mockk<HttpClient>()
        val repository = EtaRepositoryImpl(client, etaResponseMapper)
        expectThat(repository.getLinesAndStations()).hasSize(4)
    }
}
```

在 `setUp`，這次我們多了一個新的寫法：`MockKAnnotations.init(this)`。首先，因為 `setUp` 加了 `@Before`，所以這個 function 每次執行 test case 前都會行一次。而 `MockKAnnotations.init(this)` 的意思是叫 MockK 把所有加了 `@MockK` 的 property 都初始化，即是代你執行了 `mockk<XXX>()`，[Mockito](https://site.mockito.org/) 都有[類似用法](https://javadoc.io/doc/org.mockito/mockito-core/latest/org/mockito/Mock.html)。順帶一提，如果想每次行完一個 test case 之後都執行一些東西可以把它們放到 `@After` 的 function 內。

由於 `repository.getLinesAndStations` 回傳的東西是固定不變，我們就不把傳回的 `Map` 內容逐一檢查，因為這和罰抄一次沒甚麼分別。所以簡單地檢查是不是有四個項目就算了。接着就開始準備寫 `getEta` 的 test case。

## Ktor Mock Engine

`EtaRepositoryImpl` 的 constructor 有兩個 dependency：`HttpClient` 和 `Mapper<HttpResponse, EtaResult>`。後者我們之前已經寫好了 unit test，前者是我們這次測試的重點：看看 server 的 response 能否順利地被 deserialize 為 `EtaResponse` 和把 `HttpResponse` 交到 `EtaResponseMapper` 處理。現實上的 production server 會在不同時段返回不同的結果，例如只會在事故時提供事故的結果。所以我們要先準備一個假 server，然後 response 換做我們預先準備好的 JSON 檔案，這樣就可以試到不同的情景又不用大費周章部署一個測試專用的 server。

我們會把 [response body JSON 檔案](https://github.com/ericksli/eta-demo/tree/main/app/src/test/resources/api)放到 *app/src/test/resources/api* 內，留意要放在 *test* 的 *resources* 目錄內，這樣就不會在 APK/AAB 找到這些檔案。

之後我們就準備 mock server 的部分。和 OkHttp 一樣，Ktor client 都有提供測試專用的 artifact `ktor-client-mock`。它提供了 `MockEngine` 可以讓我們指定 response 內容而不會把 request 發送出去。

下面是 Ktor 網站提供的 `MockEngine` 示範：

```kotlin
class ApiClientTest {
    @Test
    fun sampleClientTest() {
        runBlocking {
            val mockEngine = MockEngine { request ->
                respond(
                    content = ByteReadChannel("""{"ip":"127.0.0.1"}"""),
                    status = HttpStatusCode.OK,
                    headers = headersOf(HttpHeaders.ContentType, "application/json")
                )
            }
            // ApiClient 內會 call httpClient.get
            val apiClient = ApiClient(mockEngine)

            Assert.assertEquals("127.0.0.1", apiClient.getIp().ip)
        }
    }
}
```

用法就是把原先傳入去 `HttpClient(engine)` 的 engine 換成 `MockEngine`。而 `MockEngine` 可以設定它會 response 甚麼的內容。由於我們幾乎每一個 test case 都會用到 `HttpClient`，所以我們把準備 `HttpClient` 的部分抽成一個 function 方便 call。

```kotlin
private fun mockHttpClient(
    status: HttpStatusCode,
    resourceName: String,
    exception: Exception? = null,
): Pair<HttpClient, CapturingSlot<HttpRequestData>> {
    val requestBlock = mockk<(HttpRequestData) -> Unit>()
    val requestSlot = slot<HttpRequestData>()
    every { requestBlock(capture(requestSlot)) } just Runs
    val engine = MockEngine { request ->
        requestBlock(request)
        if (exception != null) throw exception
        respond(
            content = ByteReadChannel(
                javaClass.classLoader?.getResourceAsStream(resourceName)
                    ?.readBytes() ?: ByteArray(0)
            ),
            status = status,
            headers = headersOf(HttpHeaders.ContentType, "application/json"),
        )
    }
    return DataModule.provideKtorHttpClient(
        engine = engine,
        logging = Optional.empty()
    ) to requestSlot
}
```

和 Ktor 網站的寫法相比我們寫得比它複雜，因為：

1. 把 `HttpRequestData` 放出來做 assertion（因為每次網址都會因應 `EtaRepository.getEta` 的參數而有所不同）
2. 要模擬裝置不能上網時會 throw exception
3. response body 靠讀取放在 resources 目錄的 JSON 檔案提供

第一部分就是靠回傳出來的 Pair 交給 caller（交 `requestSlot` 給 caller 做 assertion）；第二部分就是靠 `exception` 參數是不是 null 來決定會不會 throw exception；第三部分就是用 `javaClass.classLoader.getResourceAsStream` 讀取指定檔案。

## `getEta` throw exception

現在我們測試如果 `getEta` throw exception 時會不會回傳 `EtaResult.Error`：

```kotlin
@Test
fun `getEta throw exception`() {
    runBlocking {
        val (client, _) = mockHttpClient(
            HttpStatusCode.OK,
            "api/schedule_incident.json",
            RuntimeException("Something went wrong")
        )
        val repository = EtaRepositoryImpl(client, etaResponseMapper)
        val result = repository.getEta(Language.CHINESE, Line.TKL, Station.TKO)
        expectThat(result).isA<EtaResult.Error>().and {
            get(EtaResult.Error::e).isA<RuntimeException>()
        }
    }
}
```

檢查 request URL 的部分我們會在其他 test case 做。你或許會留意到我們今次不是用 `runBlockingTest` 而是用 `runBlocking`。因為這次用 `runBlocking` 會出現以下的錯誤：

```text
This job has not completed yet
java.lang.IllegalStateException: This job has not completed yet
    at kotlinx.coroutines.JobSupport.getCompletionExceptionOrNull(JobSupport.kt:1190)
    at kotlinx.coroutines.test.TestBuildersKt.runBlockingTest(TestBuilders.kt:53)
    at kotlinx.coroutines.test.TestBuildersKt.runBlockingTest$default(TestBuilders.kt:45)
    at net.swiftzer.etademo.data.EtaRepositoryImplTest.getEta normal(EtaRepositoryImplTest.kt:45)
```

出現「This job has not completed yet」的原因應該是 [Ktor 開了新 thread 做 HTTP request/respond](https://github.com/Kotlin/kotlinx.coroutines/issues/1204)。但我找不到地方讓我換走 `Executor` 或者 `Dispatcher`（但 `HttpClientEngineConfig` 可以改變 `threadsCount`）。而 Ktor 網站的示範亦都是用 `runBlocking`，可能它真的沒有辦法用 `runBlockingTest`。

## 參考

- [Unit Testing Coroutine Suspend Functions using `TestCoroutineDispatcher`](https://craigrussell.io/2019/11/unit-testing-coroutine-suspend-functions-using-testcoroutinedispatcher/)
- [Testing Coroutines on Android (Android Dev Summit '19)](https://www.youtube.com/watch?v=KMb0Fs8rCRs)

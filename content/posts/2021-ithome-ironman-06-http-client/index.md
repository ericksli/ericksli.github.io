---
title: "2021 iThome 鐵人賽 Day 6：HTTP Client"
tags:
  - 2021 iThome 鐵人賽
  - Android
date: 2021-09-21T00:00:00+08:00
canonicalURL: https://ithelp.ithome.com.tw/articles/10268767
---

> 本篇文章是 [2021 iThome 鐵人賽](https://ithelp.ithome.com.tw/2021ironman)參賽題目「[寫一個列車抵站時間 Android App](https://ithelp.ithome.com.tw/users/20139666/ironman/4661)」的第 6 篇，你可到 iThome [查看原文](https://ithelp.ithome.com.tw/articles/10268767)。
>
> [文章目錄]({{< ref "2021-ithome-ironman" >}})

在 Android 開發如果要用到 HTTP client 的話基本上大家都預設用 [OkHttp](https://square.github.io/okhttp/) + [Retrofit](https://square.github.io/retrofit/) 這個組合。這次我們試試一些新東西：[Ktor](https://ktor.io/)。

Ktor 是 JetBrains 出的 server library，就是用來開發 server side 的 web application。但它的功能比較簡單。不過我們不是用它的 server library，是用它的 client library。近年來 Kotlin 推廣用 Kotlin 寫跨平台應用（網頁、Android、iOS、backend），在 mobile app 那邊叫 Kotlin Multiplatform Mobile (KMM)，它就是要用 Kotlin 來寫 Android 和 iOS 共用的部分（通常就是 business logic、接駁 backend 那部分），至於 UI 的部分就各自用回該平台的方法寫。正因為共通的部分必須要用純 Kotlin 來寫，code 不能引用 Java Standard Library 的東西，所以 OkHttp 和 Retrofit 就不能直接在 KMM 上面用，取而代之就是 Ktor Client。

因應不同平台實際處理 HTTP request 的 client（Ktor 稱為 engine）各有不同，Ktor 把這些 HTTP client 封裝了一次。例如在 Android 可以用 OkHttp、CIO，在 iOS 就是用 [`NSURLSession`](https://developer.apple.com/documentation/foundation/nsurlsession)。所以在建立 Ktor client 時要因應不同平台有不同的設定，但你調用 Ktor 的地方就不用加那些 `if (Android) { ... }` 的東西。

以下是我們這次會用到的 dependency：

```groovy
implementation "io.ktor:ktor-client-core:$ktorVersion"
implementation "io.ktor:ktor-client-okhttp:$ktorVersion"
implementation "io.ktor:ktor-client-logging:$ktorVersion"
implementation "io.ktor:ktor-client-serialization:$ktorVersion"
testImplementation "io.ktor:ktor-client-mock:$ktorVersion"
implementation "com.squareup.okhttp3:okhttp:$okhttpVersion"

implementation "org.jetbrains.kotlinx:kotlinx-serialization-json:$kotlinSerializationVersion"

// logging the HTTP request and response
implementation "org.slf4j:slf4j-api:$slf4jVersion"
implementation "com.github.tony19:logback-android:$logbackAndroidVersion"
```

## Ktor Client 基本用法

如果我們甚麼都不理，只是單純用 Ktor client call API 的話，大概會是這樣：

```kotlin
val httpClient = HttpClient(OkHttp) {
    expectSuccess = true
    install(Logging) {
        logger = Logger.DEFAULT
        level = LogLevel.ALL
    }
    install(JsonFeature) {
        serializer = KotlinxSerializer(kotlinx.serialization.json.Json {
            coerceInputValues = true
            ignoreUnknownKeys = true
        })
    }
}
val response: EtaResponse = httpClient.get<EtaResponse>("https://rt.data.gov.hk/v1/transport/mtr/getSchedule.php?line=TML&sta=TIS&lang=TC")
```

設定 Ktor client 的寫法用了很多 lambda，就像那些 *build.gradle* 般做了專門而設的 DSL。上面的 HTTP client 設定就是幫它加了 logging 和用 Kotlin Serialization 做 JSON deserialization。

然後發送 HTTP request 就是簡單一句 `httpClient.get<EtaResponse>` 就能拿到 deserialize 好的 response data class object。

如果見到網址有一大串 query parameter 感覺不爽的話，可以寫成這樣：

```kotlin
httpClient.get<EtaResponse>("https://rt.data.gov.hk/v1/transport/mtr/getSchedule.php") {
    parameter("line", "TML")
    parameter("sta", "TIS")
    parameter("lang", "TC")
}
```

這個 `httpClient.get` 是 suspending function，IDE 會在 suspending function 的行數顯示箭頭型的 [gutter icon](https://www.jetbrains.com/help/idea/settings-gutter-icons.html) 作提示。Suspending function 要有 Coroutine scope 包住才能用，以 `Activity` 為例，你不能在 `onCreate` 內 call 這一句，因為 `onCreate` 不是 suspending function，只有在 suspending function 內才能 call 另一個 suspending function，或者是在 coroutine scope 內。簡單來講，coroutine scope 就是用來連接 coroutine 和非 coroutine 的地方，coroutine scope 另一個用途是用來一併停止未完結的 suspending function，例如 `onDestroy` 時就可以 call coroutine scope 的 `cancel`。這個意念跟 RxJava 的 `CompositeDisposable` 類似。其實現在 AndroidX 的 library 已經幫我們在 `Activity`、`Fragment`、`ViewModel` 等地方為我們造好了對應其 lifecycle 的 coroutine scope，我們只需要直接調用就可以了，詳細的內容我們之後會示範。

## Dagger 設定

看過基本用法後我們要把 Ktor client 的設置放到 Dagger module 入面，這樣就可以經 Dagger 取得 Ktor client 的 instance。以下是大約的寫法：

```kotlin
@Module
@InstallIn(SingletonComponent::class)
interface DataModule {

    @BindsOptionalOf
    fun bindLogging(): HttpClientFeature<Logging.Config, Logging>

    companion object {
        @Provides
        @Singleton
        fun provideOkHttpClient(): OkHttpClient = OkHttpClient.Builder()
            .build()

        @Provides
        fun provideHttpClientEngine(okHttpClient: OkHttpClient): HttpClientEngine = OkHttp.create {
            preconfigured = okHttpClient
        }

        @Provides
        fun provideLogging(): HttpClientFeature<Logging.Config, Logging> = Logging.apply {
            prepare {
                logger = Logger.DEFAULT
                level = LogLevel.ALL
            }
        }

        @Provides
        @Singleton
        fun provideKtorHttpClient(
            engine: HttpClientEngine,
            logging: Optional<HttpClientFeature<Logging.Config, Logging>>,
        ): HttpClient = HttpClient(engine) {
            expectSuccess = true
            logging.ifPresent { install(it) }
            install(JsonFeature) {
                serializer = KotlinxSerializer(kotlinx.serialization.json.Json {
                    coerceInputValues = true
                    ignoreUnknownKeys = true
                })
            }
        }
    }
}
```

Dagger module 是用來向 Dagger 提供一些 Dagger 未能自動 instantiate 的 object。如果你有看過 Dagger 的教學，都是要在 class 的 constructor 加上 `@Inject` 然後在執行時那些寫在 constructor 的 parameter 就會自然地取得那些 object。這個自動找到 dependency 塞入去 constructor 給你用的動作就是 Dagger 幫你做的，但它那個自動功能只能 inject 其他在 constructor 加了 `@Inject` 的 class。但遇到其他 third party 的 class 例如 Ktor client 又或者是 Android SDK 入面的 `ConnectivityManager` 之類就要靠我們自己寫 Dagger module 來提示 Dagger 如何 instantiate 這些 class。`@Provides` 就是用來手動教 Dagger 如何 instantiate 那個 object。`@Provides` 的 function 名是不重要，因為 Dagger 只看 parameter type 和 return type，但習慣上都是會跟 annotation 名作前綴。`@Provides` function parameter 就是用來取得其他 dependency，例如 `provideKtorHttpClient` 需要用到 `HttpClientEngine` 和 `HttpClientFeature<Logging.Config, Logging>` 來 instantiate `HttpClient`。

你或許會留意到第二個 parameter 被 `Optional` 包住，這個 `Optional` 是 Java 8 的東西，就是表示 `HttpClientFeature<Logging.Config, Logging>` 可能會有亦可能會無，有點像 nullable 的意思。因為那個被加註 `@BindsOptionalOf` 的 function，Dagger 能看懂 `Optional`。如果你全個 app 都沒有 `@Provides` 那個 logging feature 它亦不會 build fail。我把 logging 包了 `Optional` 是因為在 testing 或在 release build 時我們就不用為 `HttpClient` 加 logging。

在 module 除了看到 `@Module` 之外，還有 `@InstallIn(SingletonComponent::class)`，這個 annotation 是 Hilt 的東西。Hilt 就是幫你訂好一個 Android app 會有那些 component。Component 主要作用是用來控制那些由 Dagger inject 的 dependency object 是不是在某範圍內重用還是每次要用到那個 dependency 都去 instantiate 一個新的。Hilt 的 `SingletonComponent` 就是跟 `Application` 共生死，它有對應的 scope 叫 `@Singleton`。上面 `OkHttpClient` 和 `HttpClient` 都加了 `@Singleton`，意思是如果那個 `SingletonComponent` 都是同一個 instance 的話，那我經那個 component 拿到的 `OkHttpClient` 和 `HttpClient` 都會是同一個 instance。正因為 `Application` 在執行時只會有一個 instance，所以 `OkHttpClient` 和 `HttpClient` 就變相成為平時我們理解的 singleton 一樣，只是不是用 object class 而是靠 Dagger 控制。而 `OkHttpClient` 和 `HttpClient` 要設成一個 app 共用同一個 instance 是因為 HTTP client 和 SQLite database connection 之類的東西建立成本比較高，所以不應每 call 一次 HTTP request 或 database query 都造一個全新的 connection。[`OkHttpClient`](https://square.github.io/okhttp/4.x/okhttp/okhttp3/-ok-http-client/) 亦有同樣的提示：

> OkHttp performs best when you create a single `OkHttpClient` instance and reuse it for all of your HTTP calls. This is because each client holds its own connection pool and thread pools. Reusing connections and threads reduces latency and saves memory. Conversely, creating a client for each request wastes resources on idle pools.

最後有一樣東西或許大家有留意到就是 `DataModule` 是 interface，入面只放 `@BindsOptionalOf` function（還有日後的 `@Binds` function），而內裏的 companion object 就放了 `@Provides` 的 function。這是因為按照 [Dagger 網站的說明](https://dagger.dev/dev-guide/) `@Provides` 最好是 static 而 `@Binds` 就因為 [Dagger 會生成對應的 code](https://dagger.dev/api/latest/dagger/Binds.html)，所以用 interface 就夠了。

> And for any module whose `@Provides` methods are all static, the implementation doesn't need an instance at all.

> `@Binds` methods are a drop-in replacement for `Provides` methods that simply return an injected parameter. Prefer `@Binds` because the generated implementation is likely to be more efficient.

> Using `@Binds` is the preferred way to define an alias because Dagger only needs the module at compile time, and can avoid class loading the module at runtime.

我們已經準備好 Ktor client，下一篇我們會寫處理 backend API call 的部分。

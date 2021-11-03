---
title: "2021 iThome 鐵人賽 Day 14：Flipper"
tags:
  - 2021 iThome 鐵人賽
  - Android
date: 2021-09-29T00:00:00+08:00
canonicalURL: https://ithelp.ithome.com.tw/articles/10274513
---

> 本篇文章是 [2021 iThome 鐵人賽](https://ithelp.ithome.com.tw/2021ironman)參賽題目「[寫一個列車抵站時間 Android App](https://ithelp.ithome.com.tw/users/20139666/ironman/4661)」的第 14 篇，你可到 iThome [查看原文](https://ithelp.ithome.com.tw/articles/10274513)。
>
> [文章目錄]({{< ref "2021-ithome-ironman" >}})

在繼續實作 domain layer 之前，我們會介紹一個方便日常開發的工具：[Flipper](https://fbflipper.com/)。

Android Studio 有個功能是查看 HTTP request 和 UI layout，但有時不太方便。如果是查看 HTTP request 的話，有些人會用 proxy server 來截取 HTTP request 和 response。但有個問題是裝置要先安裝 proxy server 的 root certificate，而且部分 app 或 SDK 會做 cert pinning，駁了 proxy server 就用不到那些 app 或 SDK（Google Places SDK 會有這個問題）。

Flipper 的前身是 [Stetho](http://facebook.github.io/stetho/)，或許大家以前有見過，就是 Facebook 借 [Chrome DevTools](https://developer.chrome.com/docs/devtools/) 介面來提供 HTTP traffic、UI layout、Shared Preferences、SQLite database 查閱功能的那個 library。但因為用 Chrome DevTools 的界面做 UI，功能就會受到 DevTools 的限制，不能提供超越 DevTools 界面的功能。還有是 iOS 又不能用，app 結束後那個 DevTools 視窗就要作廢不能重用。所以就促成 Facebook 開發 Flipper（前稱 Sonar）。Flipper 是一個用 Electron 做的 desktop app 來做介面，並提供 Android 和 iOS SDK 把 app 的內容交予 Flipper desktop app。為了方便我們做 UI 時能看清楚 HTTP request，我們現在要做的是把 SDK 加到 app 入面。

首先是 app module 的 *build.gradle* 加上以下的 dependency：

```groovy
debugImplementation "com.facebook.flipper:flipper:$flipperVersion"
debugImplementation "com.facebook.soloader:soloader:$soloaderVersion"
debugImplementation "com.facebook.flipper:flipper-network-plugin:$flipperVersion"
```

留意 Flipper Android SDK 在 Flipper 網站和 GitHub 的最新版本未必能在 Maven Central 找到，所以最好還是先[檢查 Maven Central 那邊最新版本是甚麼](https://mvnrepository.com/artifact/com.facebook.flipper/flipper)。

由於我們不想在 app 日後上架時都夾附 Flipper，我們就借用 [build type](https://developer.android.com/studio/build/build-variants) 來控制：`debug` 才能用 Flipper；`release` 就不要有 Flipper 的 dependency。所以我們這次用 `debugImplementation` 而不是 `implementation`。

接下來就是按照 [Flipper 網站的指示](https://fbflipper.com/docs/getting-started/android-native#application-setup)在 `Application` class 的 `onCreate` 加上 Flipper 初始化的 code。不過我們應該還未有自己的 `Application` class，現在就先建立一個叫 `EtaDemoApp` 的 `Application` subclass。

```kotlin
@HiltAndroidApp
class EtaDemoApp : Application() {

    @Inject
    lateinit var flipperHelper: FlipperHelper

    override fun onCreate() {
        super.onCreate()
        flipperHelper.init()
    }
}
```

`@HiltAndroidApp` 是 Dagger Hilt 的 annotation，它是加在 `Application` 的 subclass。而 `@Inject     lateinit var flipperHelper` 那句是叫 Dagger inject 一個 `FlipperHelper` 好讓我們能在 `onCreate` 能用到它。這個 `@Inject` 的用法跟之前放在 constructor 時的用法不同，原因是 Android 的主要 class（例如 `Application` 和「四大組件」之稱的 `Activity`、 `Service`、`ContentProvider`、`BroadcastReceiver`）的 constructor 都是由 Android 系統去 call，Dagger 或其他 dependency injection library 就不能用 constructor injection 的方法把 dependency 提供給那些 class。取而代之是依靠系統 call 那些 `onCreate` 的 lifecycle callback 時你才有機會叫 dependency injection library 幫你拿到 dependency。所以我們改用 field injection 來取得 `FlipperHelper`。但你或許會覺得很神奇我們都沒有在 `onCreate` call 到 Dagger 的 function 都能拿到 dependency。這是因為我們加了 `@HiltAndroidApp` 在 `EtaDemoApp`。Dagger Hilt 的 Gradle plugin 在 compile app 時會幫我們重寫一個新的 `EtaDemoApp`，裏面的 `onCreate` 就會幫我們 call 了 Dagger 來做 field injection。意念上就像中國 Android 開發教學文章時常提及的 aspect-oriented programming (AOP)。（就是經常被拿來幫全個 app 的 `OnClickListener` 加插 event tracking code 那些教學）如果有用過未有 Dagger Hilt 之前的 Dagger，你會發現我們無寫過甚麼 `appComponent.inject(this)` 之類的東西。因為 Dagger Hilt 替我們做了這些東西，那我們就可以專注寫其他的 code，亦都令 Dagger 變得易用。

開了 `EtaDemoApp` 之後，同時亦要在 *AndroidManifest.xml* 的 `application` tag 加上 `name` attribute 指明要用 `EtaDemoApp`。

```xml
<application
    android:name=".EtaDemoApp"
    android:allowBackup="true"
    android:icon="@mipmap/ic_launcher"
    android:label="@string/app_name"
    android:roundIcon="@mipmap/ic_launcher_round"
    android:supportsRtl="true"
    android:theme="@style/Theme.ETADemo">
    <activity
        android:name=".MainActivity"
        android:exported="true">
        <intent-filter>
            <action android:name="android.intent.action.MAIN" />

            <category android:name="android.intent.category.LAUNCHER" />
        </intent-filter>
    </activity>
</application>
```

你會看到 `EtaDemoApp` 入面用了一個 `FlipperHelper` ，這個 class 是我們現在會寫的 class，我們就是把那些調用 `AndroidFlipperClient` 的東西全部塞進去而不是直接把那些 code 直接寫進去`EtaDemoApp` 內。原因是我們剛在 *build.gradle* 設定只有在 `debug` build type 時才有 Flipper 的 dependency。如果直接把那些調用 `AndroidFlipperClient` code 放進去 `EtaDemoApp` 的話我們 build `release` build type 的 app 就會報錯。解決方法是我們會在 `debug` 和 `release` build type 的 *src* 目錄各造一個 `FlipperHelper`，`debug` 那個 helper 就如 Flipper 網站示範的寫法在 `FlipperHelper.init` call 一堆 `AndroidFlipperClient` 的 method 來初始化 Flipper；但在 `release` 那個 helper 就放個空白的 `init` 的 function 來令 compiler 成功 build 到 app。

## `Debug` 部分

以下是 debug build type 的 `FlipperHelper`，它是放在 *app/src/debug/java/net/swiftzer/etademo/flipper/FlipperHelper.kt*。特別強調所放的位置是因為它要放在 `debug` build type 的 source directory。

```kotlin
class FlipperHelper @Inject constructor(
    @ApplicationContext private val context: Context,
    private val inspectorFlipperPlugin: InspectorFlipperPlugin,
    private val crashReporterPlugin: CrashReporterPlugin,
    private val databasesFlipperPlugin: DatabasesFlipperPlugin,
    private val sharedPreferencesFlipperPlugin: SharedPreferencesFlipperPlugin,
    private val networkFlipperPlugin: NetworkFlipperPlugin,
) {
    fun init() {
        SoLoader.init(context, false)
        if (!FlipperUtils.shouldEnableFlipper(context)) return
        val client: FlipperClient = AndroidFlipperClient.getInstance(context)
        client.addPlugin(inspectorFlipperPlugin)
        client.addPlugin(crashReporterPlugin)
        client.addPlugin(databasesFlipperPlugin)
        client.addPlugin(sharedPreferencesFlipperPlugin)
        client.addPlugin(networkFlipperPlugin)
        client.start()
    }
}
```

`init` 入面的東西基本上就是 Flipper 網站所寫的內容，只是那些 plugin 換成經 Dagger 在 constructor inject 拿到的。留意 constructor 的 `context` 加了 `@ApplicationContext`，這個 annotation 是 Dagger Hilt 提供的。因為 context 有分 `Activity` 和 `Application`，兩個的 lifecycle 是不同的，所以你要指明你要的是那一個 `context`。這種用來告訴 Dagger 要 inject 相同 abstract class/interface 之下的那一款 subclass object 叫做 qualifier，我們稍後會再介紹。

由於我們打算用 Dagger inject 那些 Flipper plugin，所以我們要另外準備一個 Dagger module：`FlipperDebugModule`。叫它做 `FlipperDebugModule` 是因為這個 module 是放在 `debug` build type 的 source directory，它會放在 *app/src/debug/java/net/swiftzer/etademo/flipper/FlipperDebugModule.kt* 。

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object FlipperDebugModule {

    @Provides
    fun provideInspectorFlipperPlugin(@ApplicationContext context: Context): InspectorFlipperPlugin =
        InspectorFlipperPlugin(context, DescriptorMapping.withDefaults())

    @Provides
    fun provideCrashReporterPlugin(): CrashReporterPlugin = CrashReporterPlugin.getInstance()

    @Provides
    fun provideDatabasesFlipperPlugin(@ApplicationContext context: Context): DatabasesFlipperPlugin =
        DatabasesFlipperPlugin(context)

    @Provides
    fun provideSharedPreferencesFlipperPlugin(@ApplicationContext context: Context): SharedPreferencesFlipperPlugin =
        SharedPreferencesFlipperPlugin(context)

    @Provides
    @Singleton
    fun provideNetworkFlipperPlugin(): NetworkFlipperPlugin = NetworkFlipperPlugin()

    @Provides
    @FlipperInterceptor
    fun provideFlipperInterceptor(networkFlipperPlugin: NetworkFlipperPlugin): Interceptor =
        FlipperOkhttpInterceptor(networkFlipperPlugin)
}
```

Flipper plugin 這個部分或許不用 Dagger 來 inject 亦可以，但因為我們需要為 Ktor client 所用的 OkHttp client 加插 `FlipperOkhttpInterceptor` 才能在 Flipper desktop app 看到 HTTP traffic，所以還是最少需要把 `NetworkFlipperPlugin` 和 `FlipperOkhttpInterceptor` 納入 Dagger dependency graph 內。這次我們用 `object` 來做 `FlipperDebugModule` 是因為它只會放 `@Provides` function，沒有 abstract function，所以全部 function 都是 static 效能較佳。 

留意 `provideFlipperInterceptor` return type 是 OkHttp 的 `Interceptor`，這是因為我們不想把 Flipper 的 class 外露給其他地方知道，這樣 `release` build type 就能順利地 compile。另一個要留意是我們加了 `@FlipperInterceptor`，這個是我們自製的 Dagger qualifier annotation。雖然這個 app 應該不會再有其他 OkHttp interceptor，但在現實中的 app 有時會有多於一個 interceptor，所以順帶示範了 qualifier 的用法。`FlipperInterceptor` 放在 data package 內，因為跟網路相關。這個就不用分 build type 放，放在 main 的 source directory 就可以了。

```kotlin
@Qualifier
@MustBeDocumented
@Retention(AnnotationRetention.RUNTIME)
annotation class FlipperInterceptor
```

如果日後再有其他 OkHttp interceptor，那就照樣加一個新的 qualifier annotation 標記它。

另一樣要留意是 `provideNetworkFlipperPlugin` 我們加了 `@Singleton`。這是因為沒有加 `@Singleton` 的話每次一有地方要用到 `NetworkFlipperPlugin` Dagger 就會 call 你寫的 `provideNetworkFlipperPlugin` 來 instantiate 一個全新的 `NetworkFlipperPlugin`。但這樣的話你就不可能在 Flipper desktop app 看到 HTTP traffic。 **這是因為 `FlipperHelper.init` 所用的 `NetworkFlipperPlugin` 和加插在 OkHttp 的 `FlipperOkhttpInterceptor` 所用的 `NetworkFlipperPlugin` 是兩個完全不同的 instance。** 為了令兩者所用的 `NetworkFlipperPlugin` 都是同一個 instance，所以就加了 `@Singleton` 這個 scope。在 Dagger Hilt 的定義下，`@Singleton` 的範圍就是 `Application` class 的存活範圍。由於在 Android 下 `Application` class 只有一個 instance，所以可以理解為整個 app 最多只有一個 `NetworkFlipperPlugin` 的 instance。

回到 `DataModule`，我們要在 `provideOkHttpClient` 加插 `FlipperOkhttpInterceptor`。由於在 `release` build type 時整個 app 都不會出現 `provideFlipperInterceptor`，所以我們會好像之前 logger 般把它包成 `Optional`。

```kotlin
@Provides
@Singleton
fun provideOkHttpClient(@FlipperInterceptor flipperInterceptor: Optional<Interceptor>): OkHttpClient {
    val builder = OkHttpClient.Builder()
    flipperInterceptor.ifPresent { builder.addNetworkInterceptor(it) }
    return builder.build()
}
```

由於加了 `Optional` ，我們亦都要為它補上 `@BindsOptionalOf`。

```kotlin
@Module
@InstallIn(SingletonComponent::class)
interface DataModule {
    @BindsOptionalOf
    @FlipperInterceptor
    fun bindFlipperInterceptor(): Interceptor

    // ...
}
```

留意我們每次用到 `Interceptor` 都要加上 `@FlipperInterceptor`，否則 Dagger 不會在 dependency graph 找到這個 `Interceptor`。

最後就是為 `debug` build type 加一個 `activity`，它是放在 *app/src/debug/AndroidManifest.xml*。

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="net.swiftzer.etademo">

    <application>
        <activity
            android:name="com.facebook.flipper.android.diagnostics.FlipperDiagnosticActivity"
            android:exported="true" />
    </application>
</manifest>
```

## `Release` 部分

由於 Flipper 只會在 `debug` build type 才會用到，換到 `release` 的部分我們只需要做個空白的 `FlipperHelper` 滿足 compiler 的要求就可以了。這個 `FlipperHelper` 要放在 *app/src/release/java/net/swiftzer/etademo/flipper/FlipperHelper.kt*。

```kotlin
class FlipperHelper @Inject constructor() {
    fun init() {
        // No-op
    }
}
```

由於這個 `FlipperHelper` 沒有在 constructor 用到那些 Flipper plugin，所以我們就不用寫一個 `FlipperReleaseModule` 之類的東西，就是這麼簡單。

---

你或許會問為甚麼我們不乾脆把 `EtaDemoApp` 分 `debug` `release` 兩個版本而要另外開一個 `FlipperHelper` 分兩邊放。這是因為 application class 通常都會有其他東西，為了一個 Flipper 而要維護兩個（或更多個）application class 是費時失事。如果 build variant 再增加下去的話，我會建議另開兩個 Gradle module 放有 Flipper 和無 Flipper 版的 `FlipperHelper` ，然後再 `xxxImplementation project(':flipper')` 或 `project(':flipper-noop')` 這樣。其實 Flipper 本身都有提供 no-op 版 artifact，但不是所有 Flipper plugin 都有提供 no-op 版 artifact，所以還在在本篇示範了如何自行做 no-op。

## 小結

我們現在已經加了 Flipper，它的 network 功能我們之後會用到（我們要看它有沒有定時自動更新班次）。為甚麼要用一篇文章寫 Flipper 呢？這是因為以前工作時經常被 backend 同事問到 Android app 如何 call 那個 endpoint，最直觀的方法就是用 proxy 或者 Flipper 這類工具先看看 network traffic 然後才在 app 的 source code 找 call 那個 endpoint 的位置。這樣我就不用把 code 由頭看到尾，還有是 backend 同事能自己直接試，不用再走來問我這頁 call 了甚麼 endpoint、payload 是甚麼之類的問題。

當然，如果要做到改 response、延遲 request/response 這些功能的話，還是需要用到 proxy server。我平常用開的 proxy server 是 [Whistle](https://github.com/avwo/whistle)，proxy server 還有其他選擇，例如 [Charles](https://www.charlesproxy.com/)、[Proxyman](https://proxyman.io/)、[Fiddler](https://www.telerik.com/fiddler) 等等。

另外亦借安裝 Flipper 介紹了 Dagger Hilt 比 Dagger 簡化了甚麼地方，Dagger 的 scope 和 qualifier 用法，還有是 build type 的用法。

這次示範的完整 code 可以在 [GitHub repo](https://github.com/ericksli/eta-demo/tree/main/app/src/debug/java/net/swiftzer/etademo/flipper) 找到。

## 參考

- [MAD Skills series: Hilt under the hood](https://medium.com/androiddevelopers/mad-skills-series-hilt-under-the-hood-9d89ee227059)
- [从 Dagger 到 Hilt，谷歌为何执着于让我们用依赖注入？](https://www.youtube.com/watch?v=iCx5fJk6KQc)
- [Configure build variants](https://developer.android.com/studio/build/build-variants)

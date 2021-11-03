---
title: 2021 iThome 鐵人賽「寫一個列車抵站時間 Android App」文章目錄
date: 2021-10-16T00:00:00+08:00
---

以下是我參加 [2021 iThome 鐵人賽](https://ithelp.ithome.com.tw/2021ironman) 所寫的系列文章「[寫一個列車抵站時間 Android App](https://ithelp.ithome.com.tw/users/20139666/ironman/4661)」的目錄。原文在 iThome 首次發表，在完賽後轉貼到這裏。

文章示範的 code 可以在 [GitHub repo](https://github.com/ericksli/eta-demo) 找到。

---

1. [Intro]({{< ref "2021-ithome-ironman-01-intro" >}})<br>
簡單的開場白，介紹將會提及的內容。
2. [Architecture]({{< ref "2021-ithome-ironman-02-architecture" >}})<br>
Architecture Components (MVVM)、Modularization 和這次示範 app 所用的 architecture。
3. [Endpoint]({{< ref "2021-ithome-ironman-03-endpoint" >}})<br>
示範 app 會用到的 API endpoint 和 API response 對應的 data class。
4. [Deserialization]({{< ref "2021-ithome-ironman-04-deserialization" >}})<br>
比較 Gson、Moshi、Kotlin Serialization，並為上篇準備的 data class 加上 Kotlin Serialization 的 annotation。
5. [Dependency injection]({{< ref "2021-ithome-ironman-05-dependency-injection" >}})<br>
為 project 加入 Dagger Hilt。
6. [HTTP Client]({{< ref "2021-ithome-ironman-06-http-client" >}})<br>
Ktor client 簡介、基本用法及把 Ktor client 加進 Dagger dependency graph。當中提及 `@Inject`、`@InstallIn`、`@Singleton` 及 `@BindsOptionalOf` 用法。
7. [Data layer implementation (1)]({{< ref "2021-ithome-ironman-07-data-layer-implementation-1" >}})<br>
車站及路綫 enum、domain layer repository interface、實作接駁 API endpoint、Dagger `@Binds` 用法及 Ktor client 跟 Retrofit 比較。
8. [Data layer implementation (2)]({{< ref "2021-ithome-ironman-08-data-layer-implementation-2" >}})<br>
API response data class 轉換到 domain layer data class 的 mapper。
9. [Date & time]({{< ref "2021-ithome-ironman-09-date-time" >}})<br>
討論為甚麼要用 Java Time 而不是用 JDK 傳統的 `Calendar`、`Date` 的原因、為 project 加入 desugering library 及介紹 Time Zone Database。
10. [Data layer testing (1)]({{< ref "2021-ithome-ironman-10-data-layer-testing-1" >}})<br>
JUnit 4 test、MockK 及 Strikt 基本用法。
11. [Data layer testing (2)]({{< ref "2021-ithome-ironman-11-data-layer-testing-2" >}})<br>
用 Strikt 為 object 每個 property 做 assertion。
12. [Data layer testing (3)]({{< ref "2021-ithome-ironman-12-data-layer-testing-3" >}})<br>
Ktor client 搭配 logback-android 在 unit test 時出現錯誤的解決方法、運用 Ktor mock engine 模擬 server response。
13. [Data layer testing (4)]({{< ref "2021-ithome-ironman-13-data-layer-testing-4" >}})<br>
運用上篇的 Ktor mock engine 測試觸發 HTTP request 的 code 和經過 Kotlin Serialization deserialize 後的 response object 是否合符預期，另加 Strikt 自定義的 assertion。
14. [Flipper]({{< ref "2021-ithome-ironman-14-flipper" >}})<br>
安裝 Flipper（只在 debug build type 才安裝）、Dagger Hilt 的 `@HiltAndroidApp` 及 `@ApplicationContext` 用法、Dagger `@Qualifier` 及 `@BindsOptionalOf` 用法、OkHttp client 加入 interceptor。
15. [Domain layer implementation]({{< ref "2021-ithome-ironman-15-domain-layer-implementation" >}})<br>
實作 use case。
16. [Domain layer testing]({{< ref "2021-ithome-ironman-16-domain-layer-testing" >}})<br>
Use case 的 unit test。
17. [Navigation (1)]({{< ref "2021-ithome-ironman-17-navigation-1" >}})<br>
安裝 Navigation component，準備將會用到的 `Fragment` 和 layout XML 檔案、簡單介紹 Dagger Hilt 的 `@AndroidEntryPoint` 及在 `Fragment` 使用 data binding 時要注意的地方。
18. [Navigation (2)]({{< ref "2021-ithome-ironman-18-navigation-2" >}})<br>
定義 Navigation graph 並設定轉頁的參數、設定 `MainActivity` 以顯示 navigation graph 的頁面及使用轉頁參數時要注意的地方。
19. [Station list screen (1)]({{< ref "2021-ithome-ironman-19-station-list-screen-1" >}})<br>
實作車站列表頁介面，使用了 `RecyclerView` 的 `ListAdapter`、`DiffUtil`、data binding，並示範了 Dagger Hilt 的 `@ActivityScoped` 用法。
20. [Station list screen (2)]({{< ref "2021-ithome-ironman-20-station-list-screen-2" >}})<br>
實作車站列表頁的 `ViewModel`，示範了 Dagger Hilt 的 `@HiltViewModel`、Kotlin Flow 的 `Channel`、`StateFlow`/`MutableStateFlow`、`combine`、`repeatOnLifecycle` 用法和 Navigation component 防止用戶快速連按轉頁按鈕導致 crash 的方法。
21. [ETA screen (1)]({{< ref "2021-ithome-ironman-21-eta-screen-1" >}})<br>
介紹抵站時間頁的流程，實作頁面基本的 layout XML、`RecyclerView`、`ViewModel` 及 `onBackPressedDispatcher`，另外亦示範了 Kotlin `Sequence`。
22. [Whistle proxy]({{< ref "2021-ithome-ironman-22-whistle-proxy" >}})<br>
安裝 Whistle proxy、Whistle proxy 基本用法、在 Android 裝置安裝根證書、設定 network security configuration 及介紹 Proxy Toggle。
23. [ETA screen (2)]({{< ref "2021-ithome-ironman-23-eta-screen-2" >}})<br>
`SavedStateHandle` 用法及以 Kotlin Flow 控制各式錯誤介面顯示。
24. [ETA screen (3)]({{< ref "2021-ithome-ironman-24-eta-screen-3" >}})<br>
自動更新班次，介紹 Kotlin Coroutine 的 `CoroutineScope`、`Job`、`delay` 用法並使用 Java Time 的 `Clock` 取得當前時間。
25. [ETA screen (4)]({{< ref "2021-ithome-ironman-25-eta-screen-4" >}})<br>
顯示錯誤 banner，介紹 Kotlin Coroutine 的 `scan` (`runningFold`) 用法，順帶講解 Kotlin Collection 的 `fold` 和 `reduce`。
26. [Station list screen testing]({{< ref "2021-ithome-ironman-26-station-list-screen-testing" >}})<br>
安裝 Robolectric，為引用了 Android `Context` 的 class 寫 unit test；針對 Flipper 在 unit test 時報錯的解決方法；替換 `Dispatchers.Main` 及運用 Turbine 測試 Flow。
27. [ETA screen testing (1)]({{< ref "2021-ithome-ironman-27-eta-screen-testing-1" >}})<br>
示範 JUnit 4 parameterized test；介紹 cold flow (`StateFlow`) 跟 hot flow (`SharedFlow`) 的分別。
28. [ETA screen testing (2)]({{< ref "2021-ithome-ironman-28-eta-screen-testing-2" >}})<br>
運用 ThreeTen-Extra 提供的 `MutableClock` 及 Kotlin Coroutine 的 `DelayController` 改變時間並測試自動更新班次的邏輯。
29. [Leftover topics]({{< ref "2021-ithome-ironman-29-leftover-topics" >}})<br>
Two-way data binding、`RecyclerView` 局部更新、unidirectional data flow、instrumentation test 及 Coroutine dispatcher。
30. [Wrapping up]({{< ref "2021-ithome-ironman-30-wrapping-up" >}})<br>
參賽總結。

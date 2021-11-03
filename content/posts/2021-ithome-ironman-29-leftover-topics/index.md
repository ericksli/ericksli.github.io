---
title: "2021 iThome 鐵人賽 Day 29：Leftover topics"
tags:
  - 2021 iThome 鐵人賽
  - Android
date: 2021-10-14T00:00:00+08:00
canonicalURL: https://ithelp.ithome.com.tw/articles/10281516
---

> 本篇文章是 [2021 iThome 鐵人賽](https://ithelp.ithome.com.tw/2021ironman)參賽題目「[寫一個列車抵站時間 Android App](https://ithelp.ithome.com.tw/users/20139666/ironman/4661)」的第 29 篇，你可到 iThome [查看原文](https://ithelp.ithome.com.tw/articles/10281516)。
>
> [文章目錄]({{< ref "2021-ithome-ironman" >}})

我們終於來到第廿九篇，我們這次討論的題目都是之前討論過的東西的延伸。因為篇幅和時間有限就只好把它們合併成一篇。

## Two-way data binding

我們在示範 app 一直都是在用 one-way data binding，只要在 layout XML 加上 `@{ ... }` 就能用到 `LiveData` 或 `StateFlow` 的值，並且能在 `LiveData` 或 `StateFlow` 的值改動時自動更新 UI（要設定好 `LifecycleOwner`）。Two-way data binding 適合在一些用戶輸入的 UI 組件使用，例如 `TextEdit`、`CheckBox` 之類。寫法會是這樣：

```xml
<EditText
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:text="@={viewModel.query}" />
```

`query` 可以是 `MutableLiveData` 或 `MutableStateFlow`。留意是要用 mutable 的，因為當用戶改變 `EditText` 的值時會直接把最新的值寫入去 `MutableLiveData` 或 `MutableStateFlow`。然後你可以在 `ViewModel` 把那個 `MutableLiveData` 或 `MutableStateFlow` 用 `map` 之類的 operator 轉化成其他動作（例如當用戶在輸入文字後就 call API 搜尋內容）。

其實 data binding 的原理是那些 `@{ ... }` 和 `@={ ... }` 語法會在 compile 時轉成 binding 的 class，入面還是會 call 那些 view 的 getter、setter、listener，只是不用我們自己寫而已。這樣就可以少寫一部分 code。

但 data binding 還是有 Android view 既有的問題。以 `CheckBox` 為例，你可以設定 `OnCheckedChangeListener` 來得知用戶改變剔選狀態。如果以 code 的形式改變 `CheckBox` 的狀態可以用 `setChecked` 這個 method。但 call 完 `setChecked` 後之前設定好的 `OnCheckedChangeListener` 都會收到 callback，變相很難分辨究竟那個 `onCheckedChanged` callback 是由用戶輸入行為觸發還是由 code call 了 setter 觸發。在 StackOverflow 有人提議可以改用 `OnClickListener` 取代 `OnCheckedChangeListener`，這樣就肯定 callback 是因為用戶輸入而觸發。另一個做法是在 call setter 前先把 `CheckBox` 的 `OnCheckedChangeListener` 設定 `null`，當 call 完 setter 後就把 `OnCheckedChangeListener` 還原。第一個做法不合乎語義，第二個做法又太古怪。但我們用 two-way data binding 好像不用理這個問題？不是，只是基本的 view 例如 `TextEdit` 和 `CheckBox` data binding 本身已附送 binding adapter，它背後的實作已經有處理這個問題。而 two-way data binding 的文檔都有特別提及這種 [setter 和 listener 之間做成的無限循環問題](https://developer.android.com/topic/libraries/data-binding/two-way#infinite-loops)。它的解決方法是在 binding adapter call setter 前檢查當前 view 的值是不是和要設定的值一樣，如果一樣就不要再 call setter，以免觸發 listener 導致 data binding 發覺 view 的值有改變之後把背後的 `MutableLiveData` 或 `MutableStateFlow` 值改成 view 的值，因而再 call 多次 view 的 setter。

## `RecyclerView` 局部更新

之前示範了用 `DiffUtil.ItemCallback` 做自動計算新舊內容比對然後更新 `RecyclerView` 的 list item。但有時把 list item 的所有 view 都做一次 bind 的話可能會導致出現的效果。以 Instagram 的 news feed 為例，如果要更新 like 數的話，像我們之前的寫法會在資料變動後把整個 list item 重新 bind 過，導致影片重新載入。但我們期望看到的是影片是繼續播放不中斷而 like 的數字轉了。要做到這樣的效果我們可以 override `DiffUtil.ItemCallback` 的 `getChangePayload`。這個 method 會在 `areItemsTheSame` return `true` 並且 `areContentsTheSame` return `false` 時執行。`getChangePayload` 預設是 return `null`，你需要在 method 內找出兩個 object 之間的差異，然後把結果 return。它其實沒有限定 return type，你可以 return 一個 `Set` 內裏有表示不同 property 的 enum 表示改動過的 property。亦有其他人用 `Bundle` 來放兩個 object 之間的差異。

改完 `DiffUtil.ItemCallback` 後就要更改 `ListAdapter`。之前我們都是 override `onBindViewHolder(holder: RecyclerView.ViewHolder, position: Int)`。但我們現在 override 了 `getChangePayload` 後就要在 adapter override `onBindViewHolder(holder: RecyclerView.ViewHolder, position: Int, payloads: MutableList<Any>)`。

```kotlin
override fun onBindViewHolder(
    holder: RecyclerView.ViewHolder,
    position: Int,
    payloads: MutableList<Any>,
) {
    if (payloads.isEmpty()) {
        // 完全不同，用回普通的 onBindViewHolder 做 bind
        super.onBindViewHolder(holder, position, payloads)
        return
    }
    // payload 第一個元素會有 getChangePayload return 出來的 object
    val diff = (payloads.first() as? Set<DisplayContent>).orEmpty()
    val item = getItem(position) ?: return
    // 之前已經 bind 過，按照 diff 有的 property 去 call 對應的 view setter
}
```

做了局部更新後，我們應該要把 list item 由 data binding 轉為 view binding。因為用 data binding 又再直接從 code call view setter 覆寫 data binding 做的東西會很怪。

## 由 data binding 到 unidirectional data flow 再到 Compose

當習慣用 `LiveData` 和 `StateFlow` 後很自然地會把整頁的狀態以 `LiveData` 和 `Flow` 的形式放到 `ViewModel` 內。之後就是在 `ViewModel` 外露一些 method 回應用戶輸入。例如當用戶按下按鈕時會觸發 `ViewModel` 的一個 method。然後這個 method 會做一些東西（例如 call 了 backend 並等待 response）並且改變被 data binding 觀察的 `LiveData` 和 `StateFlow` 的值從而改變 UI。我們的示範 app 就是用這種作法，這樣就做到 unidirectional data flow 的效果。UI 只需要按照最新的狀態來顯示就可以了。但問題是 Android 傳統的 view 本身的設計就不是讓人這樣做的。如果是 `TextView` 你不斷 call `setText("hello")` 從用戶肉眼看起來就只是一個「hello」而已，沒有閃動之類的事發生。但到了 `EditText` call setter 那時如果本身輸入法有拼寫檢查的候選字的話 call 了 `setText` 就會清空輸入法候選字。所以如果刻意令 `EditText` 的狀態和 `ViewModel` 完全同步（callback 一收到改動就要 call `EditText.setText` 來統一兩邊狀態）的話用戶就感覺到有異樣。再到極端例子 `WebView`，它並沒有外露完整的 getter 和 setter 讓你能完全掌控 `WebView` 內部狀態。如果要保留和還原狀態就只能用 `onSaveInstanceState` 和 `onRestoreInstanceState`，但那個 `Bundle` 不是讓你去讀而是讓 `WebView` 自己之後讀的。這種設計很容易會因為 `ViewModel` 的狀態跟 view 的狀態不完全同步而造成 bug。有些用 Android 傳統 view 做 unidirectional data flow 的示範 project 沒有特別做這類的示範（可能和我們的示範 app 一樣沒有用戶輸入的部分），讓人誤以為 unidirectional data flow 是容易做到，但到了實作時就發現問題多多。像以前那些 MVP 例子也是，很多都沒有考慮到 configuration change 的問題，只要一旋轉裝置之前的狀態就會消失。當然現在有 Architecture Component 和 `SavedStateHandle` 情況好了不少。但我想講的是傳統的 Android view 是很難完全做到所有狀態都交到 `ViewModel` 保留和控制，除非轉用 Compose 來寫 UI。

Compose 就是聲明式的 UI，我們不能像以前可以 call view 的 getter 取得它的狀態。相反，大部分的 Compose UI 都是沒有自己的狀態，狀態都是由外部（即是 `ViewModel`）傳進來，自己保留的狀態都是一些 `ViewModel` 不太重視的東西（例如動畫相關的值）。

以前我們做下面這類的 Chip 界面很多時都是在 layout XML 開一個空白的 `ViewGroup` 然後用 code 建構每一個 `Chip` 再塞進去 `ViewGroup`。然後當這些 Chip 更新時就把整個 `ViewGroup` 的 child view 清除再重新建構一堆新的 `Chip` 再塞進去。

{{< figure src="chip.png" title="Material Design 的 Chip" >}}

圖片擷自 [Material Design 網站](https://material.io/components/chips)

這樣做就不用花時間比對新舊 data 之間的差異而且不用處理那些 view 可以重用的問題。我們的示範 app 就是用了 `RecyclerView` 和 `DiffUtil.ItemCallback` 來做這些東西。但上面的例子大家應該都不會用 `RecyclerView` 來做。其實清空 `ViewGroup` 再做一堆新 view 有點像 Compose 的用法，但 Compose 能夠自動幫我們做新舊對比，而且不限於 `RecyclerView` 這類 UI。還有是用 Compose 的話就不用做清空的動作，因為它的寫法是 `ViewModel` 提供當前該頁全部的狀態，不單止是因應剛才用戶按了某個 chip 而只提供那個 chip 的資料。在我們的示範 app 我們都是循這個方向去寫，如果要轉做 Compose 的話相信不會有大問題。

## Instrumentation test

我們一直寫的測試都是在電腦上執行，而不是在 Android 裝置。我們亦用了 [Robolectric](http://robolectric.org/) 來補足 Android SDK 獨家的 class。如果是要拿到 Android 上面執行（不論是實機還是模擬器）的測試是叫做 instrumentation test。一般可以再細分兩類：UI test 和非 UI test。UI test 應該很容易明白，就是跟 UI 有關的，例如檢查界面顯示的內容、轉頁之類是不是合符預期。而非 UI 但又要放在 Android 執行的測試的例子有即場建立 in-memory SQLite database 檢查 SQL statement、foreign key constraint 是否正確。這些東西如果靠 mock 是不能真正檢查到這些東西，就算寫到出來亦會像謄文般把實作的 code 在 test case 中抄一遍（因為太多東西要 mock）。

如果是 UI test 的話可以參考我之前參加「[Android # Best practice Challenge for MVVM x RecyclerView](https://medium.com/gdg-taipei/android-best-practice-challenge-for-mvvm-x-recyclerview-acd9e9ad0dae)」而做的 [GitHub repo](https://github.com/ericksli/DiffUtilRV)。UI test 基本上都是用 [Espresso](https://developer.android.com/training/testing/espresso) 來控制 UI 及檢查 UI 的元素，它背後是用 [Hamcrest](http://hamcrest.org/) 這套 assertion 框架，跟我們之前示範用的 [Strikt](https://strikt.io/) 有點似。不過 Espresso 難在它報錯時只會提供一大串 view hierarchy 給你看，但你又看不明白。所以最好還是寫了一小點就執行來看看是不是沒有問題，這樣就容易除錯。

如果已經用 Compose 的話就不是用 Espresso，要用 [Compose 專用的測試 artifact](https://developer.android.com/jetpack/compose/testing)。其實 Espresso 和 Compose 的 UI test 寫法都是大同小異，基本上都是靠標記一些記號（例如 view ID）然後在 view tree 找到那些元素，之後檢查它的 attribute 或者是做一些用戶跟 app 的互動（例如按下按鈕）。為 view 加上測試的記號是 UI test 常用手法，例如 `ImageView` 用載入圖片 library 載入圖片的話可以順帶幫它 `setTag` 標記圖片網址，然後在 UI test 核對圖片網址。就算是 Compose 都是叫人用 semantics property 來做 UI test。

## Coroutine

我們在示範 app 已經用了 Coroutine 和 Flow。但其實我們一直都沒有主動切換 thread。除了 Ktor client 那部分之外剩下的東西（包括那些 mapper）都是在 UI thread 執行。Coroutine 的運作方式就是在一條 thread 內跑到調用 suspending function 的指令時切換到不同的 Coroutine 來營造出幾個 Coroutine 在同時執行的效果。我們之在做自動更新時用的是 Coroutine 提供的 `delay` 而不是 Java 的 `Thread.sleep` 是因為 `Thread.sleep` 真是會把執行 Coroutine 的 thread（即是 UI thread）卡死，但 `delay` 就是會切換執行另一個 Coroutine，直至時間到為止。亦因為 Coroutine 不會在 suspending function 內每一句 statement 之間幫我們檢查現在 Coroutine 是不是已被取消，所以 Kotlin 的文檔有提到如果是寫循環語句的話最好在每次迭代時都檢查一次 [`isActive`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/is-active.html) 以確保我們在 Coroutine 取消後就停止執行。

Ktor client 和 OkHttp client 已經為我們處理了 thread 的事宜，否則我們一進去班次頁就出現 `NetworkOnMainThreadException`。如果真的想把部分流程放在另一條 thread 執行的話，就要用到 [coroutine dispatcher](https://kotlinlang.org/docs/coroutine-context-and-dispatchers.html)。這跟 RxJava 的 [scheduler](https://www.baeldung.com/rxjava-schedulers) 和 Java 的 thread pool 有點似。常用的 dispatcher 有：

- `Dispatchers.Main` 在 main thread 上行（視乎你用了甚麼 Coroutine 的 artifact，Android 的話是 UI thread）
- `Dispatchers.Default` 按裝置 CPU 核心數量來決定開多少條 thread 的 thread pool，但最少會有兩條，適合用來執行偏向 CPU 運算的東西
- `Dispatchers.IO` 都是 thread pool，但是專為 I/O 處理而設。如果它的 thread pool 不夠 thread 用的話可以隨時開新的 thread，用完後會銷毀。其實背後都是跟 `Dispatchers.Default` 共用 thread pool，如果本身有東西在 `Dispatchers.Default` 執行中的話就不會把原先的東西切換 thread

Dispatcher 的用法是用 `launch`、`withContext` 包住想切換 thread 的部分：

```kotlin
suspend fun example() {
    println("這句在 call example() 的 dispatcher 執行")
    withContext(Dispatchers.Default) {
        println("這句在 Dispatchers.Default 執行")
    }
    println("現在回到原先的 dispatcher 執行")
}

// 如果整個 function 都想在 Dispatchers.Default 執行可以這樣寫
suspend fun example2() = withContext(Dispatchers.Default) {
    println("這句在 Dispatchers.Default 執行")
}
```

如果是 `Flow` 的話，亦可以用 `flowOn` operator 指明 dispatcher：

```kotlin
val myFlow: Flow<Int> = flowOf(1, 2, 3)
    .map { it * 2 }
    .onEach { println(it) }
    .flowOn(Dispatchers.Default) // 這句以上 (map, onEach) 是在 Dispatchers.Default 執行
    .filter { it % 2 == 0 } // 這句還是在 collect 那邊所在的 context 執行
```

不過為了方使測試，我們會用 dependency injection 取得 dispatcher。以下是用來提供 `CoroutineDispatcher` 的 Dagger module：

```kotlin
@Qualifier
@MustBeDocumented
@Retention(AnnotationRetention.RUNTIME)
annotation class IoDispatcher()

@Qualifier
@MustBeDocumented
@Retention(AnnotationRetention.RUNTIME)
annotation class DefaultDispatcher

@Qualifier
@MustBeDocumented
@Retention(AnnotationRetention.RUNTIME)
annotation class MainDispatcher

@Module
@InstallIn(SingletonComponent::class)
object CoroutinesModule {
    @Provides
    @IoDispatcher
    fun provideIoDispatcher(): CoroutineDispatcher = Dispatchers.IO

    @Provides
    @DefaultDispatcher
    fun provideDefaultDispatcher(): CoroutineDispatcher = Dispatchers.Default

    @Provides
    @MainDispatcher
    fun provideMainDispatcher(): CoroutineDispatcher = Dispatchers.Main
}
```

以上的 code 可以在 [GitHub repo](https://github.com/ericksli/eta-demo/blob/main/app/src/main/java/net/swiftzer/etademo/common/CoroutinesModule.kt) 找到。

習慣上我們會在實際需要開 thread 的地方指明 dispatcher，例如 `withContext(Dispatchers.IO) { ... }` 包住讀寫檔案的 code。因為在那個位置是最清楚自己需要用那個 dispatcher。如果把指明 dispatcher 的工作放到跟實際操作很遠的位置（例如 `Activity`）的話調用的時候就要額外花時間檢查那些 code 是不是要用其他 dispatcher 執行，在 Android Developers 的文檔亦建議 suspending function 應寫成能安全地在 main thread 上調用。

## 參考

- [Blocking threads, suspending coroutines](https://elizarov.medium.com/blocking-threads-suspending-coroutines-d33e11bf4761)
- [Improve app performance with Kotlin coroutines](https://developer.android.com/kotlin/coroutines/coroutines-adv)
- [Best practices for coroutines in Android](https://developer.android.com/kotlin/coroutines/coroutines-best-practices)
- [Kotlin flows on Android](https://developer.android.com/kotlin/flow)
- [Cancellation and timeouts](https://kotlinlang.org/docs/cancellation-and-timeouts.html)

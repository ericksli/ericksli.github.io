---
title: "Jetpack Compose 遷移 (2)"
date: 2022-07-26T14:00:00+08:00
tags:
  - Android
  - Jetpack Compose
---

[上一篇]({{< ref "jetpack-compose-migration-1" >}})提過如何將 [MetroRide](https://play.google.com/store/apps/details?id=net.swiftzer.metroride) 由傳統 view system 遷移到 Jetpack Compose。但一篇又太長，所以分拆成兩篇。

## Dependency injection

按照官方的建議，composable function 要用到的 dependency 應該由 caller 經參數提供。然後就是由外層一直傳進去。至於那個外層最遠可以去到 `Activity` 或者 `Fragment`。由於 composable function 就是 top-level function，沒有 class 包住，所以平常用開的 Dagger 或者 Koin 之類的 DI library 都無辦法輕易地在 composable function 經 DI 拿到 dependency。如果以 Dagger Hilt 來計，目前是有這幾種方式：

1. 從 `ViewModel` 的 constructor injection 拿到 dependency，然後用 Hilt 的 `viewModel` 或 `hiltViewModel` function 在 composable function 拿到 `ViewModel` instance，再從 `ViewModel` 拿到 dependency
2. 從 `Activity` 或者 `Fragment` 加 `@Inject` 做 field injection，然後帶進去 `setContent` lambda
3. 為 dependency 準備 [entry point](https://dagger.dev/hilt/entry-points.html)，`Context` 可以經 `LocalContext.current` 拿到

其實三個方法我覺得都是很不 Compose。但很多時都可以避免到在 composable function 拿 dependency。例如在 `ViewModel` 外露已經計算好結果的 `StateFlow`，composable function 就會接到 data class 形式的 state，不需接觸到其他 dependency。這樣做 preview function 都容易處理。Android platform 的 dependency 例如 system service 可以用 `LocalContext.current` 拿到 `Context` 之後就拿到。我目前是按照官方建議 screen level 的 composable function 可以拿到 `ViewModel`，但內裏用到的 composable function 部件就不會再接觸到 `ViewModel`。但即使限制 composable function 接觸到 `ViewModel` 的機會，但始終會有部分 composable function 因為 parameter 有 `ViewModel` 而不能寫 preview function。日後我會試試把 `ViewModel` 限制在 Navigation component 的 `composable`/`dialog` lambda 內使用，所有 `StateFlow` 全部在 navigation 那邊轉成 Compose 的 `State`、`ViewModel` function call 變成 callback lambda。這樣就更容易為 screen level 的 composable function 寫 preview function。因為現在沒有 XML 版 Navigation component 這種的圖像化 navigation graph，如果連 screen level 的 composable function 都無預覽的話去到大 project 就很難維護。

除了那三種方法外，如果是比較通用的東西可以用 `CompositionLocal` 而不用逐層傳入，例如剛才提及的 `LocalContext`。可以理解 `CompositionLocal` 就是 [service locator](https://en.wikipedia.org/wiki/Service_locator_pattern)。其實 Compose 的 Material Design theming 都用了 `CompositionLocal` 來減少傳遞參數。例子有 [`LocalContentAlpha`](https://developer.android.com/reference/kotlin/androidx/compose/material/package-summary#LocalContentAlpha()) 用來改變某個範圍的顏色透明度。我在這次遷移應該算是用得克制，只加了兩個自訂 `CompositionLocal`：一個是用來提供 Java 的 [`Clock`](https://docs.oracle.com/javase/8/docs/api/java/time/Clock.html)，另一個是用來拿用戶語言設定（因為 app 有自己的語言設定，不是按系統設定），方便顯示文字內容。用法大致上是這樣：

```kotlin
@Composable
fun MetroRideAppContainer(
    viewModel: MainViewModel,
    splashScreenVisibleCondition: (SplashScreen.KeepOnScreenCondition) -> Unit,
    quitApp: () -> Unit,
    // 略……
) {
    val appBlocking by viewModel.appBlocking.collectAsState()
    val uiLanguage by viewModel.uiLanguage.collectAsState()

    splashScreenVisibleCondition { appBlocking == AppBlocking.Checking }

    CompositionLocalProvider(
        LocalClock provides viewModel.clock,
        LocalUiLanguage provides uiLanguage,
    ) {
        val navController = rememberNavController()
        MetroRideTheme {
            AppNavHost(
                modifier = Modifier
                    .fillMaxWidth()
                    .weight(1f),
                navController = navController,
            )
        }
    }
}
```

## Interop

MetroRide app 如果講跟傳統 view system 之間的 interop 的話就只有 AdMob 的 `AdView` 需要用到，[Composable function 的 code](https://gist.github.com/ericksli/d4e4ab21ddfcb0054bd8852f3b9ff31e) 我已經放到 Gist。但 LeakCanary 有時會投訴 AdMob 的 `WebView` 會有 memory leak。暫時找不到解決方法。另外亦為 IDE preview 做了個專用 UI，方法是檢查 `LocalInspectionMode` 的 `current`。

{{< figure src="adview-preview.png" title="AdMob composable function preview" >}}

至於其他非傳統 view system 之間的 interop，其實 Google 之前都出了一堆 library 讓你在用 Compose 時減少直接接觸 Android platform 的功能。例如 Jetpack Lifecycle-aware components 將 `Activity` lifecycle 抽離、Activity Result APIs 取代 `startActivityForResult`、`SavedStateHandle` 取代 `onSaveInstanceState` 等等其實都會在 Compose 上用到。在 AdMob 那個 Gist 已經透過 `LifecycleEventObserver` 來得知當前的 lifecycle 狀態。

## 主題

Jetpack Compose 有提供 [Material Design](https://material.io/design/introduction) 2 和 3 的 artifact。目前我仍使用 2，因為 3 還在 alpha，組件未夠齊（即使 Compose 版 Material Design 2 都是比傳統 view system 版少組件）。但講自訂程度我覺得 Compose 比傳統 view system 更好。傳統 view system 有 style resource 但要花時間查有那些地方可以用 style 改動，但 Compose 可以自己查 Material Design 組件的 composable function 源碼，發覺有地方不能自訂就把它的 code 抄走再改。而且 Compose 部分組件有提供原始版 composable function，例如負責文字輸入的 [`BasicTextField`](https://developer.android.com/reference/kotlin/androidx/compose/foundation/text/package-summary#BasicTextField(kotlin.String,kotlin.Function1,androidx.compose.ui.Modifier,kotlin.Boolean,kotlin.Boolean,androidx.compose.ui.text.TextStyle,androidx.compose.foundation.text.KeyboardOptions,androidx.compose.foundation.text.KeyboardActions,kotlin.Boolean,kotlin.Int,androidx.compose.ui.text.input.VisualTransformation,kotlin.Function1,androidx.compose.foundation.interaction.MutableInteractionSource,androidx.compose.ui.graphics.Brush,kotlin.Function1))，它就是單純提供文字輸入功能。你喜歡加標題可以自己另外加個 `Text`、框線等等。但如果是傳統 view system 你要抄走某個 class 的 code 都不容易，因為要兼顧 Java/Kotlin code 的 access modifier 和 XML styleable attribute。

MetroRide 的 UI 因為都是按照 Material Design 2 而造，所以沒有做太多自訂主題。最多都是按照 Material Design 2 做了 dialog（Compose 版 Material Design 2 做出來 dialog 跟 Material Design 網站講的式樣不同，所以我特別處理過）和搜尋欄。但在舊公司用 Compose 開發新功能時因為那個 app 的主題跟 Material Design 有出入所以試過自訂過 UI。大概是寫一個 composable function 包住 Compose 本身提供的組件 composable function，但已經把需要調整的參數都設定好，又或者是現有的組件被其他新組件包住。這個是 Compose 做自訂 UI 組件的方法：composition。

## 小結

Compose 遷移就寫到這裏。其實遷移工作斷斷續續地做了半年。因為是 side-project 沒有時間限制，做了一部分又放下來做其他東西，再加上重鐵和輕鐵抵站時間在很久之前做了出來（重鐵那個在上去年參加 [iThome 鐵人賽]({{< ref "2021-ithome-ironman" >}})時拿來做示範 app，輕鐵是 MetroRide app 之前已經有的功能），隔了一段時間後都忘了之前的 code 寫了甚麼。其實還想分享一下 Navigation component，不過我覺得可以再開多一篇文。所以這部分會在下一篇出現。

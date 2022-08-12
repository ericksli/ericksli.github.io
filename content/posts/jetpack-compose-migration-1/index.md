---
title: "Jetpack Compose 遷移 (1)"
date: 2022-07-24T14:06:00+08:00
tags:
  - Android
---

近幾個月斷斷續續替 [MetroRide](https://play.google.com/store/apps/details?id=net.swiftzer.metroride) 的界面由傳統 view system（即是 layout XML）轉為 [Jetpack Compose](https://developer.android.com/jetpack/compose)，順帶補上去年參加 [iThome 鐵人賽]({{< ref "2021-ithome-ironman" >}})時用來做示範的重鐵抵站時間功能。昨天新版 app 上架了就來分享一下遷移過程。其實這個 app 在之前的版本有把其中一頁靜態的頁面（延誤定義）改用 Compose，那時是在 `Fragment` 內的 `onCreateView` 加入 `setContent` 顯示 Compose 內容。由於那時只是做排版，沒有遇到大問題。之後開始慢慢轉用 Compose 才遇到問題。

## 遷移方式

由於我本身打算一次過把全個 app 的 UI 都換成 Compose，所以除了先前的延誤定義頁之外我就沒有再特別去把原有的 `Fragment` 淘空 layout XML，而是直接造一個全新的 `MainActivity` 來顯示全新製造的 Compose UI，舊有的 layout XML 和 `Fragment` 就盡量不去改，方便遷移時能拿來對照參考。當全部頁面都轉成 Compose 後就會刪除舊的 `Fragment` 和 layout XML。另外，這個 app 本身是 single `Activity` app，所以只有單一 `Activity`，遷移後亦都會維持這個做法。

除了大改的列車抵站時間部分，原有頁面的 `ViewModel` 和背後的 code 都不會大改，因為這次目的是想轉用 Compose 和補元列車抵站時間功能。現有 code 在 `ViewModel` 和 UI 之間的通訊方式是以 `LiveData` 和 `ViewModel` 的 function，再加上使用 databinding。在遷移現有頁面時 `ViewModel` 要改動的地方不算太多，而背後拿資料的部分都不用改。

<!-- more -->

## 狀態

Compose 和傳統 view system 其中一個明顯差別就是狀態處理。Compose 特別強調狀態，明確地分別出 composable function 的參數、`remember` 和 `rememberSavable`。`remember` 就是在 Compose rerender (recompose) 時能保存的 state，而 `rememberSavable` 就更進一步連 process 被 kill 後重開 app 都能保存 state。即是跟 `savedInstanceState` 一樣，只是重新包裝方便將 Compose 帶去非 Android 平台用。

官方建議盡量寫 stateless 的 composable function，把 state 都推去上層管理（正式叫法是 [state hoisting](https://developer.android.com/jetpack/compose/state#state-hoisting)），你可以把 state 推到上去 `ViewModel`，即是跟傳統 `Activity`/`Fragment` 般一個頁面配一個 `ViewModel` 來管理 state。但不代表要把所有 state 都推到去 `ViewModel` 管理。一般建議的做法是比較關 UI 事的 state 就保留在 composable function 內放置（靠 `remember` 和 `rememberSavable`），例如 scroll、animation 的 state。而偏向資料性的 state 就交由 `ViewModel` 管理。目前我都在摸索除了那些很明顯的 scroll、animation 之類的 UI state 外，還有甚麼 state 是應該放去 Compose 那邊。以顯示列車班次為例，UI 要顯示班次抵站的倒數時間。究竟應該把定時計算倒數分鐘的 code 放去非 Compose 的地方，然後經 `ViewModel` 的 `StateFlow` 傳去要顯示分鐘的 composable function 讓它直接拿那個預先計算好的數字顯示出來還是 `ViewModel` 只向 composable function 提供絕對時間（`Instant` 之類），composable function 會定時計算倒數分鐘並顯示出來（會有 side-effect）。這次我選了前者，但下次再選的話可能會選後者。我沒有太仔細看官方的 sample project，但似乎因為現在 UI 的部分可以直接寫 Kotlin code，不像以往 XML layout 般難把 Java/Kotlin code 塞入去（databinding 都是 Java syntax 而且不能分行寫），現在可以把偏 UI 相關的 code 都放去 composable function 內。但因為這次沒有寫 UI test，反而是要刪走以前針對傳統 view system 寫的 UI test，所以還未能拿捏到應不應該把這類的 logic 下放到 composable function 內。

這次因為遷移時還保留着現有的 `Fragment` 和 layout XML，所以不太想修改現有的 `ViewModel` 以免動到舊有的 code。我之前在舊公司試過新頁面用 Compose 寫，在寫 code 時其實是會自然地傾向把 composable function 寫到 stateless 和減少 logic，最好連 UI 顯示式樣都由非 Compose 的地方計算好。以 WhatsApp 的訊息列表為例（見下圖），會以 sealed interface/class 的形式向 UI 表示每間房間的最新訊息欄的式樣（自己發送單剔、自己發送雙剔、別人發送、房間沒有訊息等等），然後 composable function 放個 `when` 來實際控制 UI 顯示式樣（例如顏色、icon），而不是在 composable function 內拿到個未處理的 object 再在 composable function 內寫一堆 code 去決定顯示那款最新訊息式樣。同時亦會慢慢傾向造一個 `StateFlow` 去表達整頁的 state。如果要用 data class 並以 immutable 的方式表達整頁的 state 的話，在複雜的環境下可能需要借助 [kotlinx.collections.immutable](https://github.com/Kotlin/kotlinx.collections.immutable) 和 [Arrow Optics](https://arrow-kt.io/docs/optics/) 之類的 library 來生成新的 state object，因為那些 state 可能放到很深入，用 data class `copy` function 不夠方便，會很累贅。

{{< figure src="whatsapp-last-message.png" title="WhatsApp 最新訊息欄" >}}

另一樣應該都算關 state 事的地方是 Compose 對那些要費時完成的動作（例如 scroll 去某個位置、彈出 bottom sheet 之類）的 function 都定義為 suspending function。以往在傳統 view system 可能只是 call 完 function 就算，但 Compose 就要你開一個 coroutine scope 才能 call。

## Navigation

由於是另起爐灶，Navigation 我沒有做甚麼遷移，要做的東西就是重新做過一個新的 `NavHost`。如果是分階段遷移的話一般都是叫你用 `Fragment` 來顯示 composable function 並沿用現有的 XML 式 navigation graph 直頭全部頁面都換成 Compose 才轉用 Compose 版 Navigation。Compose 版 Navigation 跟 XML 明顯分別是沒有 IDE 預覽、轉用 route（string 形式）而不是用 R class ID 和沒有 SafeArgs，而傳遞參數都改以 query parameter 形式而不是用 `Bundle`。很多人都不習慣這個新設定，有人用 Kotlin Symbol Processing (KSP) [還原了 SafeArgs 功能](https://github.com/raamcosta/compose-destinations)。官方就[建議大家寫 sealed class 來表達 route](https://developer.android.com/jetpack/compose/navigation#bottom-nav)。我覺得 Compose Navigation 應該比 XML 版 Navigation 更容易做到 runtime 隨時切換 navigation graph（例如用戶登入前和登入後），過份追求跟 XML 版 Navigation 功能一致可能會令隨時切換 navigation graph 變得更難做到。另一樣比較多人不喜歡的地方是傳遞參數要用 query parameter。我初初以為 route 是像 RESTful 般如果有子目錄就是代表是某一個 route 的子頁 (sub-graph)，但實際試用過才發現那些 route 就只是一個名字而已。而不能傳 `Parcelable` 參數是因為官方建議是應傳 ID 之類的簡單參數，開了另一頁後才憑那個 ID 從 local storage（SQLite database、Data Store、SharedPreferences、file system）找回那個大 object。以前工作遇過有人會用 singleton object 來暫存那些大 object，因為太大不能塞進去 intent extras。但這個做法沒有考慮到系統 kill process 後用戶從 [Recents screen](https://developer.android.com/guide/components/activities/recents) 再次打開 app 的情景，所以還是應該把大 object 放去 local storage。

另外，route 其實跟 deep link 沒有關連，因為 deep link 是另外宣告的，但 deeplink 用到的 argument 要在 `arguments` 一併宣告。

```kotlin
composable(
    route = "stationDetail/{id}",
    arguments = listOf(
        navArgument("id") {
            type = NavType.IntType
            defaultValue = -1
        },
        navArgument("lrlStopId") {
            type = NavType.IntType
            defaultValue = -1
        },
    ),
    deepLinks = listOf(
        navDeepLink {
            uriPattern = "https://lrnt.mtr.com.hk/moblink/?station_id={lrlStopId}"
        },
    ),
) {
    val viewModel = hiltViewModel<StationDetailViewModel>()
    StationDetailScreen(
        viewModel = viewModel,
        onNavigateUpClick = { navController.navigateUp() },
    )
}
```

加完 `deepLinks` 後記得要自己在 *AndroidManifest.xml* 加上 `intent-filter`，不然就做不到 deep link。另外，在 Android 12 (API level 31) 起 deep link 的 domain 必須要放置 [Digital Asset Links JSON](https://developer.android.com/training/app-links/verify-site-associations)，否則預設只會開 web browser。惟用戶可以自行到 app settings 開啟 deep link 功能。

對於那些 SafeArgs 的 library，我反而期望有人做到類似 backend framework routing 的 middleware/interceptor 功能（又或者是 OkHttp 的 interceptor）。即是每次收到要轉頁的請求時都可以先經過一些 middleware，middleware 可以決定是否放行或者轉址去另一頁，而那些 middleware function 要是 suspending function。有了 middleware 就可以很容易做到登入後才能進入某些頁面的要求，不用再在每一頁加入登入檢查的 code。這大概是要封裝一次 Compose Navigation，幫它加上 queue 來為轉頁動作排隊，還要補上等待轉頁時的載入中畫面。

有一樣東西我想分享是如何做到 navigation graph level 的 `ViewModel` 共用，那時在做這個部分找到一大輪才知道這個寫法。我的 project 用了 Dagger Hilt，如果不是用 Dagger Hilt 的話可以把 `hiltViewModel` 改成對應寫法。重點是要用 `navController.getBackStackEntry` 拿到 navigation graph 的 entry，再憑那個 entry 設定 `ViewModelStoreOwner` 就能拿到同一個 `ViewModel` instance。下面是設定頁及其 dialog 的 sub-graph，dialog 會與設定頁共用同一個 `SettingsViewModel`。

```kotlin
@Composable
fun AppNavHost(
    modifier: Modifier = Modifier,
    navController: NavHostController,
) {
    NavHost(
        modifier = modifier,
        navController = navController,
        startDestination = "status",
    ) {
        // 略……
        settingsGraph(navController)
    }
}

fun NavGraphBuilder.settingsGraph(navController: NavHostController) {
    navigation(startDestination = "settingsHome", route = "settings") {
        composable("settingsHome") { backStackEntry ->
            val parentEntry = remember(backStackEntry) {
                navController.getBackStackEntry("settings")
            }
            val viewModel = hiltViewModel<SettingsViewModel>(parentEntry)
            SettingsScreen(
                viewModel = viewModel,
                navigateUp = { navController.popBackStack() },
                // 略……
            )
        }
        dialog(
            route = "language?index={index}",
            arguments = listOf(
                navArgument("index") {
                    type = NavType.IntType
                    defaultValue = -1
                },
            ),
        ) { backStackEntry ->
            val parentEntry = remember(backStackEntry) {
                navController.getBackStackEntry("settings")
            }
            val viewModel = hiltViewModel<SettingsViewModel>(parentEntry)
            val selectedIndex = backStackEntry.arguments?.getInt("index") ?: -1
            LanguageSettingDialog(
                preSelected = LanguageSetting.values()
                    .getOrElse(selectedIndex) { LanguageSetting.SYSTEM_DEFAULT },
                onConfirmSelect = { languageSetting ->
                    viewModel.updateLanguage(languageSetting)
                    navController.popBackStack()
                },
                onDismissRequest = { navController.popBackStack() },
            )
        }
        dialog(
            route = "theme?nightMode={nightMode}",
            arguments = listOf(
                navArgument("nightMode") {
                    type = NavType.IntType
                    defaultValue = AppCompatDelegate.MODE_NIGHT_UNSPECIFIED
                },
            ),
        ) { backStackEntry ->
            val parentEntry = remember(backStackEntry) {
                navController.getBackStackEntry("settings")
            }
            val viewModel = hiltViewModel<SettingsViewModel>(parentEntry)
            val nightMode = backStackEntry.arguments?.getInt("nightMode")
                ?: AppCompatDelegate.MODE_NIGHT_UNSPECIFIED
            ThemeSettingDialog(
                preSelected = nightMode,
                onConfirmSelect = { newValue ->
                    viewModel.updateTheme(newValue)
                    navController.popBackStack()
                },
                onDismissRequest = { navController.popBackStack() },
            )
        }
    }
}
```

## 小結

其實實際的遷移工作不用做這麼久，只是因為這是 side project 沒有時間限制所以才慢慢做。這次先分享到這裏，我本來想一併分享其餘內容，但都是留待下篇才寫。下次應該會寫 interop 的部分。

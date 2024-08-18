---
title: AndroidX Navigation component for Jetpack Compose type safety
date: 2024-08-18T22:00:00+08:00
tags:
  - Android
---

AndroidX Navigation component 是 Google 推出的 [single `Activity` app](https://www.youtube.com/watch?v=2k8x8V77CrU) navigation library。本身是用 `Fragment` 來做每一頁的內容，然後再用新的 Android resource type——navigation 來定義 navigation graph（即是聲明一個 navigation graph 內有什麼 `Fragment`、打開 `Fragment` 時要什麼參數和各 `Fragment` 之間如何導覽的 XML 檔案）。如果加上 [Safe Args](https://developer.android.com/guide/navigation/use-graph/safe-args) Gradle plugin 的話就會按 navigation graph XML 檔案生成那些 Java code 去讓你在 `Fragment` 內轉頁時調用，那就不會怕轉頁時漏了幾個參數沒有傳到，因為漏了的話就不能成功 compile。

<!-- more -->

## Navigation component for Compose 的 navigation 方式

AndroidX Navigation 有推出了 Compose 版，其實核心都是沿用原本處理 `Fragment` navigation 的那些 code。但每頁的「代號」由 Android resource ID 改為網址式的 string 表示（叫做 route）。轉頁在 AndroidX Navigation Compose 的理解就是按網址找到對應的 composable function 去 call 然後在網址抽取打開該頁要用到的參數。用法大概就是這樣：

```kotlin
composable("shop/{shopId}?promoCode={promoCode}") { backStackEntry ->
    val shopId: String? = backStackEntry.arguments?.getString("shopId")
    val promoCode: String? = backStackEntry.arguments?.getString("promoCode")

    // 該頁顯示的內容
    Column {
        Text(shopId.orEmpty())
        Text(promoCode.orEmpty())
    }
}
```

可以見到 `shop/{shopId}?showPromo={showPromo}` 就是該頁面的「代號」，而 `shopId` 是必需而 `showPromo` 是可選。要打開該頁的話就要用 `navController.navigate`：

```kotlin
navController.navigate("shop/abc123?promoCode=foo")
```

如果要傳更多參數就用 `/` （必需參數）和 `&` （可選參數）串接起來。這個網址單純就是為了表達不同的 route 和要傳遞的參數，跟 RESTful 沒有關係。

轉用網址來代表頁面的好處是方便 multiplatform。因為 resource ID 和 `Parcelable` 是 Android 才有（`Fragment` 版 AndroidX Navigation 是可以傳 `Parcelable` 參數），加上把參數塞到網址就可以很簡單地處理到 deep link。但對 Android 開發者而言是不太習慣，尤其是沒有簡單直接方法傳 `Parcelable` 參數和 [Fragment Result API](https://developer.android.com/guide/fragments/communicate#fragment-result) 功能。可能因為 Google 的目標是想把 AndroidX Navigation 在 web app 都能用所以就把這些原本 `Fragment` 版有的東西都收起不做，然後直接在文檔寫不建議傳複雜 object，應該要傳 object 的 ID 然後再從 database 之類的地方拿 data，因為怕會在 `Activity.onSaveInstanceState` 塞爆。我覺得防止塞爆是合理，因為要考慮到 app 放在 background 後被 system kill app 然後 user 再從 recents 開 app 需要還原 kill app 前的狀態（雖然很多 Android developer 不去管這個問題，然後以為現在的裝置有很多 RAM 不會輪到自己的 app 被 kill，被 kill 之後重開 app 還原不了先前的狀態就爛掉）。但如果只是傳一些簡單的 object 要動用到 database/file system 開一個檔案又太勞師動眾。（下面會示範如何傳 `Parcelable` 參數）

如果想了解更多 Compose 的 Navigation component 使用方式可以查閱 2022 年寫的「[Jetpack Compose Navigation component sub-graph]({{< ref "jetpack-compose-navigation-component-sub-graph" >}})」一文。

## Type safety 的 navigation 方式

在 AndroidX Navigation 2.8.0 開始（目前是 beta）Compose Navigation 可以改用相對比較 type safety 的方法定義 route。其實之前已經有其他人用 [Kotlin Symbol Processing (KSP)](https://github.com/google/ksp) 去生成 Kotlin code 去令 Compose Navigation 做到 type safety 的效果（生成那串網址、從 back stack entry 拿到參數的部分），但 Google 就不用 KSP 而是借用 [Kotlin Serialization](https://github.com/Kotlin/kotlinx.serialization) Gradle plugin 生成的 `$serializer` class 來做到同樣效果（其實單純做 navigation route 是不用再加 [JSON/ProtoBuf 之類的 artifact](https://github.com/Kotlin/kotlinx.serialization/blob/master/formats/README.md)）。我們會以下面那個 data class 做例子。這個 data class 有兩個 property，就是那個 route 的兩個參數。如果沒有 default value 就是必需，相反就是可選參數。

```kotlin
@Serializable
data class StationDetailDestination(
    val id: Int: Int = -1,
    val lrlStopId: Int = -1,
)
```

如果這個頁面沒有參數要傳就改用 `object`/`data object` 再配搭 `@Serializable`。

之後 `composable` 的寫法改為傳 type 而不再傳 `route` 和 `arguments`。

```kotlin
composable<StationDetailDestination>(
    deepLinks = listOf(
        navDeepLink {
            uriPattern = "https://lrnt.mtr.com.hk/moblink/?station_id={lrlStopId}"
        },
    ),
) {
    // 內容
}
```

之前的寫法：

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
        // 略
    ),
) {
    // 內容
}
```

新寫法不用再傳 `route` 和 `arguments` 是因為 Navigation component 改為讀取 Kotlin Serialization Gradle plugin 生成的 code。當 class 被加了 `@Serializable` 後，Kotlin Serialization Gradle plugin 會生成 `$serializer` class。把 `StationDetailDestination` 的 JVM bytecode decompile 後會找到這段內容：

```java
static {
    PluginGeneratedSerialDescriptor var0 = new PluginGeneratedSerialDescriptor(
        "net.swiftzer.metroride.app.station.detail.StationDetailDestination",
        (GeneratedSerializer) INSTANCE,
        2
    );
    var0.addElement("id", true);
    var0.addElement("lrlStopId", true);
    descriptor = var0;
    $stable = 8;
}
```

如果我們 call `StationDetailDestination.serializer().descriptor` 的話可以找回那個 data class 的名稱和有什麼 property：

```kotlin
val descriptor = StationDetailDestination.serializer().descriptor
println(descriptor.serialName)

for (i in 0 until descriptor.elementsCount) {
    println("${descriptor.getElementName(i)}, ${descriptor.getElementDescriptor(i).kind}")
}
```

以下是 output：

```text
net.swiftzer.metroride.app.station.detail.StationDetailDestination
id, INT
lrlStopId, INT
```

所以 Navigation component 就能憑以上的方法生成出以往要人手寫的 `route` 和 `arguments`（對，都是沿用之前以網址形式做 navigation，只是你不會直接看到那個網址 route）。

要打開剛才那頁就這樣寫：

```kotlin
navController.navigate(StationDetailDestination(id = 123))
```

以上的 `navigate` 會生成出這樣的 deep link 網址：

```text
android-app://androidx.navigation/net.swiftzer.metroride.app.station.detail.StationDetailDestination?id=123&lrlStopId=-1
```

Route 名前面那部分就是 data class/object 的全名。Deep link 前面那段 `android-app://androidx.navigation/` 就是 Navigation component 專屬的 deep link 前綴。

至於如何讀取傳入 route 的參數？Navigation component 提供了 `toRoute` function 來讓你取回填好參數的 object。

以下是 `NavBackStackEntry` 版：

```kotlin
composable<StationDetailDestination>(
    // 略
) { backStackEntry ->
    val args: StationDetailDestination = backStackEntry.toRoute<StationDetailDestination>()
    println(args.id) // Int
}
```

以下是 `SavedStateHandle` 版：

```kotlin
@HiltViewModel
class StationDetailViewModel @Inject constructor(
    savedStateHandle: SavedStateHandle,
) : ViewModel() {
    private val args: StationDetailDestination = savedStateHandle.toRoute<StationDetailDestination>()
    init {
        println(args.id) // Int
    }
}
```

而 sub-graph 都是同樣做法，不再傳 string：

```kotlin
@Serializable
data object StationGraph

@Serializable
data object StationListDestination

navigation<StationGraph>(startDestination = StationListDestination) {
    composable<StationListDestination> {
        // 略
    }
}
```

### `Parcelable`

要在 Navigation component Compose 傳 `Parcelable` 的話是比較麻煩，即使有了 type safety 寫法都沒有用，仍然要手寫一段 code 才能做到。

下面示範了要傳遞一個 `Reason` enum 到 `ScanErrorDestination` 的寫法：

```kotlin
@Parcelize
@Serializable
enum class Reason : Parcelable {
    UnsupportedCard,
    TagLost,
}

@Serializable
data class ScanErrorDestination(val reason: Reason)

fun NavGraphBuilder.scanErrorScreen(
    navigateUp: () -> Unit,
) {
    composable<ScanErrorDestination>(
        typeMap = ScanErrorScreenMap, // 之後會有這段 code
    ) { backStackEntry ->
        val args = backStackEntry.toRoute<ScanErrorDestination>()
        ScanErrorScreen(
            reason = args.reason,
            modifier = Modifier.fillMaxSize(),
            navigateUp = navigateUp,
        )
    }
}
```

上面 `composable` 多了一個 `typeMap` 參數，這個就是用來處理 `Parcelable` 的部分。以下是 `ScanErrorScreenMap` 的內容：

```kotlin
val NavigationProtoBuf = ProtoBuf { encodeDefaults = false }
val ReasonType = object : NavType<Reason>(isNullableAllowed = false) {
    override fun get(bundle: Bundle, key: String): Reason? =
        BundleCompat.getParcelable(bundle, key, Reason::class.java)

    override fun put(bundle: Bundle, key: String, value: Reason) {
        bundle.putParcelable(key, value)
    }

    override fun serializeAsValue(value: Reason): String =
        NavigationProtoBuf.encodeToHexString(value)

    override fun parseValue(value: String): Reason =
        NavigationProtoBuf.decodeFromHexString<Reason>(value)
}
val ScanErrorScreenMap = mapOf(typeOf<Reason>() to ReasonType)
```

`ScanErrorScreenMap` 就是用來定義 `ScanErrorDestination` 內的 property 如果有非 Navigation component 直接支援的 type 時要用對應的 `NavType` 來做 serialization/deserialization。`NavType` 簡單來講就是寫 serialization/deserialization 成 `Parcelable` 和 `String` 的實際操作部分。

先講 `Parcelable`，它就是平時轉頁時會用的 serialization 形式，對應的 function 是 `get` 和 `put`。只要在那個 enum 加上 `@Parcelize` 和 implement `Parcelable` 然後再 call 對應的 `Bundle` function 就做到。

另外兩個 function `serializeAsValue` 和 `parseValue` 是要把那個 enum 轉成 string 形式，它是對應 deep link（估計亦是日後支援 multiplatform 的處理方法，因為 `Parcelable` 是 Android 獨有的東西）。我就把那個 enum 用 Kotlin Serialization 轉成 ProtoBuf 十六進制 string 表示，你亦可以用 JSON 之類，但記得要做 URI encode/decode。

以前例子是傳 enum，其實大可傳 enum value 的 `ordinal`，這樣就不用寫那麼多 code。但如果是 data class 的話都是要這樣寫。

另外，如果在 `ViewModel` 內想透過 `SavedStateHandle` 獲取參數的話，要在 `SavedStateHandle.toRoute` 補回 `typeMap` 參數。`NavBackStackEntry.toRoute` 不用是因為 `NavBackStackEntry` 能找到 `typeMap`。

```kotlin
private val args = savedStateHandle.toRoute<ScanErrorDestination>(
    typeMap = ScanErrorScreenMap,
)
```

### Analytics

如果要做 screen view 式的 event tracking 而又不想逐頁加 code，可以 collect `NavHostController.currentBackStackEntryFlow` 來得知轉頁並在這個時候做 event tracking。下面是用 Firebase Analytics 做例子：

```kotlin
LaunchedEffect(navController) {
    navController.currentBackStackEntryFlow.collectLatest { backStackEntry ->
        backStackEntry.destination.route?.let {
            Firebase.analytics.logEvent(FirebaseAnalytics.Event.SCREEN_VIEW) {
                param(FirebaseAnalytics.Param.SCREEN_CLASS, it)
            }
        }
    }
}
```

如果用上面那個例子的話，`backStackEntry.destination.route` 其實就是 `net.swiftzer.metroride.app.station.detail.StationDetailDestination?id={id}&lrlStopId={lrlStopId}`，不會有參數的值。

### 是否完全 type safefy？

其實上面的 code 都示範了 deep link 的話仍然是要人手寫，連同 Android manifest 的 deep link `<intent-filter>`都要自己寫，所以在 compile 時不能察覺寫錯（如果用 XML 版的話是有[半自動方法](https://developer.android.com/guide/navigation/design/deep-link#implicit)生成 `<intent-filter>`）。所以 type safety 只是針對之前手寫 route 要聲明參數和讀取參數的部分而已。總體效果不如以前 XML 般，有不少地方仍要手寫 code 而且不是 compile 時檢查，例如 deep link 和 `NavType` adapter。

## 安全問題

今天逛 Reddit 發現了一篇名為「[Russian hackers destroy Jetpack Navigation from its very core, turning best practice into security vulnerability in the blink of an eye](https://www.reddit.com/r/mAndroidDev/comments/1etw0t0/russian_hackers_destroy_jetpack_navigation_from/)」的文章，內容是連結到 [Android Jetpack Navigation: Go Even Deeper](https://swarm.ptsecurity.com/android-jetpack-navigation-go-even-deeper/)。大意是如果你的 app 用了 AndroidX Navigation component for Compose 的話其實是可以讓人進入 `NavHost` 內任意一頁。

按文章內容所描述，進入任意一頁的方法是找出放了 `NavHost` 的那個 `Activity`，然後用 `Intent` 開它，開的時候要附帶 `data` `Uri`，`data` 內容就是 Navigation component 自動生成的 route（即是 `android-app://androidx.navigation/` 開首的網址）。其實不用另外做另一個 app 去 call `Intent.startActivity`，直接用 [Android Debug Bridge (adb)](https://developer.android.com/tools/adb) 都可以：

```shell
adb shell am start -W -a android.intent.action.VIEW -d "android-app://androidx.navigation/net.swiftzer.metroride.app.setting.SettingsDestination" net.swiftzer.metroride/net.swiftzer.metroride.app.entrypoint.MainActivity
```

這樣就可以直接開到 [MetroRide](https://play.google.com/store/apps/details?id=net.swiftzer.metroride) 的設定頁。相信這對需要寫 end-to-end test 的人是個好消息，是一個 feature 而不是 bug。但如果你的 app 有部分頁面是需要登入後才能看的話這就比較尷尬（如果你不是在每頁都加登入檢查的話），因為這個方法可以繞過登入頁。

其實能夠打開到設定頁的原因是因為 `NavController` 會拿 `Activity.intent` 去檢查 graph 內有沒有頁面能對應到這個 deep link（`Activity.intent.data` 就是 deep link URI），而那些由 Navigation component 自行生成的 route（即是 `android-app://androidx.navigation/` 開首的網址）都會被 match 到。結果就能中門大開隨意進入 `NavHost` 內任何一頁。

{{< figure src="nav-controller.png" title="NavController 那一句 handleDeepLink" >}}

但似乎 Google 未修正這個問題，如果想馬上解決的話除了不用 Navigation component 之外就是把 `Intent.data` 清走。

在 `Activity` 的 `onCreate`（在 `setContent` 前）加入這個 `if`：

```kotlin
if (intent.data?.scheme == "android-app" && intent.data?.authority == "androidx.navigation") {
    intent.data = null
}
```

如果 `Activity` 本身有設定 launch mode 是 `singleTop` 的話，亦應在 `onNewIntent` 刪走 `Intent.data`。

這篇文章其實有示範到其中一頁有個 `WebView` 而 `WebView` 載入的網址是從開那頁的參數提供並且那個 `WebView` 會在 request header 加入 token，結果透過這個方法就能拿到 token。這個其實應該在每次加入 token 前就要檢查一次 request URL 是不是指定的 URL 才加入 token 的 request header。如果想加強 JavaScript 與 native app 雙向溝通安全性的話可以參考 [Android WebView 筆記]({{< ref "android-webview" >}})。

## 結語

我覺得很多 Android/iOS developer 在做 navigation 時沒有像那些 backend framework 處理 request 的思維。一般那些 backend framework 在處理 request/response 時都會經過一些 [middleware](https://laravel.com/docs/11.x/middleware)，例如你可以定義一個 middleware 用來檢查用戶是否已登入，然後把它套在需要登入後才能進入的 route。這個 middleware 的大概內容是如果沒有登入就重定向到登入頁，然後直接 output response，不用再交去下一個 middleware 或者是該 route 對應的 controller。這其實跟 OkHttp 的 [interceptor](https://square.github.io/okhttp/features/interceptors/) 相似。背後的 design pattern 是 [Chain of Responsibility](https://refactoring.guru/design-patterns/chain-of-responsibility)，是 [Gang of Four Design Patterns](https://www.amazon.com/Design-Patterns-Object-Oriented-Addison-Wesley-Professional-ebook/dp/B000SEIBB8) 內的 pattern。

Google 將登入後才能進入否則重定向稱為 [conditional navigation](https://developer.android.com/guide/navigation/use-graph/conditional)。它建議先讓用戶進入受限頁面，然後再用 `Activity` 層級開一個 `UserViewModel`，入面決定用戶是否已登入。而在受限頁面的 `onViewCreated` 就 observe `UserViewModel` 外露的是否已登入 `LiveData`。當發現未登入就馬上重定向到登入頁（準碓來說是在在受限頁面上顯示登入頁，沒有清除 back stack）。正因為沒有清除 back stack，用戶可以在登入頁按返回鍵回到之前的受限頁面。為了防止不斷重定向，它在受限頁面的 `SavedStateHandle` 加了一個 boolean flag 表明用戶是否成功登入，然後在受限頁面檢查那個 boolean flag 防止不斷在兩頁之間重定向。當然，今時今日不會有人這樣寫，因為兩個 `ViewModel` 之間通訊是很麻煩的事。通常都是用 dependency injection graph 加上一個能共用 instance 的 class 然後在入面放那些是否已登入的 `Flow`/`LiveData`/`Observable`。但 Google 這個提議最大問題是當頁面一多就很難 scale，因為它要求每頁都要加檢查的 code。

如果我們沿用 AndroidX Navigation component 又想做到類似 middleware 的效果，大概就是弄一個 suspending function 包起 `navController.navigate`，在那個 suspending function 內檢查要 navigate 的 route 有沒有 middleware 要執行，有的話就逐個執行。如果最終確認是可以直接進入目標頁面那就可以執行 `navController.navigate`。如果是要重定向的話就要為重定向頁做同樣的 middleware 執行動作，直到找到最終目的地為止。而用 suspending function 的原因是那些 middleware 可能需要做 I/O 動作，不像 `navController.navigate` 可以馬上完成。而 `NavHost` 外面要加一個載入畫面，當執行 middleware 時就蓋在 `NavHost` 上面不能讓用戶按到畫面其他地方。這樣就不用每一頁都要再檢查一次用戶是否已登入，因為在進入前已經被 middleware 檢查過。

很多 navigation library 偏向是補強 AndroidX Navigation component 欠缺生成 boilerplat code 的功能、或者令整個 app 變得更 MVI。反而中國出品的 navigation library 會做這類接近 middleware 的東西，但不知道有沒有考慮非同步的問題。它們甚至能做到由 backend 控制 navigation 的目的地（例如因應 A/B testing、feature flag 的值在 navigation 時開啟不同頁面）。

## 參考

- [Navigation Compose meet Type Safety](https://medium.com/androiddevelopers/navigation-compose-meet-type-safety-e081fb3cf2f8)
- [Type-Safe Navigation with the OFFICIAL Compose Navigation Library](https://www.youtube.com/watch?v=AIC_OFQ1r3k)
- [Android Jetpack Navigation: Go Even Deeper](https://swarm.ptsecurity.com/android-jetpack-navigation-go-even-deeper/)
- [TheRouter](https://therouter.cn/)

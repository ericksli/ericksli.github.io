---
title: "2021 iThome 鐵人賽 Day 18：Navigation (2)"
tags:
  - 2021 iThome 鐵人賽
  - Android
date: 2021-10-03T00:00:00+08:00
canonicalURL: https://ithelp.ithome.com.tw/articles/10276786
---

> 本篇文章是 [2021 iThome 鐵人賽](https://ithelp.ithome.com.tw/2021ironman)參賽題目「[寫一個列車抵站時間 Android App](https://ithelp.ithome.com.tw/users/20139666/ironman/4661)」的第 18 篇，你可到 iThome [查看原文](https://ithelp.ithome.com.tw/articles/10276786)。
>
> [文章目錄]({{< ref "2021-ithome-ironman" >}})

在 Android，navigation graph 是 resource 的一種，我們先建立 *eta.xml*。

{{< figure src="nav-graph-file-system.png" title="eta.xml 在 project 的位置" >}}

先附上完整的內容，然後再慢慢講解入面的意思。

```xml
<?xml version="1.0" encoding="utf-8"?>
<navigation xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/eta"
    app:startDestination="@id/stationListFragment">

    <fragment
        android:id="@+id/stationListFragment"
        android:name="net.swiftzer.etademo.presentation.stationlist.StationListFragment"
        android:label="StationListFragment"
        tools:layout="@layout/station_list_fragment">
        <action
            android:id="@+id/action_stationListFragment_to_etaFragment"
            app:destination="@id/etaFragment" />
    </fragment>
    <fragment
        android:id="@+id/etaFragment"
        android:name="net.swiftzer.etademo.presentation.eta.EtaFragment"
        android:label="EtaFragment"
        tools:layout="@layout/eta_fragment">
        <argument
            android:name="line"
            app:argType="net.swiftzer.etademo.common.Line" />
        <argument
            android:name="station"
            app:argType="net.swiftzer.etademo.common.Station" />
    </fragment>
</navigation>
```

切換到 Design 後就能看到兩頁的 layout XML 預覽畫面和各頁之間的導航方向（就是兩頁之間的箭頭）。由於下圖是我在完成 `EtaFragment` 基本功能後才擷取所以內容比較豐富，如果按照上一篇來做的話應該只會看到兩頁只得 top bar 和下面一大片空白。

{{< figure src="navigation-editor.png" title="Navigation editor" >}}

`stationListFragment` 左上角有個小屋 icon，意思是那一個 navigation graph 的首頁。首頁的意思是一進入那個 navigation graph 會看到那頁，每個 navigation graph 都要指明一個 `Fragment` 做首頁。在 XML 的寫法是在 `<navigation>` tag 的 `app:startDestination` 指明那頁的 ID (`@id/stationListFragment`)。

而每頁的定義就是用 `<fragment>` tag 定義，以 `StationListFragment` 為例：

```xml
<fragment
    android:id="@+id/stationListFragment"
    android:name="net.swiftzer.etademo.presentation.stationlist.StationListFragment"
    android:label="StationListFragment"
    tools:layout="@layout/station_list_fragment">
    <action
        android:id="@+id/action_stationListFragment_to_etaFragment"
        app:destination="@id/etaFragment" />
</fragment>
```

- `android:id` 就是用來給那個 navigation 項目定義一個 ID，方便我們引用到（那個 `app:startDestination` 就是例子）
- `android:name` 是 `Fragment` 的全名
- `android:label` 跟 manifest 那個 `<activity>` 入面的 `android:label` 作用差不多，就是給那頁一個讓人看的名稱
- `tools:layout` 就是為了在 IDE 預覽時能看到 layout XML 而設，看到 namesapce 是 `tools` 就知道了，不寫那個都不會影響運行效果

至於入面的 `<action>` 就是設定這頁可以跳到另一頁，有點像 iOS Storyboard 的 segue。在這個例子我們設定它能跳到 `EtaFragment`。其實那個 `<action>` 的 `android:id="@+id/action_stationListFragment_to_etaFragment"` 是由 Design 介面拖曳 `StationListFragment` 右邊的圓形再在 `EtaFragment` 放手自動生成出來的，不用擔心要自己寫這麼長的名字。

我們轉去看另一個 `<fragment>`，它有另一款 tag 叫 `<argument>`。這個 tag 是定義開啟 `Fragment` 的 argument。傳統啟動 `Activity` 和 `Fragment` 如果要附帶參數的話都會用到 intent extras 或 arguments 傳遞 `Bundle`。即使用了 Navigation component 這亦不變，變的地方是把參數定義在 XML 內。配合 [safe args Gradle plugin](https://developer.android.com/guide/navigation/navigation-pass-data) 就可以令這個過程更 type safe。以往如果要做到 type safe 效果會寫一個 static 的 `newInstance` method 把這些建立 `Bundle` 的 code 塞進去，其他地方就不用知道那個 key 是甚麼，而且不會用錯 data type，但要每個 `Activity` 和 `Fragment` 都要做一次這個 static method。我們會在之後實際寫 `StationListFragment` 和 `EtaFragment` 時再仔細介紹。

```xml
 <fragment
    android:id="@+id/etaFragment"
    android:name="net.swiftzer.etademo.presentation.eta.EtaFragment"
    android:label="EtaFragment"
    tools:layout="@layout/eta_fragment">
    <argument
        android:name="line"
        app:argType="net.swiftzer.etademo.common.Line" />
    <argument
        android:name="station"
        app:argType="net.swiftzer.etademo.common.Station" />
</fragment>
```

## `MainActivity`

定義好 navigation graph 之後，我們要改改原本在 Android Studio 範本的 `MainActivity`：

```kotlin
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {

    private lateinit var binding: MainActivityBinding
    private lateinit var navController: NavController

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = MainActivityBinding.inflate(layoutInflater)
        setContentView(binding.root)
        val navHostFragment =
            supportFragmentManager.findFragmentById(binding.navHostFragment.id) as NavHostFragment
        navController = navHostFragment.navController
    }

    override fun onNewIntent(intent: Intent?) {
        super.onNewIntent(intent)
        navController.handleDeepLink(intent)
    }

    override fun onSupportNavigateUp(): Boolean = navController.navigateUp()
}
```

同樣地，我們會加上 `@AndroidEntryPoint`。這是因為這個 `Activity` 將會加載帶有 `@AndroidEntryPoint` 的 `Fragment`，所以即使這個 `Activity` 看起來沒有用到 Dagger Hilt 我們還是需要加 `@AndroidEntryPoint`。如果不加的話之後遇到帶有 `@AndroidEntryPoint` 的 `Fragment` app 就會 crash。雖然 Dagger 標榜是透過 compile 時檢查 DI 的設定，但並不包括這部分。

這次我們會用上 view binding，其實跟 data binding 有幾分相似，但不能 pass variable 去 layout XML 用，亦不能在 layout XML 加入 Java code。它的作用就是讓你不用再寫 `findViewById` 取得 layout XML 的 view。它比以前的 [Kotlin synthetics](https://developer.android.com/topic/libraries/view-binding/migration) 和 [Butterknife](https://github.com/JakeWharton/butterknife) 好在它能分析到那些 view ID 是不是只在個別 configuration 才會出現，如果是的話 binding 就會外露 nullable 的 view 讓你引用。

如果在 `Fragment` 用 view binding 的話同樣都需要在 `onDestroyView` 終止所有 binding 引用，好讓那些 view 能被 garbage collection。而 `Activity` 因為它的 lifecycle 跟 view 一樣，所以不用像 `Fragment` 用 view/data binding 般要在 `onDestroyView`  清除引用。（因為 `Fragment` 比那些 view 長命）

整個 `MainActivity` 只是需要做三件事：

1. inflate layout XML，入面會有一個 `FragmentContainerView` 用來顯示 navigation graph 的 `Fragment`（就是顯示每頁內容的位置）
2. 在 `onNewIntent` 時將 `Intent` 交予 `NavController` 處理轉頁（就是 app 開了後再啟動 deep link 時通知 Navigation component 轉去對應頁面，雖然這個示範 app 不會有 deep link 但還是示範給你們看）
3. 在 `onSupportNavigateUp` 時通知 `NavController` 跳到 navigation graph 的上一頁或者上一層 graph（這個是按下 action bar 的 back 按鈕時會觸發的 callback，雖然我們不是用系統的 action bar 而是用自行加入的 `AppBarLayout` 但為防日後不經意地用了原裝 action bar 所以還是加了）

下面就是 `MainActivity` 的 layout xml (`main_activity.xml`)。由於我們的 app 的 toolbar 都是由各自 `Fragment` 自行處理，所以全個 `MainActivity` 只需要放一個 `FragmentContainerView` 就可以了。留意 `android:name`、 `app:defaultNavHost` 和 `app:navGraph` 這三個 attribute。

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.fragment.app.FragmentContainerView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/navHostFragment"
    android:name="androidx.navigation.fragment.NavHostFragment"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    app:defaultNavHost="true"
    app:navGraph="@navigation/eta"
    tools:context=".MainActivity" />
```

如果打算加 deep link 的話可以在 manifest 對應 `MainActivity` 的 `<activity>` 加入 `<nav-graph>`，這樣你就不用針對每個 deep link 加上 `<intent-filter>`。

```xml
<activity
    android:name=".MainActivity"
    android:exported="true">
    <nav-graph android:value="@navigation/eta" />
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />

        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
```

完成後執行 app 應該會看到一個只有 toolbar 和空白內容的頁面。這個將會是下一篇會做的部分：車站列表。

## R8 注意事項

如果現在執行有 R8 處理過的 app 的話，一打開 app 就會 crash：

```text
2021-09-23 22:28:06.228 10900-10900/? E/AndroidRuntime: FATAL EXCEPTION: main
    Process: net.swiftzer.etademo, PID: 10900
    java.lang.RuntimeException: Unable to start activity ComponentInfo{net.swiftzer.etademo/net.swiftzer.etademo.MainActivity}: android.view.InflateException: Binary XML file line #11 in net.swiftzer.etademo:layout/main_activity: Binary XML file line #11 in net.swiftzer.etademo:layout/main_activity: Error inflating class androidx.fragment.app.FragmentContainerView
        略……
        Caused by: android.view.InflateException: Binary XML file line #11 in net.swiftzer.etademo:layout/main_activity: Binary XML file line #11 in net.swiftzer.etademo:layout/main_activity: Error inflating class androidx.fragment.app.FragmentContainerView
        Caused by: android.view.InflateException: Binary XML file line #11 in net.swiftzer.etademo:layout/main_activity: Error inflating class androidx.fragment.app.FragmentContainerView
        Caused by: java.lang.RuntimeException: Exception inflating net.swiftzer.etademo:navigation/eta line 24
            at androidx.navigation.p.c(Unknown Source:119)
            at androidx.navigation.NavController.i(:2)
            at androidx.navigation.fragment.NavHostFragment.M(:43)
            略……
        Caused by: java.lang.RuntimeException: java.lang.ClassNotFoundException: net.swiftzer.etademo.common.Line
            at androidx.navigation.p.d(:1)
            at androidx.navigation.p.b(:1)
```

原因是我們那個 navigation graph 有引用到 `net.swiftzer.etademo.common.Line` 和 `net.swiftzer.etademo.common.Station` 兩個 enum（就是那兩個 `<argument>`）。R8 會把這些 class 混淆（重新命名），所以在啟動時 Navigation component 會找不到這兩個 class。解決方法是把這兩個 enum 補上 `@Keep` annotation，這樣 R8 就不會把那些 class 混淆。

```kotlin
@Keep
enum class Line(val zh: String, val en: String) {
    AEL("機場快綫", "Airport Express"),
    TCL("東涌綫", "Tung Chung Line"),
    TML("屯馬綫", "Tuen Ma Line"),
    TKL("將軍澳綫", "Tseung Kwan O Line"),
}
```

基本上我們凡是見到 XML 檔有 class 的引用都應該要留是把那些 class 剔除在混淆範圍之內。

## 小結

Navigation component 看似把以往處理一般轉頁和 deep link 的麻煩事變得更易管理，但它是不是真的那麼好用呢？當然不是！如果簡單看過它的文檔或許會覺得它很美好，但去到實際使用時就發覺有大大小小的問題，感覺它就是一個半製成品般。我強烈建議大家看看 [Isaac Udy 演講的 Navigation in multi-module projects, and the problem with AndroidX Navigation](https://www.droidcon.com/2020/12/15/navigation-in-multi-module-projects-and-the-problem-with-androidx-navigation/)，裏面有提及當 multi-module 時使用 Navigation component 所遇到的問題和解決方法，但那些解決方法都會令原本的 Navigation component 特色削弱（例如 type safe 的 parameter 變不 type safe），所以我才說 Navigation component 似是一個半製成品。

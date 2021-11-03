---
title: "2021 iThome 鐵人賽 Day 17：Navigation (1)"
tags:
  - 2021 iThome 鐵人賽
  - Android
date: 2021-10-02T00:00:00+08:00
canonicalURL: https://ithelp.ithome.com.tw/articles/10276248
---

> 本篇文章是 [2021 iThome 鐵人賽](https://ithelp.ithome.com.tw/2021ironman)參賽題目「[寫一個列車抵站時間 Android App](https://ithelp.ithome.com.tw/users/20139666/ironman/4661)」的第 17 篇，你可到 iThome [查看原文](https://ithelp.ithome.com.tw/articles/10276248)。
>
> [文章目錄]({{< ref "2021-ithome-ironman" >}})

經過了兩個多星期後，我們終於開始進入 presentation layer 的部分。Presentation layer 就是做 UI 相關的東西，例如 `Activity`、`Fragment`、`ViewModel` 這些 class。而這次要做的部分是要準備基本的 navigation。

我們這個示範 app 會採用 single activity app 的做法，即是整個 app 只會有一個 `Activity`，所有顯示的頁面都是用 `Fragment` 來裝住。如果要由一頁轉去另一頁的話原理就是用 `FragmentManager` 切換顯示另一個 `Fragment`。不過我們不會直接接觸 `FragmentManager`，而是用 AndroidX 的 [Navigation component](https://developer.android.com/guide/navigation) 來幫我們處理。為甚麼原本能用多個 `Activity` 就做到的東西要轉用 single activity app 來做呢？主要原因是處理 deep link 的話 app 只有單一 `Activity` 是遠較多個 `Activity` 的 app 容易控制，單是 Manifest 入面 `<activity>` 的 [`android:launchMode`](https://developer.android.com/guide/topics/manifest/activity-element#lmode) 就搞到頭疼，後來更變成 Android 面試的經典題目。如果完全轉用 single activity 的做法的話基本上除了要做「一 app 多開」的效果外，基本上都不用特別處理 `android:launchMode`。「一 app 多開」的正式名稱是 task。意思是一個 app 可以在系統的「recent apps」顯示好幾個視窗，例如[以前的 Chrome 開新 tab 都是用這個功能來做到](https://www.youtube.com/watch?v=0L-uiWWPf0g)。這個功能在 Word 這類應用非常合適，配合 split screen 來用就可以同時上下顯示兩個 Word 文件並同時編輯。

說回 navigation 的部分，AndroidX 的 [Navigation component](https://developer.android.com/guide/navigation) 除了處理換頁時的 `Fragment` 切換和 deep link 之外，還有是管理每頁傳入的參數、換頁動畫，配搭 Dagger Hilt 的話更可以設定某些 object 的 scope 是跟 navigation graph 共生死。

## 安裝 Navigation component

首先在 project 的 *build.gradle* 加入 safe args Gradle plugin：

```groovy
dependencies {
    // ...
    classpath "androidx.navigation:navigation-safe-args-gradle-plugin:$navigationVersion"
}
```

然後在 `app` module 的 *build.gradle* 加入 Navigation component safe args plugin 和相關的 dependency：

```groovy
plugins {
    // ...
    id 'androidx.navigation.safeargs.kotlin'
}

dependencies {
    implementation "androidx.navigation:navigation-fragment-ktx:$navigationVersion"
    implementation "androidx.navigation:navigation-ui-ktx:$navigationVersion"
}
```

然後就可以加入我們的 navigation graph，但請同時加入以下的內容，因為本篇會用到：

```groovy
android {
    // ...
    buildFeatures {
        dataBinding true
        viewBinding true
    }
}
```

有人說 Navigation component 是 Android 的 [Storyboard](https://developer.apple.com/library/archive/documentation/ToolsLanguages/Conceptual/Xcode_Overview/DesigningwithStoryboards.html)，確實界面上跟 iOS Storyboard 真的很似。但跟 Storyboard 不同是 Android 的 navigation graph XML 只會儲存各頁對應的 `Fragment` class 名、layout XML 名、各頁的參數、deep link 和轉頁的連結等跟 navigation 相關的資訊，各頁的界面放了甚麼 `View` 仍然跟以往一樣是放在各個 layout XML 入面。而 iOS Storyboard 檔案除了存放 navigation 的資訊外，還包括各頁顯示的內容，所以頁數一多就會變得很慢。這亦都是部分 iOS developer 不用 Storyboard 而用 Interface Builder（XIB 檔）或者索性用 code 控制排版的原因。

但繼續討論 Navigation component 之前，我們要先準備 app 那兩頁的 `Fragment` class 和 layout XML 檔，否則我們開了那個 navigation graph XML 檔都不能做任何事情。

## `StationListFragment`

由於我們的 app 有兩頁，那我們就要準備兩個 `Fragment` class：`StationListFragment` 和 `EtaFragment`，分別對應車站列表和抵站時間頁。首先是 `StationListFragment`：

```kotlin
@AndroidEntryPoint
class StationListFragment : Fragment() {
    private var _binding: StationListFragmentBinding? = null
    private val binding: StationListFragmentBinding get() = _binding!!

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        _binding = StationListFragmentBinding.inflate(inflater, container, false)
        binding.lifecycleOwner = viewLifecycleOwner
        return binding.root
    }

    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null
    }
}
```

首先頂頭會有個 `@AndroidEntryPoint`，這個是 Dagger Hilt 的 annotation。加了它 Dagger Hilt 就會自動替你做 dependency injection（包括 field injection）。現在不明白不要緊，我們會在之後再詳細講解。

我們大部分 layout XML 都會採用 data binding，所以 `StationListFragment` 會有 `StationListFragmentBinding`。在 `onCreateView` 我們會 inflate XML，但因為用了 data binding 所以寫法會有小許不同。那句 `binding.lifecycleOwner = viewLifecycleOwner` 的意思是那個 data binding 用到的 `LiveData` 和 `StateFlow` 會按照 `StationListFragment` 的 lifecycle 控制何時開始和終結 observation。詳細用法要先賣個關子，因為我們這篇的主要目的是弄好兩頁的殼來準備 navigation graph。

你會看到我們準備了兩個 `StationListFragmentBinding` 的 property，一個是 nullable 一個就不是 nullable。而在 `onDestroyView` 我們把 `_binding` 設回 null 是[為了防止 memory leak](https://developer.android.com/topic/libraries/view-binding#fragments) 和 view 已經從 `Activity` 中移除，不應再 reference 住它。但因為把 view binding 或 data binding 的 property 弄成 nullable 在使用上不夠方便，所以就出現了另一個不是 nullable 的 property 來把 `_binding` 強行 access。

而 layout XML 我們叫它做 *station_list_fragment.xml*，這是 `StationListFragmentBinding` 命稱的來源（它會自動把命稱轉為 camel case 並加上後綴 `Binding`）。

```xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools">

    <data>

    </data>

    <androidx.coordinatorlayout.widget.CoordinatorLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <com.google.android.material.appbar.AppBarLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content">

            <com.google.android.material.appbar.MaterialToolbar
                android:id="@+id/topAppBar"
                style="@style/Widget.MaterialComponents.Toolbar.Primary"
                android:layout_width="match_parent"
                android:layout_height="?attr/actionBarSize"
                app:title="@string/app_name" />
        </com.google.android.material.appbar.AppBarLayout>

        <androidx.recyclerview.widget.RecyclerView
            android:id="@+id/recyclerView"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            app:layout_behavior="@string/appbar_scrolling_view_behavior" />
    </androidx.coordinatorlayout.widget.CoordinatorLayout>
</layout>
```

## `EtaFragment`

另一頁要準備的是 `EtaFragment`，內容都是跟之前差不多，只是換了個名字。

```kotlin
@AndroidEntryPoint
class EtaFragment : Fragment() {
    private var _binding: EtaFragmentBinding? = null
    private val binding: EtaFragmentBinding get() = _binding!!

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        _binding = EtaFragmentBinding.inflate(inflater, container, false)
        binding.lifecycleOwner = viewLifecycleOwner
        return binding.root
    }

    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null
    }
}
```

對應的 layout XML 是 *eta_fragment*：

```xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto">

    <data>

    </data>

    <androidx.coordinatorlayout.widget.CoordinatorLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <com.google.android.material.appbar.AppBarLayout
            android:layout_width="match_parent"
            android:layout_height="wrap_content">

            <com.google.android.material.appbar.MaterialToolbar
                android:id="@+id/topAppBar"
                style="@style/Widget.MaterialComponents.Toolbar.Primary"
                android:layout_width="match_parent"
                android:layout_height="?attr/actionBarSize"
                app:title="@string/app_name" />
        </com.google.android.material.appbar.AppBarLayout>

        <androidx.recyclerview.widget.RecyclerView
            android:id="@+id/recyclerView"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            app:layout_behavior="@string/appbar_scrolling_view_behavior" />
    </androidx.coordinatorlayout.widget.CoordinatorLayout>
</layout>
```

## 改 theme

從那兩個 layout XML 看到我們每頁都會有自己的 top bar (`AppBarLayout`)，所以 theme 就不應該用帶有 top bar（以前叫 action bar）的 theme。

Project 本身是用 Android Studio 的 template 建立，所以會有 *themes.xml*：

```xml
<resources xmlns:tools="http://schemas.android.com/tools">
    <!-- Base application theme. -->
    <style name="Theme.ETADemo" parent="Theme.MaterialComponents.DayNight.DarkActionBar">
        <!-- 略 -->
    </style>
</resources>
```

我們要把 `Theme.MaterialComponents.DayNight.DarkActionBar` 換成無 action bar 的 theme：

```xml
<resources xmlns:tools="http://schemas.android.com/tools">
    <!-- Base application theme. -->
    <style name="Theme.ETADemo" parent="Theme.MaterialComponents.DayNight.NoActionBar">
        <!-- 略 -->
    </style>
</resources>
```

這篇的 code 有點長，我們下一篇會寫 navigation graph 的部分和把 `MainActivity` 改成顯示那個 navigation graph 的頁面。

## 參考

- [Android 面试黑洞——当我按下 Home 键再切回来，会发生什么？](https://www.youtube.com/watch?v=r4T9zkhpmII)

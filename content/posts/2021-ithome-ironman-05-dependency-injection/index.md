---
title: "2021 iThome 鐵人賽 Day 5：Dependency injection"
tags:
  - 2021 iThome 鐵人賽
  - Android
date: 2021-09-20T00:00:00+08:00
canonicalURL: https://ithelp.ithome.com.tw/articles/10268173
---

> 本篇文章是 [2021 iThome 鐵人賽](https://ithelp.ithome.com.tw/2021ironman)參賽題目「[寫一個列車抵站時間 Android App](https://ithelp.ithome.com.tw/users/20139666/ironman/4661)」的第 5 篇，你可到 iThome [查看原文](https://ithelp.ithome.com.tw/articles/10268173)。
>
> [文章目錄]({{< ref "2021-ithome-ironman" >}})

談到 Android 的 dependency injection (DI)，大家一定會想到 Dagger 這個 DI library。因為 Dagger 2 是由 Google 開發，加上在 Android Developers 網站有對應的教學文章，所以成為不二之選。不過 Dagger 向來都有難學的印象，尤其是在以前 Dagger 2 網站那個惡名昭彰的[咖啡機教學](https://dagger.dev/dev-guide/)。本來用咖啡機來說明 DI 的概念就沒有甚麼大問題，問題在於看完那個咖啡機類比是不會知道如何在 Android app 上套用那些概念和 Dagger 提供的功能。或許大家讀電腦科學／計算機科學／資訊工程之類的課程時都有學過 DI 但還是不能順利地用 Dagger，這是因為 Android 那些組件的 constructor 不是由我們去 call，結果要在 `onCreate` 之類的 callback 加上 DI library 要我們寫的 code 才能拿到我們要用的 dependency。

Google 曾經推出過 Dagger Android 的 library，就是方便在 Android 開發時使用 Dagger。留意 Dagger 本身並不是只為 Android 設計，它本來是一個通用的 DI library。但推出後[有些人發覺它甚至比原裝 Dagger 更難用](https://www.reddit.com/r/androiddev/comments/dp93j3/daggerandroid_is_dead_long_live_dagger_in_android/)。所以他們做了一個替代品：Dagger Hilt。其實 Dagger Hilt 就是把 Dagger 在 Android 的用法標準化，本來在 Dagger 一些由我們去定義的東西在 Dagger Hilt 變成由 Hilt 提供，我們要按照 Hilt 規定的方式做 DI。其中一個例子是 component 改為由 Hilt 按 Android 各部件的 lifecycle 預先安排好的 component。再加上 Hilt 的 Gradle plugin 和 Android Jetpack library 的配合，這樣就可以將平時用 Dagger 要寫的一大堆 boilerplate code 大大減少，而且在進入新的開發團隊時不用再浪費時間理解那些 `AppComponent`、`@Singleton` 的實際意義。

現在討論 Dagger Hilt 沒有實際 code 做示範，感覺比較虛無，我們會在之後用到 Dagger Hilt 時再作講解。在結束本篇之前，相信大家心入面都會問一個問題：為甚麼要用 DI library。或許有不少人都是為用而用，或者是因為面試會問到所以要學習這些 library。我覺得最易理解為甚麼要用 DI 就是試試找一個沒有寫過單元測試 (unit test) 的項目把一些現有的 class 為它們寫單元測試，當你發現困難重重時（例如用了太多 singleton、很多地方寫死了在 unit test 很難換走導致很多情景無法製造出來）就會開始明白為甚麼要用 DI library。不過不用擔心，這次的示範 app 本身就有考慮到 unit testing 來寫的，我們會在下一篇開始用 Dagger Hilt。

## 準備工作

在最頂層的 *build.gradle* 和 app module 的 *build.gradle* 加上 Dagger Hilt 的 Gradle plugin 和 Dagger Hilt 相關的 dependency。

```groovy
// root level build.gradle
buildscript {
    ext.daggerVersion = '2.38.1'
    repositories {
        google()
        mavenCentral()
    }
    dependencies {
        // ...
        classpath "com.google.dagger:hilt-android-gradle-plugin:$daggerVersion"
    }
}
```

```groovy
// app module build.gradle
implementation "com.google.dagger:hilt-android:$daggerVersion"
kapt "com.google.dagger:hilt-compiler:$daggerVersion"
testImplementation "com.google.dagger:hilt-android-testing:$daggerVersion"
testAnnotationProcessor "com.google.dagger:hilt-compiler:$daggerVersion"
androidTestImplementation "com.google.dagger:hilt-android-testing:$daggerVersion"
androidTestAnnotationProcessor "com.google.dagger:hilt-compiler:$daggerVersion"
```

然後為自行 extend 出來的 `Application` 加上 Dagger Hilt 的 annotation。不要忘記在 manifest 的 `application` tag 加上 `name` attribute。

```kotlin
@HiltAndroidApp
class EtaDemoApp : Application() {
    override fun onCreate() {
        super.onCreate()
    }
}
```

## 小結

- Dagger Hilt 是建基於 Dagger 的 library，它就是為 Android 開發時用 Dagger 提供了一個固定的範式
- Dagger Hilt 有 Gradle plugin 能減少平常用 Dagger 時所寫的 boilerplate code，加上其他 Jetpack library 的配套令我們用 Dagger 更自然（令人感覺上更似 dependency injection 而不是 [service locator](https://www.reddit.com/r/androiddev/comments/d1b4i1/understanding_the_difference_between_di_and_sl/)）
- 用 DI library 很大部分的原因是為了方便寫 unit test 時能把依賴的組件換成其他東西，這樣就能測試到不同的場景

## 補充資料

- [Dependency Injection: Comparing Dagger, Koin and Hilt - Maia Grotepass](https://www.youtube.com/watch?v=-MJm4UGoh5I)
- [Understanding the difference between DI and SL](https://www.reddit.com/r/androiddev/comments/d1b4i1/understanding_the_difference_between_di_and_sl/)

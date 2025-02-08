---
title: "Amper Android manifest maxSdkVersion"
date: 2025-02-08T15:00:00+08:00
tags:
  - Android
---

最近發現原來 [MetroRide](https://play.google.com/store/apps/details?id=net.swiftzer.metroride) 在 Android 15 的裝置上找不到，經 Google 網頁搜尋打開 Google Play 看到紅字說裝置不支援。後來才發現原來 Android manifest 被加上 `android:maxSdkVersion="34"`。

在 IDE 看 app 的 merged manifest 找不到被加了這句，要從 build 出來的 Android app bundle 的 manifest 才會找到這句。就算在 *build.gradle.kts* 加上 `maxSdk = null` 亦不能移除：

```kotlin
android {
    defaultConfig {
        maxSdk = null
    }
}
```

於是翻查 Git history 就把上一個版本更新過的 library 檢查一次，順便更新一下 project 的 dependency。最後發現原因是因為上年九月的 app release 改用 [Ksoup](https://github.com/fleeksoft/ksoup) 來抽取 HTML 內容而導致 app 被加了這句 `maxSdkVersion`。

大概原因是因為 Ksoup 用了 JetBrains 出的 [Amper](https://github.com/JetBrains/amper) 來設定這個 project，但舊版本的 Amper 有一個 bug ([AMPER-813](https://youtrack.jetbrains.com/issue/AMPER-813)) 是它會幫 Android manifest 加上 `android:maxSdkVersion="34"`，現在 Amper 已經修正了這個問題，而新版 Ksoup 亦用了新版 Amper。所以只要更新 Ksoup 版本就能解決這個問題。

其實在上架的時候 Goole Play console 會顯示上一個版本和下一個版本的差異（類似下圖），它有標明 API levels 是 26-34 而不是 26+，只是我那時沒有特別留意就讓它送審。所以大家在發佈新版前要留意一下那些權限、API level 之類有沒有突然被改動。

{{< figure src="aab-diff.webp" title="比較兩個 Android app bundle 的差異" >}}

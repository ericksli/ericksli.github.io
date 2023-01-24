---
title: Timber live template for Java/Kotlin
tags:
  - Android
  - Java
  - Kotlin
date: 2017-05-22T22:53:34+08:00
---


最近轉了用 [Kotlin](https://kotlinlang.org/) 來寫自己的 Android app，但發現 Android Studio 在 Kotlin 檔案內無法使用 Logcat `logd`、`logm` 之類的 [Live template](https://www.jetbrains.com/help/idea/using-live-templates.html)。[Anko](https://github.com/Kotlin/anko) 的 `AnkoLogger` 因為用了 [`Log.isLoggable`](<https://developer.android.com/reference/android/util/Log.html#isLoggable(java.lang.String, int)>) 來包住 `Log.d` 之類的 method 所以在開發時看 log 不夠方便。於是就轉了用 [Timber](https://github.com/JakeWharton/timber) 來做 logging。但是轉了 logging library 都是沒有方便的方法來產生 log message。所以最後我參考了 Android Studio 的 log live template 來做了適用於 Java 和 Kotlin 的 Timber live template。

{{< figure src="timber.gif" title="Live template 示範" >}}

Live template 我已經放到 [Gist](https://gist.github.com/ericksli/1afdfb1590e2cefb33c415b4f03bf645)，是兩個 XML 檔來的。一個是 Java 版一個是 Kotlin 版。大致上和原裝的 live template 相似，只是將 `log` 改成 `tim`。例如 `timd` 會生成 `Timber.d`。

安裝 live template 的方法是把這些 XML 檔案放到 [IDE 的設定目錄](https://www.jetbrains.com/help/idea/directories-used-by-the-ide-to-store-settings-caches-plugins-and-logs.html#config-directory)內的 `templates` 目錄，不同作業系統的設定目錄位置都會不同。如果找不到 `templates` 目錄就自己建立一個。把檔案放到指定位置後重新開啟 IDE 便會生效。

---

**2020 年 8 月 18 日更新：** 如果是用 macOS 而又用 Android Studio 的話，安裝的路徑改為 `/Users/xxx/Library/Application Support/Google/AndroidStudioPreview4.2/templates`（將 `AndroidStudioPreview4.2` 換成你使用的版本）。

**2023 年 1 月 24 日更新：** Windows 版 Android Studio Electric Eel (2022.1.1) 的安裝路徑是 `%APPDATA%\Google\AndroidStudio2022.1\templates`，其他版本的路徑則需改變 `AndroidStudio2022.1` 的部分。
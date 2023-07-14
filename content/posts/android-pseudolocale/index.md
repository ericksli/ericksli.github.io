---
title: "Android pseudolocale"
date: 2023-07-14T23:40:00+08:00
isCJKLanguage: true
tags:
  - Android
---

在處理 app UI 多國語言時，我們不時要留意是不是預留了足夠空間來顯示文字。一般而言，中文內容通常都比其他語言短，你的 UI 可能看起來沒有問題，但換到其他寫得比較長的語言就可能不夠位顯示。但在開發初期可能還未開始翻譯，只有英文版，未必能在早期察覺這個問題。

Android SDK 其實有一個功能叫 pseudolocale，字面意思是假的地區設定。Android emulator 內置支援兩款 pseudolocale：`en-XA` 和 `ar-XB`，分別對應左至右和右至左語系。兩款 pseudolocale 都是用你的 app 的預設 string resource 加工而成。

XA 就是把預設的 string resource 的英文字母換成有變音符號的字母再在最尾加上 `one two three` 之類的文字去加長整段字，作用是為了模擬 app UI 在較長的左至右語言的情況，看看會不會塞爆 UI；而 XB 就是把預設的 string resource 的英文字母反轉次序，同時亦會令界面變成右至左模式，這樣就可以模擬 app UI 在右至左語言的情況。由於兩個 pseudolocale 都是由預設的 string resource 衍生出來（理論是預設是用來放英文），所以加工後你都應該能看得懂，不會因為切換了 pseudolocale 而不懂得操作自己的 app。

要啟用 pseudolocale，首先要在 project 的 *build.gradle.kts* 加入 `resourceConfigurations` 和在 build type 加入 `isPseudoLocalesEnabled = true`。

```kotlin
android {
    defaultConfig {
        applicationId = "net.swiftzer.metroride"
        versionCode = 37
        versionName = "0.37.0"
        // 限制 resource 的 locale 並加入 pseudolocale
        resourceConfigurations.addAll(listOf("en", "zh-rHK", "en-rXA", "ar-rXB"))
    }
    
    buildTypes {
        getByName("debug") {
            isDefault = true
            isDebuggable = true
            // 啟用 pseudolocale
            isPseudoLocalesEnabled = true
        }
    }
}
```

上面的 code 除了加了 `en-rXA` 和 `ar-rXB` 外，還加了 `en` 和 `zh-rHK`。這是因為我的 app 只是準備了預設（英文）和香港中文。如果沒有加 `resourceConfigurations` 的話，你的 app 通常都會被加了一大堆不同語言的 resource 檔案（例如文字和圖片）。這是因為 appcompat 和其他 Android library 都會附帶了不同語言的 resource 檔案。如果你的 app 本身只支援數種語言，大可把其餘不支援的語言 resource 檔案在 build app 時清走，這樣就可以減少最終 app 的大小，而且如果你打算在 runtime 強行改變 app 語言的話這個設定是跑不掉的。

{{< figure src="apk-resources.png" title="加了 resourceConfigurations 限制後只剩下 default 和 zh-rHK" >}}

加了之後就轉去 emulator 設定。首先要啟用 Developer options，就是連按七次 Build number，直至看到「You are now a developer!」的 toast。**啟用後要將 emulator 重新開機。**

之後就可以在平時設定系統語言的地方看到 XA 和 XB 的語言。把它們調到最前，然後 build app 並開啟 app 就能看到以 pseudolocale 形式顯示的 UI。這樣就可以開始逐頁檢查排版有沒有問題。

{{< figure src="settings.png" title="Android 設定內的語言設定頁" >}}

{{< figure src="xa.png" title="在 XA 的情況" >}}

{{< figure src="xb.png" title="在 XB 的情況" >}}

當然，如果你的 app 本身有強行改 app locale 的話應該不能馬上就可以顯示到 pseudolocale，你可以參考下面的 code 來檢查當前系統 locale 來決定如果設定 app locale：

```kotlin
val systemLocale = ConfigurationCompat
    .getLocales(Resources.getSystem().configuration)[0]!!
when {
    systemLocale.language == "en" && systemLocale.country == "XA" -> {
        // 這是 XA pseudolocale
    }
    systemLocale.language == "ar" && systemLocale.country == "XB" -> {
        // 這是 XB pseudolocale
    }
}
```

試完後，緊記在 release build 拿走 pseudolocale，否則上載到 Google Play Console 會覺得你的 app 會支援阿拉伯文。（除非你的 app 本身真的是支援阿拉伯文）

```kotlin
android {
    buildTypes {
        getByName("release") {
            isPseudoLocalesEnabled = false
            defaultConfig {
                resourceConfigurations.removeAll(setOf("en-rXA", "ar-rXB"))
            }
        }
    }
}
```

## 不應改變的 string resource

由於 pseudolocale 會改變你的 string resource 內容，如果某些 string resource 被改動的話會導致 app crash，令你不能看到 UI。這種情況通常會在日期時間格式出現。要避免因為開了 pseudolocale 而令到 app crash，可以在相關的 string resource 加入 `translatable="false"`，這樣在開了 pseudolocale 後這些 string resource 都不會被改變。

```xml
<string name="mtr_alert_timestamp_format_12hr" translatable="false">E, dd MMM h:mm a (zzz)</string>
```

在 Android Studio 會警告你翻譯了不應翻譯的 string resource，但不影響 build app。

## 參考

- [Test your app with pseudolocales](https://developer.android.com/guide/topics/resources/pseudolocales)
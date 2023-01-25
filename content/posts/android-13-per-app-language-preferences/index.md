---
title: "Android 13 Per-app Language Preferences"
tags:
  - Android
date: 2023-01-25T14:27:00+08:00
---

最近抽點時間把 [MetroRide](https://play.google.com/store/apps/details?id=net.swiftzer.metroride) 參照 [Now in Android](https://github.com/android/nowinandroid) 示範項目更新一下，例如改用 TOML 版的 [version catalog](https://docs.gradle.org/current/userguide/platforms.html)（之前是用 Kotlin DSL）、轉用 `includeBuild` 加 convention plugin 取代之前把 plugin 放在 `buildSrc` 內、更新 dependency 版本和把 target SDK 升到最新（即是 Android 13；API level 33）。這次想分享的是適配 Android 13 的 per-app language preferences（個別應用程式語言偏好）功能。

{{< figure src="app-language-setting.png" title="App 內置的語言設定" >}}

<!-- more -->

這個功能相信大家都期待已久，尤其是香港的 app developer。可能以前 Google 那邊的人不太明白為甚麼要有這個功能，因為通常系統語言跟個別 app 語言都是一致。然而，香港用戶會有特別要求。例如不少用戶都把系統語言設定成英文、日文之類，但去到個別 app 例如新聞、地圖那類 app 又想用中文顯示（即使 app 界面不是用中文但仍想文章、地名之類用中文）。

{{< youtube DUKnNWwcNvo >}}

這個功能的用法是在系統設定內選取系統 > 語言及輸入 > 應用程式語言中可以針對不同的 app 個別設置語言，不用跟系統語言。而同樣設定亦可在系統設定中的個別 app 的設定頁找到（應用程式 > 你的 app > 語言）。

這次官方介紹影片和[網頁](https://developer.android.com/guide/topics/resources/app-languages)都講得夠清楚，有提及適配步驟和邊緣案例。大致上就是加 [Appcompat](https://developer.android.com/jetpack/androidx/releases/appcompat) library，在 manifest 加上要支援的 locale 和 `AppLocalesMetadataHolderService`，最後就是用 Appcompat 的 `getApplicationLocales` 和 `setApplicationLocales` 去讀取和設定語言。

{{< figure src="app-languages.png" title="系統設定內的「應用程式語言」頁" >}}

看起上來似乎很簡單，但實際上又不是。有部分東西 Android developers 沒有寫到。

## 基本適配方法

首先在在 XML resource directory (`src/main/res/xml`) 建立 `locale_config.xml` 指明你的 app 支援那些語言：

```xml
<?xml version="1.0" encoding="utf-8"?>
<locale-config xmlns:android="http://schemas.android.com/apk/res/android">
    <locale android:name="en" />
    <locale android:name="zh-Hant-HK" />
</locale-config>
```

`zh-Hant-HK` 是香港繁體中文。介紹 [per-app language preferences 那頁](https://developer.android.com/guide/topics/resources/app-languages)其實有同時出現過 `zh-Hant-MO` 和 `zh-HK` 式樣，我是用 `zh-Hant-HK` 而不是 `zh-HK`。其實在 Android 7 開始就[支援 `zh-Hant-HK` 那種 locale 式樣](https://developer.android.com/guide/topics/resources/multilingual-support#postN)（如果是 resource directory 就會是 `values-b+zh+Hant+HK`）。

然後在 manifest 加上 `android:localeConfig="@xml/locale_config"`。

```xml
<application
    android:name=".app.MetroRideApplication"
    android:allowBackup="true"
    android:dataExtractionRules="@xml/data_extraction_rules"
    android:fullBackupContent="@xml/backup_rules"
    android:icon="@mipmap/ic_launcher"
    android:label="@string/app_name"
    android:networkSecurityConfig="@xml/network_security_config"
    android:localeConfig="@xml/locale_config"
    android:roundIcon="@mipmap/ic_launcher_round"
    android:supportsRtl="true"
    android:theme="@style/Theme.MyApp"
    tools:targetApi="tiramisu">
</application>
```

之後在 *build.gradle.kts* 或 *build.gradle* 加上 `resourceConfigurations`（以 Kotlin 示範）：

```kotlin
android {
    defaultConfig {
        applicationId = "net.swiftzer.metroride"
        resourceConfigurations.addAll(listOf("en", "zh-rHK"))
    }
}
```

`resourceConfigurations` 的作用是用來在 build app 時移除不支援的語言 resource。如果以前有做過強行更改 app 語言的話相信都有做這同樣設定。（`en` 其實不加也沒所謂）因為 AndroidX 的 library 會附有多國語言，如果不移除多餘的 resource 就會干擾強行更改 app 語言的效果，去到現在就算 Android 原生支援切換 app 語言也要做這個設定。

之後回到 manifest 加上 `AppLocalesMetadataHolderService`：

```xml
<service
    android:name="androidx.appcompat.app.AppLocalesMetadataHolderService"
    android:enabled="false"
    android:exported="false">
    <meta-data
        android:name="autoStoreLocales"
        android:value="true" />
</service>
```

`autoStoreLocales` 定成 `true` 並將 `android:enabled` 設成 `false` 意思是把 app 語言設定值交由 Appcompat 儲存。如果你本身有用其他方式儲存 app 語言設定值的話（例如 shared preferences 或 DataStore）可以把這個交由 Appcompat 管理。Android developers 文檔亦建議這樣做，因為系統內置的備份功能會自動備份這個設定。

另外，因應把設定交托到 Appcompat 管理，你可能需要寫遷移的 code 把本來的設定值帶到去 Appcompat。我因為懶所以沒有做到。

來到這裏其實大致上可以用了。如果你本身有做到強行切換 locale 的話就要把之前寫的 code 拿走（即是改 base context 之類的 code）。

要取得當前 app 語言設定，只需要用 `AppCompatDelegate.getApplicationLocales()` 就能拿到 `LocaleListCompat`。如果 `LocaleListCompat` 內容是空的話就表示用戶選了系統預設。如果不是空的話可以用 `toLanguageTags()` 查到（例如選了香港繁中就會是 `zh-Hant-HK`；加拿大英文就會是 `en-CA`）。

如果是設定 app 語言設定就要用 `setApplicationLocales()`，參數要傳入一個 `LocaleListCompat`。如果想設成香港繁中就用 `LocaleListCompat.forLanguageTags("zh-Hant-HK")`；如果想設成系統預設就用 `LocaleListCompat.getEmptyLocaleList()`。

留意一點是 `getApplicationLocales()` 和 `setApplicationLocales()` 都不可早於 `Activity.onCreate()` 時 call。

只需要把先前的語言設定 UI 改為使用 `AppCompatDelegate` 的 method 就基本上完成適配。

## 獲取當前系統 locale

如果你的 app 內所有 UI 文字都是由 string resource 提供的話那就真的是完成，但其實適配沒那麼簡單！因為 MetroRide 有部分文字是由 SQLite database 提供，亦有部分是由 backend 提供，backend 甚至會分中英文版會有不同路徑或者參數。如果用戶切換語言亦都需要一併更新這些內容。

在未有 Android 13 的 per-app language preferences 功能前 MetroRide 就做了一個 `SharedFlow` 來提供當前顯示語言的設定值。那些並非 string resource 提供的文字還有設定值都是由那個 `SharedFlow` 獲取設定值和觸發更新。大致上就是開一個 class 入面有一個 private 的 `MutableSharedFlow` 記錄當前語言設定值，然後外露一個 `Flow` 讓其他 class 可以 observe 它。留意要用 dependency injection library 把這個 class 設成 singleton（MetroRide 是用 Dagger Hilt），不然就會更新了另一個 `MutableSharedFlow` 或者是 observe 了另一個 `MutableSharedFlow`。

做那個 `Flow` 就要考慮到更新 `MutableSharedFlow` 的問題，就是準備兩個 method 分別觸發 `getApplicationLocales()`（我把它放到 `fetchAppLocale()`）和 `setApplicationLocales()`，順帶亦更新 `MutableSharedFlow` 的值（我把它放到 `updateLocale()`）。

### 觸發 `getApplicationLocales` 的時機

首先要在 launcher activity 的 `onCreate` 觸發（因為它規定要在 `Activity.onCreate()` 後才能 call，所以不可能放到 `Application.onCreate()` 以至用 [App Startup](https://developer.android.com/topic/libraries/app-startup)）。這就可以在開 app 時把當前的值塞進 `MutableSharedFlow`。

另一個觸發時機是 `LOCALE_CHANGED` broadcast。如果用戶在系統設定切換語言，不論是系統語言還是這個 app 的語言都會有 `LOCALE_CHANGED` broadcast。由於用戶可以隨時去系統設定切換語言，所以要做一個 `BroadcastReceiver` 去觸發 `getApplicationLocales`。

```xml
<receiver
    android:name=".app.locale.LocaleChangedBroadcastReceiver"
    android:exported="false">
    <intent-filter>
        <action android:name="android.intent.action.LOCALE_CHANGED" />
    </intent-filter>
</receiver>
```

在 `LocaleChangedBroadcastReceiver.onReceive()` 內就是觸發 `fetchAppLocale()`。

上一部分提及過用 `getApplicationLocales()` 拿到設定值，但沒有提過如果拿到空的 `LocaleListCompat` 時（用戶選了系統預設）要怎樣判斷實際上要顯示甚麼語言。這個部分其實用下面那一句就行了：

```kotlin
ConfigurationCompat.getLocales(Resources.getSystem().configuration)[0]
```

完整的 code 可以參考 [Gist](https://gist.github.com/ericksli/f5d4047c920de4a510fe0cce581a8d85)。

## Locale fallback 問題

以下是 MetroRide 沿用的 locale fallback 處理方法：

1. String resource 只做了預設（英文）和香港繁體中文 (`zh-rHK`) 兩款
2. App 內的語言設定頁可以讓用戶選擇系統預設、英文和中文
   - 系統預設即是如果系統語言設定選用了中文不論是繁中還是簡中全部指去用香港繁體中文 (`zh-HK`)，其他語言則用英文 (`en`)

以往的做法是硬改 `ContextWrapper` 強行設定用 `zh-HK` 來達至簡中 fallback 成繁中。但用了 `LocaleManager` 就做不到簡中 fallback 去繁中，[因為繁中和簡中是兩種 script，所以不會這樣 fallback](https://youtu.be/Tq7TSUzAGm8?t=71)。所以現在就放棄了簡中 fallback 成繁中，由它顯示英文就算了。如果你的 app 有這個需求而又有用翻譯平台的話可以考慮在匯出 XML 檔時把繁中翻譯同時匯出成簡中 XML 檔案。但因為我沒有用翻譯平台，更改 string resource 都是人手直接改所以就不想把同一個繁中 XML 檔同步成幾個 XML 檔。

## 打開系統設定中的個別 app 語言設定頁

如果是 Android 13 的話，他們沒有硬性規定改變 app 語言必須透過系統設定進行，你仍可以用自製的 UI。只需要確保兩邊設定同步就可以了。如果你有需要打開系統設定中的個別 app 語言設定頁的話，可以參考以下的 code snippet：

```kotlin
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
    context.startActivity(
        Intent(Settings.ACTION_APP_LOCALE_SETTINGS).apply {
            data = Uri.fromParts("package", context.packageName, null)
        }
    )
}
```

{{< figure src="system-settings-vs-app-ui.png" title="系統設定內的「應用程式語言」頁和 app 本身的設定頁" >}}

## 舊 Android 版本未能成功切換 app 界面語言

MetroRide 本身支援 Android 5 或以上，正當我以為已經試完沒有問題的時候多手試試 API level 21 時效果如何。試過後發現原來經 app 內的 UI 切換語言後 UI 的 string resource 沒有切換成功。然後 Compose UI 有部分都爛掉。可能 Appcompat 只支援 Android 7 或以上。（因為系統語言排序是 Android 7 才開始有，而那些強行改 context wrapper 的 code 都是會特別針對 Android 7 或以上做處理。我沒有 API level 21 至 26 之間逐個去試。）

所以最後決定把 `minSdk` 升到 API 26 (Android 8.0)，反正都沒有用戶用這麼舊的 Android 版本。

另一個有可能出事的位置是 `WebView`。[因為 `WebView` 內的 Google Chrome 會干擾 app locale](https://gist.github.com/amake/0ac7724681ac1c178c6f95a5b09f03ce#runtime-contamination)。我沒有特意去試這個位置，因為我用的是 [Custom Tabs](https://developer.chrome.com/docs/android/custom-tabs/) 而不是 `WebView`。

## 小結

其實這個功能都算易用，麻煩的地方主要是因為設定值要交由系統管理而不是直接由自己管理，所以就要加 broadcast receiver 之類的 code。好處方面很明顯就是 Appcompat 幫你做了以往那些 `attachBaseContext` 的東西不用再自己做，令 code 乾淨了不少。而新 API 用法都算簡單，跟之前切換 night mode 的 [`setDefaultNightMode()`](https://developer.android.com/reference/androidx/appcompat/app/AppCompatDelegate#setDefaultNightMode(int)) 有點像。

## 參考

- [Per-app language preferences](https://developer.android.com/guide/topics/resources/app-languages)
- [Language and locale resolution overview](https://developer.android.com/guide/topics/resources/multilingual-support)
- [Building for a multilingual world](https://www.youtube.com/watch?v=Tq7TSUzAGm8)
- [Correct localization on Android 7](https://gist.github.com/amake/0ac7724681ac1c178c6f95a5b09f03ce)

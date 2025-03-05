---
title: Flipper 在 Android 15 裝置上無法啟動 app
date: 2025-03-05T12:40:00+08:00
tags:
- Android
- React Native
---

[Flipper](https://fbflipper.com/) 是 Facebook 出品的 React Native debugger（不是 React Native 的 app 都可以用），前身是 [Stetho](https://facebook.github.io/stetho/)，[之前都有介紹過]({{< ref "2021-ithome-ironman-14-flipper" >}})。界面由 Stetho 時期借用 Chrome Developer Tools 到 Flipper 有 Electron desktop app 再演變到 localhost 的 web app，功能都是讓你查看 logcat、HTTP request/response、shared preferences、SQLite database、UI layout 等等。

如果在 app target Android 15 後用 Android 15 的裝置一打開 app 就出現下面那種 crash 的話，不用擔心，只需要升級一下 dependency 就可以了。

```text
E  FATAL EXCEPTION: FlipperEventBaseThread
   Process: net.swiftzer.metroride.dev, PID: 8402
   java.lang.UnsatisfiedLinkError: dlopen failed: empty/missing DT_STRTAB in "/data/app/~~1297Du5FiuE_QM7YrEF_zw==/net.swiftzer.metroride.dev-tu0kr-SEfZDkaugyqnDnng==/base.apk!/lib/x86_64/libfbjni.so"
       at java.lang.Runtime.loadLibrary0(Runtime.java:1081)
       at java.lang.Runtime.loadLibrary0(Runtime.java:1003)
       at java.lang.System.loadLibrary(System.java:1765)
       at com.facebook.soloader.nativeloader.SystemDelegate.loadLibrary(SystemDelegate.java:24)
       at com.facebook.soloader.nativeloader.NativeLoader.loadLibrary(NativeLoader.java:52)
       at com.facebook.soloader.nativeloader.NativeLoader.loadLibrary(NativeLoader.java:30)
       at com.facebook.jni.HybridData.<clinit>(HybridData.java:34)
       at com.facebook.flipper.android.FlipperThread.run(FlipperThread.java:25)
E  FATAL EXCEPTION: FlipperConnectionThread
   Process: net.swiftzer.metroride.dev, PID: 8402
   java.lang.NoClassDefFoundError: <clinit> failed for class com.facebook.flipper.android.EventBase; see exception in other thread
       at com.facebook.flipper.android.FlipperThread.run(FlipperThread.java:25)
```

[`com.facebook.flipper:flipper`](https://mvnrepository.com/artifact/com.facebook.flipper/flipper) 在 2024 年 11 月 21 日推出了最後一個版本 [0.273.0](https://mvnrepository.com/artifact/com.facebook.flipper/flipper/0.273.0)。這個版本的 Maven POM 依賴了 [`com.facebook.fbjni:fbjni:0.3.0`](https://mvnrepository.com/artifact/com.facebook.fbjni/fbjni/0.3.0) 和 [`com.facebook.soloader:soloader:0.10.4`](https://mvnrepository.com/artifact/com.facebook.soloader/soloader/0.10.4)。上面那個 stacktrace 就是因為 libfbjni.so 這個 native library 沒有因應 Android 15 改用 16 KB page size 而重新 build 過導致。要解決這個問題，只需要另外在 build.gradle 或 build.gradle.kts 特別聲明用最新版的 [`com.facebook.fbjni:fbjni:0.7.0`](https://mvnrepository.com/artifact/com.facebook.fbjni/fbjni/0.7.0) 和 [`com.facebook.soloader:soloader:0.12.1`](https://mvnrepository.com/artifact/com.facebook.soloader/soloader/0.12.1) 就可以了。因為[新版的 fbjni](https://github.com/facebookincubator/fbjni/releases/tag/v0.7.0) 已經用 NDK 27 重新 build 過，而 [React Native 都依賴 fbjni](https://mvnrepository.com/artifact/com.facebook.react/react-android/0.78.0)，所以應該不用太擔心它很快會被 Facebook 廢棄。

Flipper 現在已經被[廢棄](https://github.com/react-native-community/discussions-and-proposals/blob/main/proposals/0641-decoupling-flipper-from-react-native-core.md)了，我自己的 side project 都因為 target Android 15 時發現 app crash 而移除 Flipper。因為我的 side project 已經轉用 Compose，所以從 Flipper 那邊只會看到 `ComposeView`，詳細的 composable function hierarchy 還是要用 Android Studio 看。Shared preferences 都改用 AndroidX DataStore 所以亦用不上 Flipper。其餘功能用 Android Studio 內置的工具大致上都做到。但如果去到規模大一些的 app，尤其是有點歷史的 app 我還是覺得 Flipper 比較方便。例如 layout 可以看到 `Activity`/`Fragment` class 名，不用特別花時間等 IDE 載入 layout 就能查到。對於有大量 `Activity`/`Fragment` 的 app 來講始終是最方便。尤其是你對 code base 不夠熟，只要打開那個頁面再查一下那個 view hierarchy 就能收窄檢查範圍。另一個最常用的功能是查看 HTTP request/response，這個惟有轉用 proxy server（例如之前介紹過的 [Whistle]({{< ref "2021-ithome-ironman-22-whistle-proxy/index.md" >}})）來看。

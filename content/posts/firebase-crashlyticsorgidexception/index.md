---
title: Firebase Crashlytics 的 CrashlyticsOrgIdException 解決方法
tags:
  - Android
  - Firebase
date: 2020-03-14T11:55:17+08:00
---


最近為自己的 app 加入 Firebase Crashlytics SDK beta（即是使用 Google 的 Firebase Crashlytics Gradle plugin 而不是用 Fabric 那個），但在 build release APK 時出現下面的錯誤：

```text
java.io.IOException: com.google.firebase.crashlytics.buildtools.exception.CrashlyticsOrgIdException: Could not fetch Crashlytics Org Id
> com.google.firebase.crashlytics.buildtools.exception.CrashlyticsOrgIdException: Could not fetch Crashlytics Org Id
  > Could not fetch Crashlytics Org Id
    > Unable to fetch Crashlytics Org Id using app id 1:731121766578:android:4e799392a2e62811
```

我的 app 是在 Firebase 開了兩個 project，一個是開發時用的，另一個是 production 用的。但只有 production 才有這個錯誤。如果把 release build 設為不上載 ProGuard/R8 mapping file 的話就不會有這個錯誤。

```gradle
firebaseCrashlytics {
    mappingFileUploadEnabled false
}
```

起初以為是那個 google-services.json 放錯位置，但我檢查過位置是正確。之後我找了 Firebase 支援，他們最後回覆說原因是我[沒有在 production 的 Firebase project 交過一次 crash report](https://stackoverflow.com/questions/59837528/build-failed-with-crashlyticsorgidexception/60029241#60029241)，所以 production Firebase project 的 Crashlytics Org Id 就沒有產生出來，導致交不了 ProGuard/R8 mapping file。只不過他們的文檔沒有寫明一定要 crash 一次才算正式完成安裝 Crashlytics。

{{< figure src="firebase-crashlytics.png" title="成功看到 deobfuscate 過的 crash report" >}}

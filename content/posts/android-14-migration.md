---
title: "Android 14 migration"
date: 2024-02-02T00:05:00+08:00
---

最近把 [MetroRide](https://play.google.com/store/apps/details?id=net.swiftzer.metroride) 的 API level 升到 34 ([Android 14](https://developer.android.com/about/versions/14))，中間發現了一些問題，在這裏記錄一下。

## Foreground service

MetroRide 有用到 AndroidX WorkManager 來下載離線資料並放到 SQLite database 內，而且開了 foreground service。在 Android 14 規定要加 permission `FOREGROUND_SERVICE_DATA_SYNC` 標明 foreground service 的目的。

不過去到最後還是把 WorkManager 的 foreground service 抽走。原因是因為如果 target API level 34 而 manifest 又有加到 foreground service permission 的話在 Google Play 上架時要提供影片解釋每個 foreground service 的用法。可能是因為太多 app 用不能清掉的 notification 賣廣告唯有出此下策。為了這個 data sync 拍片解釋實在太煩，所以把它拿掉就算了。

## Google Play Core

另一個要改的地方是 Play Core。本身在開 app 時有用 Play Core 的 in-app update 檢查有沒有更新，如果有的話就會提示用戶更新 app。本身一直都是用 `com.google.android.play:core:1.10.3` 和 `com.google.android.play:core-ktx:1.8.1` 這兩個 artifact，而且是最新版。直到提交了 app 去 Play Store 做 pre-launch report 才知道有 taget API level 34 會 app crash，那時我還不以為然把它上架。最後才發現原來在 Android 14 的裝置會一開 app 即死。

Stack trace 大概是這樣：

```text
Fatal Exception: java.lang.SecurityException: net.swiftzer.metroride: One of RECEIVER_EXPORTED or RECEIVER_NOT_EXPORTED should be specified when a receiver isn't being registered exclusively for system broadcasts
       at android.os.Parcel.createExceptionOrNull(Parcel.java:3069)
       at android.os.Parcel.createException(Parcel.java:3053)
       at android.os.Parcel.readException(Parcel.java:3036)
       at android.os.Parcel.readException(Parcel.java:2978)
       at android.app.IActivityManager$Stub$Proxy.registerReceiverWithFeature(IActivityManager.java:6137)
       at android.app.ContextImpl.registerReceiverInternal(ContextImpl.java:1913)
       at android.app.ContextImpl.registerReceiver(ContextImpl.java:1853)
       at android.app.ContextImpl.registerReceiver(ContextImpl.java:1841)
       at android.content.ContextWrapper.registerReceiver(ContextWrapper.java:772)
       at com.google.android.play.core.listener.zzc.zzb(com.google.android.play:core@@1.10.3:3)
       at com.google.android.play.core.listener.zzc.zzf(com.google.android.play:core@@1.10.3:4)
       at com.google.android.play.core.appupdate.zzf.registerListener(com.google.android.play:core@@1.10.3:1)
       at com.google.android.play.core.ktx.AppUpdateManagerKtxKt$requestUpdateFlow$1$1.onSuccess(AppUpdateManagerKtx.kt:61)
       at com.google.android.play.core.ktx.AppUpdateManagerKtxKt$requestUpdateFlow$1$1.onSuccess(AppUpdateManagerKtx.kt:47)
       at com.google.android.play.core.tasks.zze.run(zze.java:1)
       at android.os.Handler.handleCallback(Handler.java:958)
       at android.os.Handler.dispatchMessage(Handler.java:99)
       at android.os.Looper.loopOnce(Looper.java:230)
       at android.os.Looper.loop(Looper.java:319)
       at android.app.ActivityThread.main(ActivityThread.java:8893)
       at java.lang.reflect.Method.invoke(Method.java)
       at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:608)
       at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:1103)
```

```text
Fatal Exception: android.os.RemoteException: AppUpdateService : Binder has died.
       at com.google.android.play.core.internal.zzas.zzt(com.google.android.play:core@@1.10.3:2)
       at com.google.android.play.core.internal.zzas.zzu(com.google.android.play:core@@1.10.3:2)
       at com.google.android.play.core.internal.zzas.zzm(zzas.java:72)
       at com.google.android.play.core.internal.zzal.zza(com.google.android.play:core@@1.10.3:6)
       at com.google.android.play.core.internal.zzah.run(com.google.android.play:core@@1.10.3)
       at android.os.Handler.handleCallback(Handler.java:938)
       at android.os.Handler.dispatchMessage(Handler.java:99)
       at android.os.Looper.loop(Looper.java:223)
       at android.os.HandlerThread.run(HandlerThread.java:67)
```

後來才發現 Google Play Core 和 Google Play Services 一樣把以往只提供單一 artifact 分拆成多個 artifact 來避免 app 佔用太多儲存空間，而舊的 artifact 沒有再更新。因為 [Android 14 規定 broadcast receiver 要加 receiver flag](https://developer.android.com/develop/background-work/background-tasks/broadcasts#context-registered-receivers)。舊版的 Google Play Core 沒有加到所以只要一 target 到 Android 14 就會 crash。

解決方法是轉用 `com.google.android.play:app-update:2.1.0` 和 `com.google.android.play:app-update-ktx:2.1.0`。

另外，本身在 `Application` 的 `onCreate` 加了檢查 app 是否完整。因為用了 App Bundle，Google Play 會按照當前裝置配置來準備針對該裝置的 APK。（所以要把 release keystore 交到 Play Store 讓 Play Store 為那些 APK 做 code signing）如果用戶把這個 APK 抄去另一部裝置用可能會用不到。但似乎分拆 artifact 後就沒有再提供這個 method，而 Android developers 網站亦都無再介紹用這個 method。所以最後就刪了這個檢查。

```kotlin
override fun onCreate() {
    // Prevent app crash due to copying APK file to another device
    @Suppress("DEPRECATION")
    if (MissingSplitsManagerFactory.create(this).disableAppIfMissingRequiredSplits()) {
        Toast.makeText(this, R.string.app_integrity_check_fail, Toast.LENGTH_SHORT)
            .show()
        return
    }
    super.onCreate()
    // 其他 code
}
```

## Google Play recovery tools

交了新的 app 版本後，無意中發現原來 Google Play Console 現在可以做到 [force update](https://android-developers.googleblog.com/2024/01/prompt-users-to-update-to-your-latest-app-version-google-play.html)。效果跟 in-app update 差不多，但不用改 code 就做到 force update 效果。不過在我這個清況發揮不到作用，因為它一開 app 就 crash，似乎不夠時間讓它顯示 force update 的 UI。

大致上就是這樣，因為我的 app 功能比較少，所以都沒什麼特別的東西要改。

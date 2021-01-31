---
title: React Native Android Multi-window 多視窗支援
tags:
  - Android
  - React Native
date: 2017-12-01T16:05:05+08:00
---


Android 7.0 (N) 新增一次顯示多個 app 功能 (Multi-window)。即是可以兩個 app 上下或左右並排。如果想你的 React Native app 能支援這個功能的話，首先要檢查 *build.gradle* 的 SDK 版本（24 或以上）。

之後在 *AndroidManifest.xml* 應該會找到下面類似的 `<activity>`：

{{< highlight xml >}}
<activity
    android:name=".MainActivity"
    android:label="@string/app_name"
    android:configChanges="keyboard|keyboardHidden|orientation|screenSize"
    android:windowSoftInputMode="adjustResize">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
{{< /highlight >}}

在 `android:configChanges` 補上 `smallestScreenSize` 和 `screenLayout`：

{{< highlight xml >}}
<activity
    android:name=".MainActivity"
    android:label="@string/app_name"
    android:configChanges="keyboard|keyboardHidden|orientation|screenSize|smallestScreenSize|screenLayout"
    android:windowSoftInputMode="adjustResize">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
{{< /highlight >}}

之後就可以支援這個功能了。

修改 `android:configChanges` 的原因是因為 Android 在進行 Multi-window 動作時（例如由一個視窗變成兩個視窗並排顯示時、視窗尺寸改變時），都會被當作 [configuration change](https://developer.android.com/guide/topics/resources/runtime-changes.html) 處理。即是 activity 會執行 `onDestroy` 之後再執行 `onCreate`。但 React Native 是由它自己處理 configuration change，所以 React Native 的 project 就在 `<activity>` 加入 `android:configChanges`，令原本因旋轉畫面之類的 configuration change 都不會執行 `onDestroy` 和 `onCreate`，重新整理 activity 界面就交由 React Native 處理。但 Multi-window 是有新的 `android:configChanges` 常數，所以現在就要補回，否則用 Multi-window 會把 React Native app 的狀態掉失。

---
title: Parse + Android 收 Push Notification
date: 2015-08-30T17:13:46+08:00
tags:
  - Android
  - Parse
---

[Parse](https://www.parse.com) 的免費 plan 包含了一百萬個接收者的 push 配額，應該足夠一般 app 使用，而且比起自設 server 發送 push notification 更加方便（不用去 Google Developers Console 開 project）。但 Parse 網站所提供的 Android 的教學有點不清楚，而且都過時。在此分享一下 Android 版的基本 setup。<!--more-->

**第一步：** 在 Android Studio 按照平時開新 project 的方法來建立一個新的 project。

**第二步：** 在 module 的 _build.gradle_ 加入 Parse 的 dependency。

{{< highlight groovy >}}
dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    compile 'com.android.support:appcompat-v7:23.0.0'
    compile 'com.parse.bolts:bolts-android:1.2.1'
    compile 'com.parse:parse-android:1.10.1'
}
{{< /highlight >}}

Parse 的教學是下載 JAR 到自己的 project，但其實也可以用 Gradle 來安裝。

不加入 [Bolts framework](https://github.com/BoltsFramework/Bolts-Android) 和 Google Play services 也是可以的，Parse 會檢查設備是否有 Google Play service，如果有的話就會用 Google Cloud Messaging (GCM)，沒有的話可以 fallback 到 Parse 自己的 socket connection。

**第三步：** 建立一個 `Application` 的 sub-class。在 `onCreate` 的時候就向 Parse 註冊接收 push notification。

{{< highlight java >}}
public class MyApplication extends Application {
    private static final String TAG = MyApplication.class.getSimpleName();

    @Override
    public void onCreate() {
        super.onCreate();

        // Enable Local Datastore.
        Parse.enableLocalDatastore(this);

        // Add your initialization code here
        Parse.initialize(this);

        // Subscribe to a channel
        ParsePush.subscribeInBackground("MyChannel", new SaveCallback() {
            @Override
            public void done(ParseException e) {
                if (e == null) {
                    Log.d(TAG, "Successfully subscribed to Parse channel");
                } else {
                    Log.d(TAG, "Failed to subscribe to Parse channel");
                }
            }
        });

        ParseUser.enableAutomaticUser();
        ParseACL defaultACL = new ParseACL();
        // Optionally enable public read access.
        // defaultACL.setPublicReadAccess(true);
        ParseACL.setDefaultACL(defaultACL, true);
    }
}
{{< /highlight >}}

**第四步：** 在 _strings.xml_ 加入兩個 `<string>`，入面的 xxx 是用來放 Parse app 的 App ID 和 Client key。這些 key 可以在 Parse app 的 Settings > Keys 找到。

{{< highlight xml >}}
<string name="parse_app_id">xxxxx</string>
<string name="parse_client_key">xxxxx</string>
{{< /highlight >}}

**第五步：** 修改 _AndroidMenifest.xml_，令 Android app 能接收 Push notification。

在 `<application>` 之前，加入權限相關的 tag：

{{< highlight xml >}}
<uses-permission android:name="android.permission.INTERNET"/>
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
<uses-permission android:name="android.permission.WAKE_LOCK"/>
<uses-permission android:name="android.permission.VIBRATE"/>
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED"/>
<uses-permission android:name="android.permission.GET_ACCOUNTS"/>
<uses-permission android:name="com.google.android.c2dm.permission.RECEIVE"/>

<permission
    android:name="your.package.name.permission.C2D_MESSAGE"
    android:protectionLevel="signature"/>
<uses-permission android:name="your.package.name.permission.C2D_MESSAGE"/>
{{< /highlight >}}

將 `your.package.name` 改成 app 的 package name。如果你的 app 只支援有 Google Play service 的設備的話，可以不加 `RECfEIVE_BOOT_COMPLETED`。

在 `<application>` 內使用剛才的 Application sub-class：

{{< highlight xml >}}
<application
    android:name=".MyApplication"
    android:allowBackup="true"
    android:icon="@mipmap/ic_launcher"
    android:label="@string/app_name"
    android:theme="@style/AppTheme">
{{< /highlight >}}

在 `<application>` 內加入 Parse 的 meta data tag：

{{< highlight xml >}}
<meta-data
    android:name="com.parse.APPLICATION_ID"
    android:value="@string/parse_app_id"/>
<meta-data
    android:name="com.parse.CLIENT_KEY"
    android:value="@string/parse_client_key"/>
<meta-data
    android:name="com.parse.push.notification_icon"
    android:resource="@mipmap/ic_launcher"/>
{{< /highlight >}}

Notification icon 暫時設成 app icon，你可以再改成其他 icon。

同樣在 `<application>` 內，加入處理 push notification 的 service 和 receiver：

{{< highlight xml >}}
<service android:name="com.parse.PushService"/>

<receiver android:name="com.parse.ParseBroadcastReceiver">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED"/>
        <action android:name="android.intent.action.USER_PRESENT"/>
    </intent-filter>
</receiver>
<receiver
    android:name="com.parse.ParsePushBroadcastReceiver"
    android:exported="false">
    <intent-filter>
        <action android:name="com.parse.push.intent.RECEIVE"/>
        <action android:name="com.parse.push.intent.DELETE"/>
        <action android:name="com.parse.push.intent.OPEN"/>
    </intent-filter>
</receiver>
<receiver
    android:name="com.parse.GcmBroadcastReceiver"
    android:permission="com.google.android.c2dm.permission.SEND">
    <intent-filter>
        <action android:name="com.google.android.c2dm.intent.RECEIVE"/>
        <action android:name="com.google.android.c2dm.intent.REGISTRATION"/>
        <category android:name="your.package.name"/>
    </intent-filter>
</receiver>
{{< /highlight >}}

將 `your.package.name` 改成 app 的 package name。如果你的 app 只支援有 Google Play service 的設備的話，可以移除帶有 `BOOT_COMPLETED` 及 `USER_PRESENT` 的 `<receiver>` tag。

當收到 push notification 的時候，Parse SDK 預設會彈出一個 notification，按一下 notification 會開啟 main activity。如果需要自訂的話可以 override `ParsePushBroadcastReceiver` 的 `onPushReceived`、`onPushOpen`、`onPushDismiss` 和 `getNotification`。

**第六步：** 在 main activity 的 `onCreate` 加入 Parse tracker。

{{< highlight java >}}
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    ParseAnalytics.trackAppOpenedInBackground(getIntent());
}
{{< /highlight >}}

**第七步：** 現在已經完成設定，在設備 run 一次 app，讓它向 Parse 註冊 push notification。然後到 Parse 網站發送一個 push，你的設備應該會收到 push。而且 database 的 Installation class 會有一筆新記錄。

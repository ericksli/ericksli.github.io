---
title: "Firebase Cloud Messaging legacy API"
date: 2024-06-01T14:00:00+08:00
tags:
  - Android
  - Firebase
draft: true
---

如果有用 Firebase Cloud Messaging (FCM) 或者其他關於 FCM 的第三方 SDK 的話應該會收到通知說 6 月 21 日會停用舊版的 FCM API（即是供 server 發送 notification 那個 API endpoint）。之後就要轉用 [HTTP v1 API](https://firebase.google.com/docs/reference/fcm/rest/v1/projects.messages/send)。

這個改動主要改變了拿 API token 的方法，以前是直接用 Firebase console 提供的 token 就能 call 那些 endpoint，但現在變成要先拿到 service account 的 JSON 檔案，之後再用 [Google 提供的 Auth Library](https://github.com/googleapis?q=auth) 生成有效期較短的 OAuth 2.0 access token 才能 call 到新的 endpoint。

在 6 月 21 日之後如果要發送 FCM 推送的話（例如想測試自己的 app 能不能正常處理推送），最簡單的方法是用 [Firebase Admin SDK](https://firebase.google.com/docs/admin/setup) 發送推送。

首先要去 Firebase 的 project settings 頁，在 Firebase Admin SDK 分頁會找到「Generate new private key」按鈕，按一下這個按鈕。

{{< figure src="firebase-project-settings.png" title="Project settings" >}}


之後會彈出一個警告，再按「Generate key」。

{{< figure src="generate-new-key-dialog.png" title="警告" >}}

按下之後就能下載到一個 JSON 檔案。

之後就可以用 Firebase Admin SDK 來發送通知，以下會用 Java 版的 Admin SDK 及 Kotlin 示範：

```kotlin
val googleCredentials = GoogleCredentials
    .fromStream(FileInputStream("credentials.json"))
    .createScoped("https://www.googleapis.com/auth/firebase.messaging")
val firebaseOptions = FirebaseOptions.builder()
    .setCredentials(googleCredentials)
    .build()
val firebaseApp = FirebaseApp.initializeApp(firebaseOptions)

FirebaseMessaging.getInstance(firebaseApp).send(Message.builder()
    .setToken("e0xX9...")
    .setNotification(Notification.builder()
        .setTitle("Title goes here")
        .setBody("body message goes here")
        .setImage("https://firebase.google.com/static/docs/cloud-messaging/images/Localization_v2.png")
        .build())
    .build())
```

留意要先加 `com.google.firebase:firebase-admin` 這個 library。

由於 Admin SDK 已經封裝了一次 HTTP v1 API，所以可以直接用他們的 function 來寫 request。如果想自己寫 HTTP request 的話，你可以只用開首那個建構 `GoogleCredentials` 部分，之後再由 `GoogleCredentials` 拿到加進 `Authorization` header 那個 token：

```kotlin
val googleCredentials = GoogleCredentials
    .fromStream(/* ... */)
    .createScoped(/* ... */)
googleCredentials.refresh()
val accessToken = googleCredentials.accessToken.tokenValue
```

`Authorization` header 的值是 `"Bearer $accessToken"`。

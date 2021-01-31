---
title: Firebase Cloud Messaging
tags: 
- Android
- Firebase
date: 2020-02-29T14:07:18+08:00
---


最近工作需要做 Firebase Cloud Messaging (FCM) 整合，發現了向 Firebase API 直接送出 push 的 HTTP request 都可以生成不同種類的 message。

如果要整合到 Android 的話，需要建立一個新的 `Service` class 並繼承自 [`FirebaseMessagingService`](https://firebase.google.com/docs/reference/android/com/google/firebase/messaging/FirebaseMessagingService)。這個 `Service` 有一個叫 `onMessageReceived` 的 callback method 來接收來自 FCM 的 push 和它的 payload。但原來不是所有的 push 都能被那個 callback 接到，要視乎 push 的種類和你的 app 當時在甚麼情況而定。

<!-- more -->

FCM 的 push 有分兩種：Notification message 和 Data message。Notification message 就是那些在發送時預先指明式樣的 push。即是標題、內文、[notification channel](https://developer.android.com/training/notify-user/channels) 之類的 push。這種 push 可以用 Firebase 的 [Notifications composer](https://console.firebase.google.com/u/0/project/_/notification) 造出來，完全不用寫 code。如果裝置收到 push 時你的 app 是在 foreground 的話就會觸發 `onMessageReceived` callback 讓你自己處理；在 background 時就直接後會由 Firebase SDK 直接生成 Android 看到的系統通知（即是在 system tray 看到的 `NotificationCompat`）而不會觸發 `onMessageReceived` callback。如果要在 Firebase API 發送 push 的話 request body 大概是這樣：

{{< highlight json >}}
{
  "message": {
    "token": "bk3RNwTe3H0:CI2k_HHwgIpoDKCIZvvDMExUdFQ3P1...",
    "notification": {
      "title": "Portugal vs. Denmark",
      "body": "great match!"
    }
  }
}
{{< /highlight >}}

而 Data message 就不會由 Firebase SDK 自動生成 UI，不管你的 app 是在 foreground 還是 background 都是要靠自已在 `FirebaseMessagingService` 的 `onMessageReceived` callback 生成 `NotificationCompat` 才會顯示出來（Notification message 不論 app 是在 foreground 還是 background 都會觸發 `onMessageReceived` callback）。而它只會附帶 key-value pair payload，沒有那些預先設計好的標題、內文欄之類。Firebase API 的 request body 會是這樣：

{{< highlight json >}}
{
  "message": {
    "token": "bk3RNwTe3H0:CI2k_HHwgIpoDKCIZvvDMExUdFQ3P1...",
    "data": {
      "Nick": "Mario",
      "body": "great match!",
      "Room": "PortugalVSDenmark"
    }
  }
}
{{< /highlight >}}

但其實一個 push 可以兩者皆是，即是又可以提供用戶可見的標題、內文，又可以提供自訂的 key-value pair payload。Firebase API 的 request body 又會是這樣：

{{< highlight json >}}
{
  "message": {
    "token":"bk3RNwTe3H0:CI2k_HHwgIpoDKCIZvvDMExUdFQ3P1...",
    "notification": {
      "title": "Portugal vs. Denmark",
      "body": "great match!"
    },
    "data": {
      "Nick": "Mario",
      "Room": "PortugalVSDenmark"
    }
  }
}
{{< /highlight >}}

跟之前兩款不同的地方是它既有 `notification` 又有 `data` 兩個 object。在這種情況下 app 是在 foreground 的話會觸發 `onMessageReceived` callback；在 background 的話就像 Notification message 般直接在裝置的 system tray 顯示 notification。當用戶按下 notification 開 app 時那些 payload 就會由 intent extra 送到預設開 app 的 `Activity` 內。

所以如果你想不論何時都能讓 app 即時處理到 push 的話，那就不要附帶 `notification` object。這樣就變成 Notification message，不論 foreground 還是 background 都可以被 app 收到。

## 用了 Notification message 但 `onMessageReceived` 沒有被 call

有時即使裝置已經收到 Notification message 的 push 但仍然收不到 `onMessageReceived` callback，在 Logcat 可以看到這段 log：

```
2020-02-18 11:36:41.862 3369-3369/? W/GCM: broadcast intent callback: result=CANCELLED forIntent { act=com.google.android.c2dm.intent.RECEIVE pkg=net.swiftzer.metroride (has extras) }
```

這個情況在 Firebase 的文檔應該沒有提及，但 [Stack Overflow](https://stackoverflow.com/a/53404817) 有人回答過相同問題。沒有 callback 但出現以上的 log 原因是因為這就是 Android framework 的設計：如果用戶主動 kill app 的話即使收到 FCM push 都不會啟動你的 app，直至用戶再次主動開啟你的 app 才會解除限制，重新開機也不可重設，一定要主動開 app 才會解除。用戶主動 kill app 是指用戶在設定頁找到你的 app 再按強制停止，如果是因為系統不夠 RAM 而 kill app 就不算。

但這個回答有提過這個機制在某些品牌會變成用戶在最近使用 app 的畫面掃走 app 時都會有這個效果。所以比較穏妥的做法是如果那個 notification 不需要 app 去特別處理（例如 payload 的話）都附帶 `notification` object。

## 參考

- [About FCM messages](https://firebase.google.com/docs/cloud-messaging/concept-options)（上面的 JSON 都是在那裏抄的）
- [Receive messages in an Android app](https://firebase.google.com/docs/cloud-messaging/android/receive)
- [Error broadcast intent callback: result=CANCELLED forIntent { act=com.google.android.c2dm.intent.RECEIVE pkg=com.flagg327.guicomaipu (has extras) }](https://stackoverflow.com/questions/39480931/error-broadcast-intent-callback-result-cancelled-forintent-act-com-google-and)

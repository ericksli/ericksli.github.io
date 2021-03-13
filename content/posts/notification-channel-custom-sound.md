---
title: "Notification Channel 自訂音效"
date: 2021-03-13T22:10:00+08:00
draft: true
tags:
  - Android
---

自從由 Android 8 開始，如果要顯示 [notification](https://developer.android.com/guide/topics/ui/notifiers/notifications) 的話就一定要指定一個 [notification channel](https://developer.android.com/training/notify-user/channels)，否則系統不會顯示。Notification channel 的目的是讓用戶能自行調節 app 的各式 notification 的提示方法，例如有沒有音效、會不會彈出 heads-up notification 之類。如果 app 想自訂 notification 的聲音亦都要經 notification channel 設定（但用戶可以之後自行變更 notification channel 的音效）

但是要留意，設定自訂音效時那個 `Uri` 要用檔案名稱來設定，不要用 resource ID。以下是錯誤例子：

```kotlin
val uri = Uri.Builder()
    .scheme(ContentResolver.SCHEME_ANDROID_RESOURCE)
    .authority(context.packageName)
    .appendPath(R.raw.example_notification.toString())
    .build()
```

其實上面那個 `Uri` 是可以播放那個音效的。但問題是那個音效設定是儲存在系統而不是在自己的 app 入面。每次 build app 的時候不會保證那個 `R.raw.example_notification` 的 resource ID 是固定，如果在設定 notification channel 時用了 resource ID 的話日後 *raw* 資料夾的檔案有增減時就播了另一個檔案（因為現在的 build tool 是順着英文字母順序編排 resource ID，只要排序前面加插一個新的檔案就會播錯音效。）由於加插 *raw* 檔案導致播錯音效較難察覺，而且日後在 *raw* 加入新檔案的機會比把音效檔案重新命名的機會較大，所以在設定 notification channel 音效的 `Uri` 就應該用檔案名稱，就像下面那例子：

```kotlin
fun createNotificationChannel(context: Context) {
    if (Build.VERSION.SDK_INT < Build.VERSION_CODES.O) return

    val channel = NotificationChannel(
        "general",
        context.getString(R.string.notification_channel_title_general),
        NotificationManager.IMPORTANCE_HIGH
    ).apply {
        description = context.getString(R.string.notification_channel_desc_general)
        val uri = Uri.Builder()
            .scheme(ContentResolver.SCHEME_ANDROID_RESOURCE)
            .authority(context.packageName)
            .appendPath("raw")
            .appendPath("example_notification")
            .build()
        val attributes = AudioAttributes.Builder()
            .setUsage(AudioAttributes.USAGE_NOTIFICATION)
            .build()
        setSound(uri, attributes)
    }

    val notificationManager = context.getSystemService(NotificationManager::class.java)
    notificationManager?.createNotificationChannel(channel)
}
```

還有一點要留意：刪除 notification channel 後再重新建立相同 channel ID 的 notification channel 並不能重設用戶對 channel 的設定（包括 app 在建立 notification channel 時所定義的通知音效）。這是一個防止濫用的機制。如果那個 app 不斷彈出通知的話那用戶可以把那個 app 的 notification channel 重要性降低（例如不彈出通知），但如果 app 刪除 channel 再建立 channel 就能把那些用戶設定重設的話那 notification channel 功能就變得多餘。故此 Android notification channel 的用戶設定是有記憶的，重新建立相同 channel ID 的 notification channel 只會把之前的設定帶回來。設定頁更會顯示那個 app 刪除 notification channel 的次數來提醒用戶可能這個 app 可能有 spam。如果之前已經錯誤地用了 resource ID 來設定 notification channel 音效的話解決方法就只有改用另一個全新的 notification channel。

## 參考

- [Android Notification Channel sounds stop working when using sound URIs that reference resource ids](https://stackoverflow.com/questions/54796492/android-notification-channel-sounds-stop-working-when-using-sound-uris-that-refe/54796493#54796493)
- [Categories deleted in notification settings](https://us.community.samsung.com/t5/Galaxy-S-Phones/Categories-deleted-in-notification-settings/td-p/1277742)

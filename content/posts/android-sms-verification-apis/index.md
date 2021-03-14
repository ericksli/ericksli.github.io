---
title: "Android SMS Verification APIs"
date: 2021-03-14T22:20:00+08:00
tags:
  - Android
---

SMS 驗證應該是一個在 Android app 頗為常見的需求。一般做法都是先讓用戶填寫電話號碼，然後 app 會把電話號碼交到 backend 再透過 SMS gateway 發送含有驗證碼短訊，
當用戶收到 SMS 後再把內文的驗證碼輸入到 app 中。如果想省卻用戶輪入文字的話有一些 app 會透過 `READ_SMS` 權限讀取 SMS 內容來抽取驗證碼，但 Google Play 已經[限制非預設短訊 app
不可以有 `READ_SMS` 權限](https://support.google.com/googleplay/android-developer/answer/10208820)。

<!-- more -->

## SMS Retriever API

如果有留意過一些 app 的驗證 SMS 的話，可以發現到有一些 SMS 內文結尾會加插一些英數字符。

{{< figure src="sms-retriever-sample.png" title="SMS Retriever 短訊例子" >}}

當系統這種 SMS 後 app 就能自動填入那個 SMS 內的驗證碼，但特別的是那些 app 並沒有要求 `READ_SMS` 權限。其實它們是用了 Google 的 [SMS Retriever APIs](https://developers.google.com/identity/sms-retriever/overview)。簡單來講，就是 Google Play Services 代你的 app 拿了 `READ_SMS` 權限，由 Google Play Services 中央處理那些讀取 SMS 驗證碼的權限處理。最尾那一串英數字符就是給 Google Play Services 判斷這個 SMS 是要交到那個 app 處理。

其實 SMS Retriever API 用法其實不太複雜，大概步驟就是：

1. Backend 發送 SMS 時在內文結尾加上一個特有的英數字符
2. app 通知 SMS Retriever 開始監測系統接收到的 SMS
3. SMS Retriever 接收到 SMS 後按照末端的英數字符通知對應的 app，app 透過 `BroadcastReceiver` 接收 SMS 內文，從中抽取驗證碼並繼續流程

### SMS 內容

按照[說明文檔](https://developers.google.com/identity/sms-retriever/verify)，SMS 不能大過 140 bytes 和包含一個 11 位長的英數字符。那個英數字符是用 app 的 keystore 加上 application ID 生成出來，除了透過 [shell script](https://github.com/googlearchive/android-credentials/blob/master/sms-verification/bin/sms_retriever_hash_v9.sh) 生成之外，Google 還留了一個 [`AppSignatureHelper`](https://github.com/googlearchive/android-credentials/blob/master/sms-verification/android/app/src/main/java/com/google/samples/smartlock/sms_verify/AppSignatureHelper.java) 方便大家生成那個字串。

先前的 SMS 例子開首有個 `<#>`，這其實是用來表示這個 SMS 是這是一個提供一次性密碼的 SMS。但現在 SMS Retriever 已經沒有這個規定。

### 開始監測

要觸發 SMS Retriever 開始監測收到的 SMS，只需要 call `SmsRetriever` 的 `startSmsRetriever` 就可以了，監測期為五分鐘。[^1] 下面的例子用了 [kotlinx-coroutines-play-services](https://github.com/Kotlin/kotlinx.coroutines/tree/master/integration/kotlinx-coroutines-play-services) 的 `await` 把原本 Google Play Services 的 [Tasks API](https://developers.google.com/android/guides/tasks) 變成 coroutine。[^2] 緊記在通知 backend 發送時馬上通知 SMS Retriever，否則 SMS Retriever 就趕不來截取那個 SMS。

[^1]: [`SmsRetrieverApi#startSmsRetriever`](https://developers.google.com/android/reference/com/google/android/gms/auth/api/phone/SmsRetrieverApi#startSmsRetriever()): Starts `SmsRetriever`, which waits for a matching SMS message until timeout (5 minutes).
[^2]: kotlinx-coroutines-play-services 的 group 和 artifact 名稱是 `org.jetbrains.kotlinx:kotlinx-coroutines-play-services`。


```kotlin
import com.google.android.gms.auth.api.phone.SmsRetriever
import kotlinx.coroutines.tasks.await

suspend fun startSmsRetriever() {
    val client = SmsRetriever.getClient(context)
    client.startSmsRetriever().await()
}
```

不要忘記加入 Google Play Services auth component（最新版本請查 [Google's Maven Repository](https://maven.google.com/web/index.html)）

```groovy
implementation 'com.google.android.gms:play-services-auth:17.0.0'
implementation 'com.google.android.gms:play-services-auth-api-phone:17.4.0'
```

### 讀取驗證碼

當裝置收到 SMS 後，SMS Retriever 就會透過 `BroadcastReceiver` 通知對應的 app。在 `BroadcastReceiver` 入面可以透過 `SmsRetriever.EXTRA_SMS_MESSAGE` 取得該 SMS 內文。之後就可以用 regular expression 之類的方法抽取驗證碼，再通知 UI 填寫驗證碼繼續流程。

```kotlin
class SmsRetrieverBroadcastReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        if (SmsRetriever.SMS_RETRIEVED_ACTION == intent.action) {
            val status = intent.extras?.get(SmsRetriever.EXTRA_STATUS) as Status?
            when (status?.statusCode) {
                CommonStatusCodes.SUCCESS -> {
                    // Success, obtain the SMS message body
                    val message = intent.extras?
                            .getString(SmsRetriever.EXTRA_SMS_MESSAGE)
                            .orEmpty()
                }
                CommonStatusCodes.TIMEOUT -> {
                    // Error
                }
            }
        }
    }
}
```

因為加了 `BroadcastReceiver`，所以要在 *AndroidManifest.xml* 加上 `<receiver>` tag：

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <application>
        <receiver
            android:name=".SmsRetrieverBroadcastReceiver"
            android:exported="true"
            android:permission="com.google.android.gms.auth.api.phone.permission.SEND">
            <intent-filter>
                <action android:name="com.google.android.gms.auth.api.phone.SMS_RETRIEVED" />
            </intent-filter>
        </receiver>
    </application>
</manifest>
```

## SMS User Consent

上面介紹的 SMS Retriever API 只適用於能夠控制到 SMS 內文的情況。但如果那個 SMS 並不能自己控制內文的話（例如由銀行發出），那就要使用 [SMS User Consent API](https://developers.google.com/identity/sms-retriever/user-consent/overview)。做法和 SMS Retriever API 相似，最大分別是系統會顯示一個 bottom sheet 詢問用戶是不是想把收到的那個 SMS 給予 app 讀取。如果同意的話 app 就會透過 `BroadcastReceiver` 接收 SMS 原文。

## 延伸閱讀

- [Is Apple moving to standardize the SMS OTP format?](https://www.smsglobal.com/blog/standard-otp-format/)
- [Origin-bound one-time codes delivered via SMS
](https://wicg.github.io/sms-one-time-codes/)

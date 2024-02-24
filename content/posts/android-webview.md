---
title: Android WebView 筆記
date: 2024-02-24T11:45:00+08:00
tags:
- Android
---

好幾年都沒有特別去用 Android 的 `WebView`，近期工作需要用到 `WebView`，所以特別去查一下並將資料放在這篇文章內方便日後翻查。

## AndroidX WebKit

AndroidX 其實有 WebKit 的 artifact [`androidx.webkit:webkit`](https://maven.google.com/web/index.html#androidx.webkit:webkit)，但不是把整個 browser 加到 app 入面（`WebView` 實際在用的 web browser 是由 [Google Play Store 提供](https://play.google.com/store/apps/details?id=com.google.android.webview)，並可以 developer options 切換），而是把部分較新的 `WebView` 功能加個檢查 function，如果目前的 `WebView` 支援那個功能的話就可以執行這部分的 code。

AndroidX WebKit 入面有 `WebViewClientCompat`，這是用來 polyfill 部分 Android SDK 內的 `WebViewClient`。相信最值得一提的地方是 `shouldOverrideUrlLoading`。這個 method 是用來控制 `WebView` 是否載入這個網址。你可以在這個 callback 內截停某些網址載入，然後做其他東西。比如因應某些網址來 `startActivity` 讓那些網址改為系統瀏覽器開啟。另一個經典用法是用作由 JavaScript 傳遞資料給 native app，但如果經 URL 傳遞很長的 string 的話（例如 base64 encode 的圖片）很容易會 lag 機。

留意 `WebViewClientCompat` 的 `shouldOverrideUrlLoading` 跟 Android SDK 那個行為不同：`WebViewClientCompat` 會把除 `loadUrl` 的頁面都改由系統預設瀏覽器開啟。

## Lifecycle

`WebView` 是一個比較麻煩的 UI 組件，因為要特別手動接駁 Android `Activity`/`Fragment` 的 lifecycle，如果做漏的話 `WebView` 的 state 就會在 configuration change 後出錯。就好似 Google Maps SDK 的 `MapView` 一樣[需要 forward lifecycle method](https://developers.google.com/maps/documentation/android-sdk/reference/com/google/android/libraries/maps/MapView)。如果是 `Activity` 的話大概是這樣：

```kotlin
class WebViewActivity : AppCompatActivity() {
    private lateinit var webView: WebView

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_webview)
        webView = findViewById(R.id.webview)

        // 在載入 WebView 第一頁前要檢查 savedInstanceState
        // 如果 configuration change 前用戶已經載入過或者按了連結去另一頁的話
        // 不檢查 savedInstanceState 就會在 configuration change 後載入第一頁
        if (savedInstanceState == null) {
            webView.loadUrl("https://example.com")

            // 如果要加額外的 header（例如盜連 CodePen），可以在 loadUrl 再加 Map
            webView.loadUrl(
                "https://cdpn.io/RhinoLu/fullpage/RrPxMv?anon=true&view=",
                mapOf("Referer" to "https://codepen.io/RhinoLu/pen/RrPxMv"),
            )
        }
    }

    override fun onPause() {
        super.onPause()
        webView.onPause()
    }

    override fun onResume() {
        super.onResume()
        webView.onResume()
    }

    override fun onDestroy() {
        super.onDestroy()
        webView.destroy()
    }

    override fun onSaveInstanceState(outState: Bundle) {
        super.onSaveInstanceState(outState)
        webView.saveState(outState)
    }

    override fun onRestoreInstanceState(savedInstanceState: Bundle) {
        super.onRestoreInstanceState(savedInstanceState)
        webView.restoreState(savedInstanceState)
    }
}
```

## JavaScript 與 native app 雙向溝通

用 AndroidX WebKit 的主要原因是為了用 `WebMessageListener`。傳統的 JavaScript 與 native app 溝通方法是靠加了 [`@JavascriptInterface` annotation](https://developer.android.com/reference/android/webkit/JavascriptInterface) 的 class 做 JavaScript 向 native 傳遞訊息，由 native 向 JavaScript 傳遞訊息就用 [`WebView.evaluateJavascript`](https://developer.android.com/reference/android/webkit/WebView#evaluateJavascript(java.lang.String,%20android.webkit.ValueCallback%3Cjava.lang.String%3E)) call JavaScript 的 function。

自從 Android 6 (API Level 23) 開始，`WebView` 開始支援 `postWebMessage`，加上 AndroidX WebKit 有 `addWebMessageListener` method，我們可以用這兩個 method 實現 native app 和 JavaScript 雙向溝通。

### Native app 向 JavaScript 通訊

在 Android app 那邊，我們假設用戶按下 `sendData` 按鈕後就會發送一個帶有當前時間的 string：

```kotlin
findViewById<Button>(R.id.sendData).setOnClickListener {
    if (WebViewFeature.isFeatureSupported(WebViewFeature.POST_WEB_MESSAGE)) {
        WebViewCompat.postWebMessage(
            webView,
            WebMessageCompat("Hello from Android @ ${Instant.now()}"),
            Uri.parse("https://cdpn.io"),
        )
    } else {
        // postWebMessage is not supported
    }
}
```

上面的 code 首先要檢查一下目前的 `WebView` 是否支援 `POST_WEB_MESSAGE`，如果支援的話就向當前 `https://cdpn.io` 作域名的網頁發送那段 string。加上 `targetOrigin` 那個參數是為了加強保安，避免把內容發送到不是我們期望的網站。如果不想限制就可以用 `Uri.parse("*")`。

而 JavaScript 那邊就可以用 `addEventListener` 收到那段 string。String 的內容是放在 `event.data` 內。

```js
window.addEventListener('message', receiver, false);
function receiver(event) {
  console.log(event.data);
}
```

### JavaScript 向 native app 通訊

首先在 Android 那邊要加 `WebMessageListener`，同樣地都要先檢查是否支援 `WEB_MESSAGE_LISTENER`。那個 `Android` 是 JavaScript 那邊看到的 object 名稱。而 `setOf("https://cdpn.io")` 是指明 `WebView` 在那些域名才把 `Android` object 塞入去 `window` object 內，都是為了加強保安。以前 `addJavascriptInterface` 是沒有這個功能，凡是載入網頁都會把 `Android` object 塞入去 `window` object 內。

```kotlin
if (WebViewFeature.isFeatureSupported(WebViewFeature.WEB_MESSAGE_LISTENER)) {
    WebViewCompat.addWebMessageListener(
        webView,
        "Android",
        setOf("https://cdpn.io"),
        MyWebMessageListener(),
    )
}
```

去到 `WebMessageListener` 的部分，只需要 override `onPostMessage` method 獲取 `message.data`。如果想在收到 message 後馬上回覆 JavaScript 的話可以用 `replyProxy.postMessage` 來向 JavaScript 通訊。

```kotlin
class MyWebMessageListener() : WebViewCompat.WebMessageListener {
    override fun onPostMessage(
        view: WebView,
        message: WebMessageCompat,
        sourceOrigin: Uri,
        isMainFrame: Boolean,
        replyProxy: JavaScriptReplyProxy,
    ) {
        Toast.makeText(this, "Received message from WebView: ${message.data}", Toast.LENGTH_SHORT).show()
        // 收到後馬上回覆 JavaScript
        if (WebViewFeature.isFeatureSupported(WebViewFeature.WEB_MESSAGE_LISTENER) && replyProxy != null) {
            replyProxy.postMessage("got the message @ ${Instant.now()}")
        }
    }
}
```

而 JavaScript 那邊就是先檢查有沒有那個 `Android` object，有的話就可以 call `postMessage` 向 native app 通訊。如果要收到來自 native app 的回應，要再為 `onmessage` 設定 callback function。

```js
document
  .getElementById("postMessage")
  .addEventListener("click", function (event) {
    if (typeof Android !== "undefined") {
      Android.postMessage("testing");
    }
  });

// Immediately receive message right after calling Android.postMessage
if (typeof Android !== "undefined") {
  Android.onmessage = function (event) {
    console.log("Android.onmessage received: ", event);
  }
}
```

雙向通訊大概就是這樣。除了傳送 string 之外，其實還可以傳送 binary data。Native app 那邊是用 `ByteArray` type；而 JavaScript 那邊就用 `ArrayBuffer` type。以為就不用刻意 base64 encode 圖片之類的 binary 文件。

另一樣要留意的是 `WebMessageListener` 執行的 thread 跟以往 `@JavascriptInterface` 不一樣。

## Back button 與進度條

`Activity` 的 `onBackPressed` method 因應 Android 14 (API level 34) 的 predictive back gesture 功能而 deprecated，如果要令 `WebView` 的上一頁加到 Android 系統的 back button 的話就要用 `onBackPressedDispatcher`。

`onBackPressedDispatcher` 的用法其實跟 `Activity` 的 `onBackPressed` 完全相反。`onBackPressed` 是用戶按了 back 後會 call，你在這個 function 內控制是否 call super class 的 `onBackPressed`（即是真的讓系統處理 back）。而 `onBackPressedDispatcher` 是要你主動通知它目前是不是交由系統處理 back，如果設定成不由系統處理的話就會執行你的 `OnBackPressedCallback`。

要做到這個效果我們需要 override `WebViewClientCompat` 來取得目前 `WebView` 的 event，主要是留意 `onPageStarted`、`onPageFinished`、`onReceivedError`、`onReceivedHttpError`，然後在裏面檢查現在的 `WebView.canGoBack`。

另一樣是進度條，進度數值和是否需要顯示進度條是靠 `WebViewClientCompat` 和 `WebChromeClient`。`WebChromeClient` 沒有 compat 版，直接 override Android SDK 提供的就可以。

這兩項的 code 大約是這樣：

```kotlin
class WebViewActivity : AppCompatActivity() {
    private lateinit var webView: WebView

    private val onBackPressedCallback = object : OnBackPressedCallback(false) {
        override fun handleOnBackPressed() {
            if (webView.canGoBack()) {
                webView.goBack()
            }
        }
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_webview)
        webView = findViewById(R.id.webview)
        webView.webViewClient = MyWebViewClient(
            onPageStarted = {
                // 顯示載入中
            },
            onPageFinished = {
                // 隱藏載入中
            },
            updateCanGoBackAndForward = { canGoBack, _ ->
                // 設定 onBackPressedCallback 是否啟用
                // 如果 WebView 沒有頁面可以返回就要停用我們的 callback，這樣就能 finish Activity
                onBackPressedCallback.isEnabled = canGoBack
            },
        )
        webView.webChromeClient = MyWebChromeClient(
            onProgressChanged = { newProgress ->
                // 設定進度條的進度
            },
        )
        onBackPressedDispatcher.addCallback(onBackPressedCallback)
    }
}
```

```kotlin
class MyWebViewClient(
    private val onPageStarted: (url: String) -> Unit = {},
    private val onPageFinished: (url: String) -> Unit = {},
    private val onReceivedError: (request: WebResourceRequest, error: WebResourceErrorCompat) -> Unit = { _, _ -> },
    private val onReceivedHttpError: (request: WebResourceRequest, errorResponse: WebResourceResponse) -> Unit = { _, _ -> },
    private val updateCanGoBackAndForward: (canGoBack: Boolean, canGoForward: Boolean) -> Unit = { _, _ -> },
) : WebViewClientCompat() {
    override fun onPageStarted(view: WebView, url: String, favicon: Bitmap?) {
        super.onPageStarted(view, url, favicon)
        onPageStarted(url)
        updateCanGoBackAndForward(view.canGoBack(), view.canGoForward())
    }

    override fun onPageFinished(view: WebView, url: String) {
        super.onPageFinished(view, url)
        onPageFinished(url)
        updateCanGoBackAndForward(view.canGoBack(), view.canGoForward())
    }

    override fun onReceivedError(view: WebView, request: WebResourceRequest, error: WebResourceErrorCompat) {
        super.onReceivedError(view, request, error)
        onReceivedError(request, error)
        updateCanGoBackAndForward(view.canGoBack(), view.canGoForward())
    }

    override fun onReceivedHttpError(view: WebView, request: WebResourceRequest, errorResponse: WebResourceResponse) {
        super.onReceivedHttpError(view, request, errorResponse)
        onReceivedHttpError(request, errorResponse)
        updateCanGoBackAndForward(view.canGoBack(), view.canGoForward())
    }
}
```


```kotlin
class MyWebChromeClient(
    val onProgressChanged: (newProgress: Int) -> Unit = {},
) : WebChromeClient() {
    override fun onProgressChanged(view: WebView, newProgress: Int) {
        super.onProgressChanged(view, newProgress)
        onProgressChanged(newProgress)
    }
}
```

## 參考

- [Build web apps in WebView](https://developer.android.com/develop/ui/views/layout/webapps/webview)
- [使用 AndroidX 增强 WebView 的能力](https://juejin.cn/post/7259762775365320741)
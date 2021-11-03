---
title: "2021 iThome 鐵人賽 Day 22：Whistle proxy"
tags:
  - 2021 iThome 鐵人賽
  - Android
date: 2021-10-07T00:00:00+08:00
canonicalURL: https://ithelp.ithome.com.tw/articles/10278714
---

> 本篇文章是 [2021 iThome 鐵人賽](https://ithelp.ithome.com.tw/2021ironman)參賽題目「[寫一個列車抵站時間 Android App](https://ithelp.ithome.com.tw/users/20139666/ironman/4661)」的第 22 篇，你可到 iThome [查看原文](https://ithelp.ithome.com.tw/articles/10278714)。
>
> [文章目錄]({{< ref "2021-ithome-ironman" >}})

由於我們在上一篇已經完成了成功載入班次的部分，接下來要做的當然是不正常的情況。雖然港鐵間中會有事故，但都可遇不可求。要檢查我們做的東西是不是正確除了寫自動化測試之外，既然我們都做到 UI 的部分那就當然要直接看實物更好吧。所以我們先換一換題目討論 proxy server。

Proxy server 是我們常用的工具，我們能用 proxy server 檢查 app 的 HTTP request（這個之前介紹的 Flipper 都做到），亦能改變 response 令它變成我們想要的東西。這就可以令我們的 app 顯示到事故、延誤等畫面。這次我們會以 [Whistle](https://wproxy.org/whistle/) 作示範，特色是它是免費而且是跨平台都能使用。其實市面上有很多選擇，例如 [Charles](https://www.charlesproxy.com/)、[Proxyman](https://proxyman.io/)、[Fiddler](https://www.telerik.com/fiddler) 等等，用法都是大同小異。例如如何安裝根證書、系統或瀏覽器設定 proxy server 都是跟平台（例如 Android、iOS、Windows、macOS 等等）而不是跟 proxy server。而 proxy server 部分只需要知道其中一個 proxy server 檢視穿過 proxy server 的 HTTP traffic 以及何如改變 HTTP request 和 response 就應該很容易掌握到其他 proxy server 的用法。

## 安裝 Whistle proxy server

Whistle 是用 Node 寫的，所以要先在系統安裝 [Node](https://nodejs.org/en/) 和 [NPM](https://www.npmjs.com/)。如果你用 Node 比較多的話，建議經 NVM 安裝（[POSIX 版](https://github.com/nvm-sh/nvm)、[Windows 版](https://github.com/coreybutler/nvm-windows)）。安裝 Whistle 的方法就是用 NPM 安裝：

```bash
npm install -g whistle
```

## 啟動及終止 Whistle proxy server

安裝後，可以在 terminal 用 `w2` 指令啟動 Whistle：

```text
PS C:\Users\Eric> w2 start
[i] whistle@2.7.23 started
[i] 1. use your device to visit the following URL list, gets the IP of the URL you can access:
       http://127.0.0.1:8899/
       http://192.168.1.207:8899/
       http://192.168.98.1:8899/
       http://192.168.198.1:8899/
       Note: If all the above URLs are unable to access, check the firewall settings
             For help see https://github.com/avwo/whistle
[i] 2. configure your device to use whistle as its HTTP and HTTPS proxy on IP:8899
[i] 3. use Chrome to visit http://local.whistlejs.com/ to get started
PS C:\Users\Eric>
```

它會顯示好幾個網址，這個就是 Whistle 網頁界面的網址。Proxy server 的 IP 和 port 就是這些網址的 IP 和 port。你可以視乎網絡界面選用對應的 IP。

要停止 Whistle proxy server，只需使用 `w2 stop`：

```text
PS C:\Users\Eric> w2 stop
[i] whistle killed.
PS C:\Users\Eric>
```

如果是用 Windows 10 的話，可能會因為 execution policy 限制而無法啟動 Whistle，遇到這個情況可以參閱 PowerShell 的 [about_Execution_Policies 文檔](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_execution_policies)更改設定。

## 在 Android 安裝根證書

由於 HTTPS 是加密的，要讓 proxy server 看到傳輸的內容就要先為裝置安裝根證書。首先用裝置的瀏覽器開啟剛才 `w2 start` 看到的網址。

然後按頂部選單的「HTTPS」，就會開到下面的頁面。只要按下「Download RootCA」就能下載根證書。

留意 Capture TUNNEL CONNECTs 有沒有剔選，要剔選 Whistle 才能看到 HTTPS 的 traffic。

{{< figure src="root-cert-qr.png" title="Whistle 下載根證書的地方" >}}

然後到系統設定 > Security > Advanced > Encryption & credentials > Install a certificate。之後選取 CA certificate。

{{< figure src="install-a-cert.png" title="安裝根證書" >}}

之後會看到一個警告畫面，點選 Install anyway，再選取之前下載的 crt 檔就能安裝。

{{< figure src="warning.png" title="警告畫面" >}}

## 設定 proxy server

安裝根證書之後就可以到 Wi-Fi 設定，Proxy 選取 Manual 就會彈出幾個文字框讓你填寫 Proxy hostname 和 Proxy port。只要填上 `w2 start` 看到的 IP 和 port 就可以了。留意 127.0.0.1 那個是 localhost，在 Android 裝置填這個幾乎肯定是不能用的，填完後按 Save。

{{< figure src="wifi-proxy.png" title="Wi-Fi 設定" >}}

之後可以試試用瀏覽器載入一些 HTTPS 網站，試試能不能正常載入，並且留意 Whistle 網頁界面有沒有東西彈出來，如果有就代表設定完成。

當用完後緊記還原這個設定，否則 Whistle 結束後 Wi-Fi 就不能用。

## 查閱 HTTP request

只需要在左邊的導航欄選取 Network 就能查閱穿過 proxy server 的 HTTP traffic。點擊想看的 request，右邊上下兩格就會顯示該筆紀錄的 request 和 response。

{{< figure src="network.png" title="Network 部分" >}}

最底深灰色的輸入欄是用來篩選 HTTP traffic，可以輸入一些字眼來令上面只顯示想查看的 HTTP traffic。

## 改變 HTTP request/response

剛才的查閱是最基本的功能，現在看看進階一點的功能：把 HTTP request/response 變成自己想要的效果。這個功能主要用到的情景有：

1. backend 未做完，但 client side（例如 Android app）想試試自己那邊接駁 backend 的效果
2. client side 想試一些難出現的效果或者情景，而這些效果或者情景是跟據 backend 的 response 決定

首先，我們要切換到 Rules 界面（在左邊導航欄），你會看到一個藍色的畫面。這個是用來輸入改變 request/response 規則的地方。在左邊導航欄和藍色輸入規則的地方之間有一欄，這個是規則分組欄。你可以把它理解為規則是放在多個檔案（分組）內，你可以分別啟用和停用某些檔案所寫的規則。預設已經為你準備了一個空白的 Default 檔案，你可以把規則分門別類地放，這樣就容易整理。Default 旁邊有個綠色剔號，意思是它目前生效中，點兩下就可以把它停用。

{{< figure src="rules.png" title="Rules 部分" >}}

現在先看看最簡單的一個寫法：把整個網站轉去另一個網站。以下是把 [google.com](http://google.com) 轉址去 apple.com：

```text
google.com apple.com
```

填好後要按 Ctrl + S 或者上方的 Save 才會生效。

如果你在瀏覽器輸入 `google.com/hi`，你就會看到它會轉址到 `apple.com/hi`。如果切換到 Network 看的話，proxy server 是做了 HTTP 301 轉址。

如果是 local 都有 server，想把 app 由 production server 轉駁去自己的 server，可以這樣寫：

```text
example.com 127.0.0.1
```

如果已經預先準備好 response 檔案的話，可以直接指向本地檔案：

```text
# macOS, Linux
example.com file:///User/foo/bar.json
# Windows
example.com file://D:\foo\bar.json
```

另一個做法是把內容放到 Whistle 的 values，然後引用它。首先在左邊導航欄點選 Values，這次的界面跟 Rules 有點像。先按 Create 並填上檔案名稱 delay.json。然後在藍色背景的編輯器填上內容：

```json
{
  "status": 0,
  "message": "Special train service arrangements are now in place on this line. Please click here for more information.",
  "url": "https://www.mtr.com.hk/alert/alert_title_wap.html",
  "curr_time": "2019-06-13 17:34:58"
}
```

{{< figure src="values.png" title="Values 部分" >}}

填好後按 Ctrl + S 或者上方的 Save 儲存。

然後切換到 Rules 並寫上我們要的 rule：

```text
https://rt.data.gov.hk/v1/transport/mtr/getSchedule.php resBody://{incldent.json}
```

這樣每次 call endpoint 都會回傳事故，這樣就可以人工測試到 app 能否順利因應 response 內容顯示對應的畫面。

要留意一樣東西是用了 `resBody://` 操作符雖然改了 response body，但其實還是會把 request 送上 server，如果那個 request 是用來建立或者刪除內容的話就可能會影響到 backend 背後的 database 內容。如果要防止把 request 送上原本的 server 的話，其中一個方法是轉用 `file://` 操作符，另一個方法是額外加上 `status://` 操作符：

```text
https://rt.data.gov.hk/v1/transport/mtr/getSchedule.php status://200 resBody://{incldent.json}
```

同一個網址可以套上多個操作符，其實開首的網址都不一定是網址，實際上是匹配規則 (pattern)。當它匹配後 Whistle 就會為那個 request 套上後面的操作符。匹配除了用普通的網址外，還可以用 regular expression，詳細可以參考 [Whistle 文檔有關匹配模式的部分](https://wproxy.org/whistle/pattern.html)。

除了改 status code 和 response body 之外，Whistle 還可以改 response header 和延長 response 回傳時間：

```text
https://rt.data.gov.hk/v1/transport/mtr/getSchedule.php statusCode://200 resDelay://4000 resHeaders://{resHeaders.txt} resBody://{incident.json}
```

`resDelay://4000` 意思是把 response 傳回去的時間延長多四秒，而 `resHeaders://{resHeaders.txt}` 的意思是 response header 換成放在 values 的 resHeaders.txt 的內容。下面是 resHeaders.txt 的內容：

```text
Content-Type: application/json
```

## 快速開關 Android proxy 設定

先前介紹從 Wi-Fi 設定改變 proxy 設定，但步驟繁複。如果想簡單一點可以用 [Proxy Toggle](https://github.com/theappbusiness/android-proxy-toggle) 這個 app。它提供了 widget 讓你放到主畫面一鍵開關 proxy 設定。但安裝和移除 app 要用到 adb（因為要改變系統設定，但不用 root）。特別留意在移除 app 前必須執行它提供的 adb 指令確保系統 proxy 設定被還原（因為不能從系統 UI 變更）。

{{< figure src="proxy-toggle.jpg" title="Proxy Toggle widget" >}}

## Android app 允許用戶加入的證書

自 Android 7 (API 24) 開始，app 可以在 manifest 的 `<application>` 加上 `android:networkSecurityConfig` 設定是否允許非 HTTPS traffic 和信任的證書以加強保安。由於我們用了自行簽發的證書，所以要先更改預設設定才能用 proxy server 檢閱 app 的 HTTPS traffic。首先在 manifest 的 `<application>` 加上 `android:networkSecurityConfig`：

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="net.swiftzer.etademo">

    <uses-permission android:name="android.permission.INTERNET" />

    <application
        android:name=".EtaDemoApp"
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:networkSecurityConfig="@xml/network_security_config"
        android:theme="@style/Theme.ETADemo">
        <!-- 略 -->
    </application>
</manifest>
```

然後分別在 debug 和 main 的 source set 加上 *network_security_config.xml*。

首先是 debug 版，下面的 XML 路徑是 *app/src/debug/res/xml/network_security_config.xml*。

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config xmlns:tools="http://schemas.android.com/tools">
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system" />
            <certificates
                src="user"
                tools:ignore="AcceptsUserCertificates" />
        </trust-anchors>
    </base-config>
</network-security-config>
```

首先是 main (release) 版，下面的 XML 路徑是 *app/src/main/res/xml/network_security_config.xml*。

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <base-config cleartextTrafficPermitted="false">
        <trust-anchors>
            <certificates src="system" />
        </trust-anchors>
    </base-config>
</network-security-config>
```

這樣就可以在 debug 版的 app 用 proxy server 了。

## 小結

本篇示範了用 Whistle proxy server 做一些簡單的 request/response 處理和在 Android app 允許非系統預載的證書。這樣在下一篇繼續做班次頁的各式錯誤頁時就能用 proxy server 改變 response 來測試 app 的效果。此外，proxy server 是 frontend（包括 web 和 mobile）常用工具。了解 proxy server 基本用法能方便日常的開發（例如在陌生的 project 可以透過觀察 HTTP traffic 然後搜尋到對應的 code 位置）。Whistle 有太多功能，這次只是介紹了它的皮毛，詳細的用法還是需要參閱[完整的文檔](https://wproxy.org/whistle/)。

---
title: TrainBoard for Android TV
tags:
  - Android
date: 2017-06-26T00:03:49+08:00
---


之前買了一個[小米盒子國際版]({{< ref "posts/xiaomi-mi-box" >}})，一直都想用它來寫一些 Android TV app。到了最近才將我第一個 Android TV app [TrainBoard](https://play.google.com/store/apps/details?id=net.swiftzer.trainboard) 上架，這個 app 亦都是我第一個用 [Kotlin]({{< ref "posts/kotlin-for-android" >}}) 寫的 Android app。TrainBoard 提供港鐵將軍澳綫、東涌綫、機場快綫、迪士尼綫及西鐵綫列車的預計到達時間，而這個 app 是為 [MTR Service Update](https://checkfare.swiftzer.net/) 而做的。

{{< figure src="play-store.jpg" title="TrainBoard on Google Play Store" >}}

<!-- more -->

{{< figure src="store-search.jpg" title="在 Android TV 的 Play Store 搜尋「TrainBoard」就會找到" >}}

這個 app 用法其實很簡單，只需要在主頁揀選車站，再揀選方向，你的電視就會變成好像車站月台的顯示屏般顯示下五班車的預計到站時間。如果因為空間不夠而看不到五筆資料的話，可以按遙控的 D-pad（方向掣）。

{{< figure src="main-page.png" title="主頁" >}}

{{< figure src="eta-lop.png" title="Live ETA" >}}

## 上架過程

其實這篇文除了介紹 TrainBoard 之外，還想講一下上架的過程。在 Google Play Store 把 app 上架向來都是不用審批，不過自從有了 Android TV 和 Android Wear 之後，Android TV 和 Android Wear app 就要先經過審批才能真正上架。以 Android TV 為例，不送審其實都可以把 Android TV 上架，但只限於網頁版，Android TV 上的 Google Play Store 是找不到這個 app（即是只可以經網頁版下載安裝）。這亦都解釋了為甚麼 Android TV 上的 Google Play Store 只有很少 app 可以下載。

TrainBoard 其實在上星期日交上 Play Store，到了星期二出了審批結果被拒上架。原因是沒有 [full-size app banner](http://developer.android.com/training/tv/start/start.html#banner)。

> We are targeting 1080P, which we consider xhdpi. Apps should include the banner in the xhdpi (320 dpi) drawables folder with a size of (320px × 180px).

不過我檢查過 *drawable-xhdpi* 確實有合符尺寸的 banner 圖，而 manifest 亦都有在 `<application>` 加上 `android:banner` attribute。

{{< figure src="banner.png" title="未能通過審批的 banner" >}}

之後我填表格上訴，然後他們回覆我被拒的原因不是尺寸問題，而是背景顏色問題。

> I understand your app recently wasn't approved for Android TV recently for a banner issue. Looking  into it it appears that the issue may have been misrepresented on our side a little. The issue is the size because the banner is a gray box. If you could make the box white or green, etc. where it sticks out more I believe that will solve your issue. A gray box isn't accepted because it blends in with the home page too much. Once this is updated please resubmit your app and we will gladly review it again for you.
>
> — <cite>James H, Google Play Developer Support</cite>

看完回覆之後馬上改一改背景顏色，果然第二日（星期四）就能順利上架。

{{< figure src="launcher.jpg" title="順利上架的 app banner" >}}

## 只限 Android TV？

是的，因為 TrainBoard 只用了 [Leanback library](https://github.com/googlesamples/androidtv-Leanback) 來做 UI，沒有為手機和平版設計過另一套 UI，即使安裝到手機或平版上效果都不理想（有些地方要用 D-pad 才能操作），所以就不讓手機和平版使用這個 app。但很快就會將 TrainBoard 移植到 Android 手機和平版上使用，請耐心等候，或者用[網頁版](https://checkfare.swiftzer.net/trainboard.php)也可以。

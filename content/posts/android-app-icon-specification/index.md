---
title: Android App Icon 規格
tags:
  - Android
date: 2019-07-20T13:43:37+08:00
---


一個 app 的第一印象應該是它的 app icon (launcher icon)。在 Android，在不同的時期前後出了好幾個 app icon 規格。但是 Android 介紹不同種類的 app icon 的文件放得非常分散，如果平時沒有一直留意的話都幾乎肯定會做錯或者做漏。以下是全部 Android app 都會用到 app icon：

- Launcher icon
- Launcher icon（圓形）
- Adaptive launcher icon
- Google Play icon

如果想簡單地做出全部 icon 的話，可以用 Android Studio 內置的 [Image Asset Studio](https://developer.android.com/studio/write/image-asset-studio)。但如果想追求完美的話，還是自行準備圖片比較好。這篇文章整理了不同 icon 的基本規格，寫的時候盡量考慮到設計師。如果有需要的話可以分享給設計師同事參考。

{{< figure src="android-studio-image-asset.png" title="Android Studio 的 Image Asset Studio" >}}

<!-- more -->

## 背景知識

### 屏幕密度

因為 Android 設備有太多款，所以 Android 在處理不同屏幕方面比起 iOS 更細緻：由最基本的屏幕密度到控制在不同屏幕尺寸下的界面排版都比 iOS 有更多設置。如果是設計 app icon 的話，主要留意的是屏幕密度。iOS 目前有[三款密度](https://developer.apple.com/design/human-interface-guidelines/ios/icons-and-images/image-size-and-resolution/)：1x、2x (`@2x`) 和 3x (`@3x`)﹔而 Android 的[密度](https://developer.android.com/training/multiscreen/screendensities#TaskProvideAltBmp)就有 1x (MDPI)、1.5x (HDPI)、2x (XHDPI)、3x (XXHDPI) 和 4x (XXXHDPI)。建議大家使用向量圖準備原檔，並且以 1x 的尺寸準備圖片。在匯出圖片時才一次過匯出全部密度的 PNG 圖片。

在 Android，一般會使用 dp 這個長度單位來設計 UI。如果不清楚 dp 是甚麼意思的話，可以參考 Material Design 的 [Pixel density 部分](https://material.io/design/layout/pixel-density.html)。但在高密度的屏幕下，如果要把圖案顯示到和低密度屏幕一樣大小（把間尺放在屏幕上量度那個圖案）而又不模糊的話，是要用更多的像素 (pixel, px)。所以就發明了 dp 這個單位來描述尺寸，即使在不同密度的屏幕下 dp 都是一樣。簡單來說，在 1x 的情況下，1dp = 1px。你要在 1x 的情景下繪畫你的圖案，然後按照上一段的 1.5x、2x、3x……倍數匯出成不同尺寸的圖片。

舉個例子：如果要準備一張 32 × 32dp 的圖片，你要匯出以下的 PNG 檔案：

|密度|放大倍數|PNG 圖片闊度及高度 (px)|
|---|---|---|
|MDPI|1x|32|
|HDPI|1.5x|48|
|XHDPI|2x|64|
|XXHDPI|3x|96|
|XXXHDPI|4x|128|

PNG 檔案名稱需要一致，只可以用小楷英文字母、底線 (`_`) 和數字（數字不可在作為名稱開首），並將圖片放到對應的資料夾內（例如 `drawable-hdpi`）。

### 主畫面

Android 的主畫面是叫作 launcher，它的功能除了主畫面外，還包括讓用戶選擇要啟動的 app（app 列表）。Launcher 本身是一個 app。和其他 app 一樣，用戶可以在 Google Play 下載其他 launcher 來替代預設的 launcher。不同的 launcher 在顯示 app icon 方面都有不同的效果。**簡單來說，app icon 顯示效果取決於 Android 版本和用戶所選用的 launcher。**

## Launcher Icon

這個是最基本的 app icon，就是 Android 最先出現的 app icon 形式。稱為 launcher icon 就是因為它主要在 launcher 內顯示。（其實也會在設定頁的應用程式清單中看到）

和 [iOS](https://developer.apple.com/library/archive/qa/qa1686/_index.html) 不同，Android 的 launcher icon 是沒有規定形狀。在 [Android 4 時代的指引（鏡像網站）](https://stuff.mit.edu/afs/sipb/project/android/docs/guide/practices/ui_guidelines/icon_design_launcher.html)是鼓勵 icon 的形狀要有特色，不要套上圓角方形之類的遮色片 (clipping mask) 或完全填滿全部像素來侷限 icon 設計。但太多人受到 iOS 的影響，都把 launcher icon 做成像 iOS 般的圓角方形。有些甚至用盡 launcher icon 尺寸，沒有按規格在四邊留足夠空間來加上陰影。結果不同款色的 launcher icon 放在同一個 launcher 時視覺完全不協調。一些 Android 設備品牌為了解決這個問題，在它們的預設 launcher 會做到像 iOS 般為所有 app icon 加上圓角方形的遮色片。亦有一些 launcher 會為所有 app icon 加上托底，令原先不同形狀的 app icon 看上來都統一。加上 Google Play Store 上架無需人工審批，即使 launcher icon 設計不符指引也沒有問題。結果直到現在看到的 app icon 設計都是不統一。

在後期的 Android 版本為了解決這個問題先後出了好幾款的 app icon 形式（就是之後介紹那些）。但這個 launcher icon 形式仍是必須的。

規格方面，尺寸是 48 × 48dp。有關設計原則、格線、形狀、高光、陰影之類可以直接查閱 [Material Design 網站](https://material.io/design/iconography/product-icons.html)。Illustrator 範本可在舊版 Material Design 網站的 [Resources – Sticker sheets & icons](https://material.io/archive/guidelines/resources/sticker-sheets-icons.html#sticker-sheets-icons-components)（Product icons 部分）下載，那些 AI 檔案已經包含了格線、基本形狀、高光、陰影的示範。

{{< figure src="ic_launcher.png" title="Launcher icon 連格線示範" >}}

不過要留意一點：這個範本的尺寸是 192 × 192pt，而不是 48 × 48px。因為 launcher icon 指引建議大家放大 400% 來設計（但需要以 4pt 的格線來繪制），然後在匯出時是縮小成不同的 PNG 檔案而不是像平時般放大圖片。

完成設計後按以下的規格匯出：

|闊度及高度 (px)|密度|放大倍數|
|---|---|---|
|48|MDPI|0.25x|
|72|HDPI|0.375x|
|96|XHDPI|0.5x|
|144|XXHDPI|0.75x|
|192|XXXHDPI|1x|

把匯出的 PNG 檔案按以下的目錄結構放好：

```text
.
├── mipmap-hdpi
│   └── ic_launcher.png (72 × 72px)
├── mipmap-mdpi
│   └── ic_launcher.png (48 × 48px)
├── mipmap-xhdpi
│   └── ic_launcher.png (96 × 96px)
├── mipmap-xxhdpi
│   └── ic_launcher.png (144 × 144px)
└── mipmap-xxxhdpi
    └── ic_launcher.png (192 × 192px)
```

*AndroidManifest.xml* 會是這樣：

{{< highlight xml >}}
<application
    android:icon="@mipmap/ic_launcher">
    <!-- 略 -->
</application>
{{< /highlight >}}

## Round Launcher Icon

圓形 app icon 是 [Android 7.1](https://developer.android.com/about/versions/nougat/android-7.1.html#circular-icons) 新增的 launcher icon 式樣。如果你本身的 launcher icon 不是圓形的話，可以在 *AndroidManifest.xml* 用 `android:roundIcon` 設定圓形 app icon。會否使用圓形 icon 視乎 launcher 而定。

設計範本都是用剛才 launcher icon 的範本，裏面有圓形格線及形狀可供參考使用。完成後同樣使用上一部分的規格匯出，並將 PNG 檔案按以下的目錄結構放好：

```text
.
├── mipmap-hdpi
│   └── ic_launcher_round.png (72 × 72px)
├── mipmap-mdpi
│   └── ic_launcher_round.png (48 × 48px)
├── mipmap-xhdpi
│   └── ic_launcher_round.png (96 × 96px)
├── mipmap-xxhdpi
│   └── ic_launcher_round.png (144 × 144px)
└── mipmap-xxxhdpi
    └── ic_launcher_round.png (192 × 192px)
```

*AndroidManifest.xml* 會是這樣：

{{< highlight xml >}}
<application
    android:roundIcon="@mipmap/ic_launcher_round">
    <!-- 略 -->
</application>
{{< /highlight >}}

## Adaptive Launcher Icon

這個由 Android 8 推出的 icon 形式官方中文譯名為「[自動調整圖示](https://developer.android.com/guide/practices/ui_guidelines/icon_design_adaptive?hl=zh-tw)」，可以說是解決各大廠商自行改變 launcher icon 形狀的終極方案：就是讓 launcher 決定 icon 的形狀，不一定是圓角方形，也不一定是圓形。但規格有訂明保證能被顯示的範圍（一個直徑 66dp 的圓形），範圍以外視作出血位，有機會被 launcher 裁走。而 icon 外圍的陰影都會由 launcher 負責。這其實和 iOS 有幾分相似，但它最大特色是 icon 圖片不是一張圖片，是兩張圖片（前景和背景）。

*AndroidManifest.xml* 如果要使用 adaptive launcher icon 就是和之前的沒有分別：

{{< highlight xml >}}
<application
    android:icon="@mipmap/ic_launcher"
    android:roundIcon="@mipmap/ic_launcher_round">
    <!-- 略 -->
</application>
{{< /highlight >}}

但是特別之處是在 *mipmap-anydpi-v26* 資料夾加入 *ic_launcher.xml* 和 *ic_launcher_round.xml*。兩個 XML 檔案內容是一樣的：

{{< highlight xml >}}
<?xml version="1.0" encoding="utf-8"?>
<adaptive-icon xmlns:android="http://schemas.android.com/apk/res/android">
    <background android:drawable="@mipmap/ic_launcher_background"/>
    <foreground android:drawable="@mipmap/ic_launcher_foreground"/>
</adaptive-icon>
{{< /highlight >}}

準備兩張 108 × 108dp 圖片：*ic_launcher_foreground.png* 和 *ic_launcher_background.png*，分別是前景圖和背景圖。前景要有透明底色，而背景圖就不可以有透明底色。範本可以在 [Google Design 的 Medium 文章](https://medium.com/google-design/designing-adaptive-icons-515af294c783)內找到。留意這個格線和先前那些範本不同，所以需要再調整一下圖案。

{{< figure src="foreground-with-guidelines.png" title="前景圖連格線示範" >}}

{{< figure src="background-with-guidelines.png" title="背景圖連格線示範" >}}

如果連同先前介紹的 icon 規格的話，要準備好以下的檔案：

```text
.
├── mipmap-anydpi-v26
│   ├── ic_launcher.xml
│   └── ic_launcher_round.xml (Same content as ic_launcher.xml)
├── mipmap-hdpi
│   ├── ic_launcher.png (Launcher icon, 72 × 72px)
│   ├── ic_launcher_background.png (Adaptive launcher icon background, 162 × 162px)
│   ├── ic_launcher_foreground.png (Adaptive launcher icon foreground, 162 × 162px)
│   └── ic_launcher_round.png (Round launcher icon, 72 × 72px)
├── mipmap-mdpi
│   ├── ic_launcher.png (Launcher icon, 48 × 48px)
│   ├── ic_launcher_background.png (Adaptive launcher icon background, 108 × 108px)
│   ├── ic_launcher_foreground.png (Adaptive launcher icon foreground, 108 × 108px)
│   └── ic_launcher_round.png (Round launcher icon, 48 × 48px)
├── mipmap-xhdpi
│   ├── ic_launcher.png (Launcher icon, 96 × 96px)
│   ├── ic_launcher_background.png (Adaptive launcher icon background, 216 × 216px)
│   ├── ic_launcher_foreground.png (Adaptive launcher icon foreground, 216 × 216px)
│   └── ic_launcher_round.png (Round launcher icon, 96 × 96px)
├── mipmap-xxhdpi
│   ├── ic_launcher.png (Launcher icon, 144 × 144px)
│   ├── ic_launcher_background.png (Adaptive launcher icon background, 324 × 324px)
│   ├── ic_launcher_foreground.png (Adaptive launcher icon foreground, 324 × 324px)
│   └── ic_launcher_round.png (Round launcher icon, 144 × 144px)
└── mipmap-xxxhdpi
    ├── ic_launcher.png (Launcher icon, 192 × 192px)
    ├── ic_launcher_background.png (Adaptive launcher icon background, 432 × 432px)
    ├── ic_launcher_foreground.png (Adaptive launcher icon foreground, 432 × 432px)
    └── ic_launcher_round.png (Round launcher icon, 192 × 192px)
```

設定好之後，視乎 launcher app 的設定，你的 adaptive icon 可能會被套用不同形狀的遮色片。部分 launcher 更會在用戶用手左右掃主畫面和按下 icon 時有動畫效果（以不同的速度移動 icon 的前景和背景圖片、放大縮小前景和背景圖片之類）。

{{< figure src="adaptive-animation.gif" title="動畫示範" >}}

因為 adaptive launcher icon 是一樣比較新的 icon 形式，其實是可以用 [vector drawable](https://developer.android.com/guide/topics/graphics/vector-drawable-resources) 而不需要準備一堆 PNG 檔案（雖然是 XML，但不是 SVG 格式，vector drawable 不支援所有 SVG 的功能）。但 vector drawable 是另一個大課題，如果同時講解 vector drawable 可能會令設計師看不懂，而且篇幅太長。所以這篇文章全部使用 PNG 作示範。

## Google Play Icon

{{< figure src="google-play-icon-sample.png" title="Google Play icon 例子" >}}

這個 icon 其實是不會放進 app 入面，只是用於 Google Play 上架。即是在 Google Play 看到的 app icon。尺寸只有一個：512 × 512px。這個 icon 的規格比其他 icon 相對簡單，簡單來說就是和 iOS app icon 差不多。都是 PNG 格式，只是圓角不一樣：圓角半徑為圖示大小的 20% (102.4px)。

另外，圖片不可以有透明背景。亦不用為 icon 加上圓角外框，因為 Google Play 網頁和 app 會即場為你的 app icon 裁圓角和加上陰影。

{{< figure src="google-play-icon-new.png" title="Google Play Console 的新式 app icon" >}}

這個 icon 規格是在近期（2019 年 6 月 24 日起）正式實行。如果還未轉用新規格的話 Google Play 會把現有的 icon 縮小到 75% 並置中在白色背景的圓角方形內（即舊版模式）。

{{< figure src="google-play-icon-legacy.png" title="Google Play Console 內的舊版模式示範" >}}

詳細的規格可以參閱 [Google Play 圖示設計規範](https://developer.android.com/google-play/resources/icon-design-specifications?hl=zh-tw)，該頁最底有 Sketch、Illustrator 和 Photoshop 範本，範本內有格線可供參考。雖然格線和 adaptive launcher icon 如果你的 app icon 圖像的關鍵部分是形狀的話，就應該按照格線調整形狀的大小和位置。如果純粹把整個 adaptive launcher icon 縮放到 512 × 512px 是錯誤的，應該要為 Google Play icon 重新調整 icon 內的圖形。

{{< figure src="play-store-icon.png" title="Google Play icon 連格線範例" >}}

## 其他 Icon

### Status Bar Icon

Android 和 iOS 不同的地方是頂部通知欄會顯示 icon，這個 icon 有不少人忽略。有些 app 做得不仔細的話會直接用 launcher icon。但效果會非常差，在 Android 5.0 (API level 21) 開始往往[只顯示一個圓角方形的白色形狀](https://stackoverflow.com/questions/30795431/android-push-notifications-icon-not-displaying-in-notification-white-square-sh)。按照 [Android 官方說法](https://material.io/design/platform-guidance/android-notifications.html)，status bar icon 是系統按照情況為 icon 填上白色、灰色、由 app 定義的 accent 色等等。以前曾聽過有些客戶要求 status bar icon 要以彩色顯示，其實這並不符合 Android 的要求，而且不能保證全部設備都能做到彩色效果。另一方面，由於 status bar icon 只有 24 × 24dp，所以不可能只把 launcher icon 縮細就當作 status bar icon 來用。正確做法是要把主體圖形抽出再作簡化，這樣用戶才能瞄一眼就知道這個通知是那一個 app 發出。

Android 開發者網站經過多次改版後已經找不到 status bar icon 的設計指引，只可以從[鏡像網站](https://stuff.mit.edu/afs/sipb/project/android/docs/guide/practices/ui_guidelines/icon_design_status_bar.html)看到當年 (Android 4.0) 年代的設計指引。自從 Android 4.0 之後 Android 都再沒有推出針對 status bar icon 的指引，所以這個鏡像網站所提的設計指引應可作現今的指引參考。同時亦可參照 [Material Design 的 system icons 部分](https://material.io/design/iconography/system-icons.html)來設計。因為 status bar icon 和 Material Design system icon 的尺寸都是 24dp，而且它提供比當年 Android 4.0 status bar icon 更清晰的設計指引。所以我覺得現時可以參考 Material Design 的指引來準備 status bar icon，但顏色方面只可以用白色 (`#FFFFFF`)，不要用半透明填色。

### TV Banner

如果要準備 Android TV app 的話，需要額外準備 TV banner 作為 Android TV launcher 的 app icon。引用自 [TV app quality](https://developer.android.com/docs/quality-guidelines/tv-app-quality) 中 TV-LB 一段：

> The app displays a 320 × 180px full-size banner as its launcher icon in the Android TV Launcher.

這個 320 × 180px 的 banner 就是在 XHDPI 的情況下的尺寸是 320 × 180px。如果換算成 dp 的話就是 160 × 91dp。

{{< figure src="banner.png" title="TV banner 範例" >}}

Banner 放在 `drawable` 資料夾就可以了，不用放在 `mipmap` 資料夾。如果講究的話可以準備不同 DPI 的圖片。但最少要有一張 320 × 180px。

以 MDPI 為基準繪製 160 × 91px 的圖片，把匯出的 PNG 檔案按以下的目錄結構放好：

```text
.
├── drawable-hdpi
│   └── banner.png (240 × 136px)
├── drawable-mdpi
│   └── banner.png (160 × 91px)
├── drawable-xhdpi
│   └── banner.png (320 × 180px)
├── drawable-xxhdpi
│   └── banner.png (480 × 271px)
└── drawable-xxxhdpi
    └── banner.png (640 × 360px)
```

*AndroidManifest.xml* 會是這樣：

```xml
<application
    android:banner="@drawable/banner" >
    <!-- 略 -->
</application>
```

其實這個 banner 有個潛規則：不可以使用黑色、深灰色之類的背景顏色。因為 Android TV 的 launcher 背景色是深色為主，如果 banner 背景顏色都是用深色的話就難以從背景色中分離出來。Banner 背景顏色太深色的話 Google 是不會讓你的 app 上架。（Android TV 新 app 上架和 iOS App Store 一樣都需要送檢，但之後的更新就不用送檢。）如果想了解更多的話可以參考我之前上架 [TrainBoard]({{< ref "posts/trainboard-for-android-tv" >}}) 的例子。

順帶一提，[Android TV design guidelines](https://designguidelines.withgoogle.com/android-tv/style/color.html) 提到在電視屏幕上不應使用純白色 (`#FFFFFF`)。因為太鮮色會刺眼。如果要用白色的話應該使用淺灰色 (`#EEEEEE`)。

## 參考及閱讀更多

- [Android Notification Icon 製作要點](https://medium.com/@binglu/android-notification-icon-%E8%A3%BD%E4%BD%9C%E8%A6%81%E9%BB%9E-ad50869f418f)<br>
  Notification icon 的規格，但 icon 應該要在在 `drawable` 資料夾而不是 `mipmap` 資料夾。
- [Getting Your Apps Ready for Nexus 6 and Nexus 9](https://android-developers.googleblog.com/2014/10/getting-your-apps-ready-for-nexus-6-and.html)<br>
  內有提及為何 launcher icon 要放入 `mipmap` 資料夾而不是 `drawable` 資料夾的原因。
- [Android Asset Studio](https://romannurik.github.io/AndroidAssetStudio/)<br>
  網上製作各式 icon 的工具，大部分功能經已內建在 Android Studio 內。
- [Philips Hue adaptive icon](https://jeroenmols.com/blog/2019/07/03/adaptiveicon/)<br>
  示範了如何準備 adaptive launcher icon。
- [Adapticon](https://adapticon.tooo.io/)<br>
  Adaptive launcher icon 效果示範和範例。
- [Understanding Android Adaptive Icons](https://medium.com/google-design/understanding-android-adaptive-icons-cee8a9de93e2)<br>
  介紹引入 adaptive launcher icon 的原因。
- [Designing Adaptive Icons](https://medium.com/google-design/designing-adaptive-icons-515af294c783)<br>
  設計 adaptive launcher icon 要留意的地方。
- [Image Resolution: PPI, DPI, Image Quality and Everything You Need to Know](https://www.videoproc.com/resource/image-resolution.htm)<br>
  介紹 PPI 跟 DPI 的分別。

---
title: 下載 Droid Sans 字型
tags:
  - CSS
  - Google IO
  - Web font
  - Typography
  - Android
date: 2011-05-11T21:43:20+08:00
---

[Droid Sans](http://www.droidfonts.com/) 是 [Android](http://www.android.com/) 使用者介面所使用的字型 ((雖然有部分手機製造商會自訂使用者介面的設計，但大部分的手機均使用 Droid Sans 作為介面字型。))，整個 Droid 字型家族是由 Ascender 的 Steve Matteson 設計。而 Droid 字型家族共分三個字型：

* Droid Sans (regular, bold)
* Droid Sans Mono (regular, bold)
* Droid Serif (regular, italic, bold, bold italic)
同時它亦有黑體中文字型：Droid Sans Fallback。特別之處是運用了 [DigiType Compact Asian 技術](http://www.ascendercorp.com/developers/displays/digitype/)，將一個包含大量中文字元的字型壓縮至約 3.5 MB 的檔案，非常適合放到手機使用。

正因為 Droid 字型家族是 Android 所使用的標準字型，在開發 Android Apps、Widget 的時候，很多時都會用縮圖軟件畫一些 Mockup 介面的圖像。如果有 Droid 字型的話就更能得心應手了。就算不用來設計 Android Apps，普通用都不錯。Google 的網站近期都用多了 Droid Sans，不論是 [Android Market](https://market.android.com/)、今天新開的 [Google Music Beta](http://music.google.com/)、[Google Apps](http://www.google.com/apps/) 還是 [Google I/O](http://www.google.com/events/io/2011/) 的網站，都可以找到 Droid Sans 的蹤影。不過包含中文字型的 Droid Sans Fallback 就因為同時包含繁簡體中文字元的關係，同一個字型入面不同字元的筆畫混合了中港台的寫法，美感不太好。而標點符號就採用了中國內地的規範，全部靠左下角，與香港及台灣要置中的做法有所不同。

要下載 Droid 字型家族到電腦使用，有幾個方法：

1. [到 Android 的 Git 下載最新版本](http://android.git.kernel.org/?p=platform/frameworks/base.git;a=tree;f=data/fonts;hb=HEAD)
2. [下載 Android 最新的 SDK](http://developer.android.com/sdk/index.html)
3. [下載經文泉驛修改的 Droid Sans Fallback——文泉驛微米黑](http://wenq.org/index.cgi?MicroHei)
如果要從 SDK 中尋找 Droid 字型的話，須下載 API Level 11 或以上的 SDK 版本（目前是有 API Level 11 的 Android 3.0 和 API Level 12 的 Android 3.1） ((舊版的 SDK 所提供的 Droid Sans Fallback 是不能夠在 Windows 上使用。))。使用 Windows 的話下載後可以到 _C:\Program Files\Android\android-sdk\platforms\**android-12**\data\fonts_ 找到字型（要替換路徑中的 _**android-12**_ 目錄成你下載的 API Level）。

另外，如果要在網頁使用的話，[Google Web Fonts](http://www.google.com/webfonts/) 亦有提供 [Droid Sans](http://www.google.com/webfonts/family?family=Droid+Sans)、[Droid Serif](http://www.google.com/webfonts/family?family=Droid+Serif) 及 [Droid Sans Mono](http://www.google.com/webfonts/family?family=Droid+Sans+Mono) 的 Web font 供網頁調用。只要在網頁中加載由 Google Web Fonts 提供的指定 CSS 並於網頁的 CSS 檔中設定好那些部分會用到該字型就能調用了。

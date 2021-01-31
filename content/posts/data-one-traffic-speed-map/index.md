---
title: Data.One 的行車速度圖
date: 2014-09-25T01:00:51+08:00
tags:
  - Data.One
  - Open data
---

香港政府的 [Data.One](http://www.gov.hk/tc/theme/psi/datasets/) 有提供運輸署的[行車速度圖](http://www.gov.hk/tc/theme/psi/datasets/speedmap.htm) XML 資料。資料以線條為單位，表示路段的行車速度。雖然它有提供 Excel 格式的路段起點和終點座標和其他基本資料，但郤沒有提供地圖示意各路段是對應那條實際的道路。而且座標系統是使用政府測量用的 HK80 而非一般網上地圖 API 的十進制 WGS84 座標。如果想使用這些資料的話，就要先將 HK80 座標轉為 WGS84，然後將點和線畫在地圖上，再對比實際的道路才能知道各條線所代表的道路。

我已經將行車速度圖 Excel 內的點轉成 WGS84 座標，而且做了個 KML 檔案標示各點和線。KML 檔可以用 [Google Earth](https://www.google.com/earth/) 開啟。（KML 內的物件已超出 Google Maps My Place 匯入 KML 檔的上限，所以要用 Google Earth 開啟。）

{{< figure src="td2.jpg" title="行車速度圖的路段" >}}

* [各點的 WGS84 座標 CSV 檔](https://gist.github.com/ericksli/4d5a01d105a7371b2b6c)
* [各點和線的 KML 檔](https://gist.github.com/ericksli/37e744d479a87b14081d)

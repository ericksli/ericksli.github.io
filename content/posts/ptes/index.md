---
title: 公共交通查詢服務 (PTES)
tags:
  - Google
  - PTES
  - 地圖
date: 2009-04-30T20:04:41+08:00
---
運輸署在 4 月 28 日下午 3 時推出了「[公共交通查詢服務 (Public Transport Enquiry Service, PTES)](http://ptes.td.gov.hk/)」網站。網站由運輸署和理工大學開發。我當日嘗試去使用這個網站，但不時出現錯誤：

> Microsoft OLE DB Provider for SQL Server error '80040e31'
>
> Timeout expired
>
> /tc/search_text/routelist_info.asp, line 134
> An error occurred on the server when processing the URL. Please contact the system administrator.
> 請查看地圖，把附近最合適的車站訂為出發地 / 目的地再搜尋。或把路程分段以減少轉乘次數。

<!--more-->

{{< figure src="ptes-ui.jpg" title="公共交通查詢服務 (PTES) 的界面" >}}

不過那天都試用到的（使用打字的方式來標示出發地和目的地），試用過幾次之後，發現了這些問題：

1. 地圖載入速度太慢
那個地圖功能最好都不要用，因為真是很慢，等了很久還未載入到地圖。地圖的瀏覽方式還是和舊版中原地圖的用法差不多，未有跟隨類似 [Google 地圖](http://maps.google.com.hk/) / [Windows Live Local](http://maps.live.com/) 的使用和載入方式，導致在瀏覽時都花不少時間來載入地圖。
2. 在輸入地名搜尋後，會出現同名的項目
出現同名項目時，逐一查看地圖，發現原來全部都是同一個位置來。
3. 不懂步行
因為系統不懂得步行，所以導致搜尋結果建議多次轉車。
4. 只有成人八達通價格
對於遊客、一家人來說很不方便。
5. 交通工具不足
除了紅色小巴外，邨巴亦沒有提供。
6. 使用界面設計欠佳
網站仍使用頁框設計，網頁右上角的「路線搜尋」我最初還以以是一個標題，後來才知道它是用來提交的按鈕。另外，一進入網站時會顯示聲明及條款，實在很麻煩。在聲明及條款那頁選了中文之後，到了主頁之後又轉回英文。地圖和搜尋框分開來放實在很不方便。
到了第 2 天才知道系統只能同時容納 1000 人使用，難怪會這麼慢。資料過多而沒有歸納，導致地方名接近。

看看 Transport for London (TfL) 的 [Journey Planner](http://journeyplanner.tfl.gov.uk)，功能比起運輸署的好得多。除了可以選擇交通模式外，還可以設定你自已可接受的步行時間、單車、出發 / 抵達時間，行動不便的人還可以設定你的要求（例如不能使用樓梯、是否使用輪椅）。這方面比起 [Google Transit](http://www.google.com/transit) 還好用。不過 <abbr title="Transport for London">TfL</abbr> 的 Journey Planner 沒有多人同行的車費合計功能，相信這功能亦不難做到，最難的地方都是[最短路徑問題](http://zh.wikipedia.org/wiki/%E6%9C%80%E7%9F%AD%E8%B7%AF%E5%BE%84) ([Shortest path problem](http://en.wikipedia.org/wiki/Shortest_path))。另一個在 <abbr title="Transport for London">TfL</abbr> Journey Planner 有的功能是每個方法都會顯示時間，非常方便。

其實政府只需要將這個系統外判給 Google 地圖或者 Windows Live Local 已經大大改善了系統的功能。只不過地政總署本身就想把地圖資料賣給其他人的，並不打算公開讓公眾隨意使用（看看 PTES 的條款以及[地政總署測繪處的網頁](http://www.landsd.gov.hk/mapping/tc/digital_map/service/imp.htm)）。所以也不要想政府會推出 API 讓其他人使用它的資料來做一個更好用的問路系統。不過我覺得外判給 Google 地圖或者 Windows Live Local 做的話 Google / Microsoft 向地政總署付的版權費可能還可以抵消到政府自行開發的費用。因為 Google 本身已經有一套程式來做這些功能，只是因為 Google 沒有香港的資料而沒有提供這服務。（不清楚 Windows Live 有沒有這些功能）如果香港政府肯提供資料的話，我相信 Google / Microsoft 這些大公司亦樂意免費在它們的地圖服務加上香港的交通資料。提資料放上這些地圖網還可以在手機等不同的媒體中使用，而且還有 API。問題只是政府是否願意把這些原本是收費的資料公開。

其實年尾推出供駕車人士使用的問路系統其實亦可以使用 Google 地圖或者 Windows Live Local，問題又是版權問題。

### 延伸閱讀

* [公共交通查詢服務網尚有很大的改善空間](http://pingsum.blogspot.com/2009/04/blog-post_29.html "公共交通查詢服務網尚有很大的改善空間")
* [案例：PTES vs Google Transit](http://www.reality.hk/articles/2009/04/29/993/ "案例：PTES vs Google Transit")

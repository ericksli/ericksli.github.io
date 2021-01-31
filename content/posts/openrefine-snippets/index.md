---
title: OpenRefine GREL 筆記
tags:
- openrefine
date: 2019-02-20T23:53:05+08:00
---


[OpenRefine](http://openrefine.org/) 是一個開源的工具，用作檢視資料、加工處理後作其他用途。簡單的例子有一堆街名，部分街名用了全寫、部分用了縮寫，想將它們全部統一用全寫。它的定位是介乎 Excel 和自己寫程式之間。有時資料用 Excel 不太方便處理，但如果自己寫程式處理又因為程式只會用一次，感覺太麻煩。OpenRefine 相信可以解決到你的需要。

{{< figure src="text-facet.png" title="Text facet 功能可以批量修改相近的資料，而不用寫程式" >}}

OpenRefine 內置了一種程式語言，名為 [General Refine Expression Language (GREL)](https://github.com/OpenRefine/OpenRefine/wiki/General-Refine-Expression-Language)，和 Excel 可以用公式差不多。我們用 OpenRefine 就是透過這種語言來把資料批量轉換成自己想要的東西。

值得一提的是 OpenRefine 以前由 Google 負責維護，所以介面會有以前 Google 產品的影子。

<!--more-->

## 移除𥤧白字元

```java
value.replace(/\s+/,"")
```

## HK1980 方格網坐標轉換成 WGS84

香港政府測量是用自己的 [HK1980 方格網坐標](https://www.geodetic.gov.hk/tc/gi/refdoc.htm)，所以在香港政府[資料一線通](https://data.gov.hk/tc/)下載和坐標相關的資料都能找到 HK1980 坐標。最初他們只提供 HK1980 座標，近年才同時提供 WGS84 十進制坐標（即是 Google Maps API 用的坐標系統）。

如果你有 HK1980 坐標的話，可以在 OpenRefine 經地政總署提供的[坐標轉換應用程式界面 (API)](https://data.gov.hk/tc-data/dataset/hk-landsd-openmap-coordinates-transformation-api)把坐標轉成 WGS84 十進制坐標。

假設你的表格有 `hk80_northing` 和 `hk80_easting` 兩欄。在其中一欄點選右邊選單按鈕 > Edit column > Add column by fetching URLs...，然後輸入下面的表達式：

```java
"http://www.geodetic.gov.hk/transform/v2/?inSys=hkgrid&n="+cells['hk80_northing'].value+"&e="+cells['hk80_easting'].value
```

之後新加入的一欄會有這個 API 的 JSON response。下面是以 (828386N, 815035E) 作示範：

```json
{"wgsLat": 22.394601560,"wgsLong": 113.970677223,"hkLat": 22.396125640,"hkLong": 113.968227240,"utmGridZone": "49Q","utmGridE": 805878,"utmGridN": 2479527,"utmRefZone": "49Q-HE","utmRefE": "058","utmRefN": "795"}
```

OpenRefine 有 parse JSON 的 function，我們可以再在 JSON response 那一欄點選右邊選單按鈕 > Edit column > Add column based on this column...，然後輸入下面的表達式：

```java
value.parseJson()["wgsLat"]
```

```java
value.parseJson()["wgsLong"]
```

便可取得 WGS84 坐標。

當然，如果有大量坐標要轉換的話都是用程式轉換比較好。但數量不大的話這個方法比較方便。

## 街燈柱編號轉換

```java
"https://www.map.gov.hk/gih-ws2/lp/"+value
```

例子：

```json
[{"UTILITYNUMBER":"FA2385","UTILITYPOINTID":1107263606,"X":815027.67999,"Y":829842.619999999,"NAME":"Tuen Mun","NAME_C":"屯門"}]
```

`X` 是 HK1980 Easting；`Y` 是 HK1980 Norting。

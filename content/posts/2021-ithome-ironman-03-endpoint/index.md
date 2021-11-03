---
title: "2021 iThome 鐵人賽 Day 3：Endpoint"
tags:
  - 2021 iThome 鐵人賽
  - Android
date: 2021-09-18T00:00:00+08:00
canonicalURL: https://ithelp.ithome.com.tw/articles/10266378
---

> 本篇文章是 [2021 iThome 鐵人賽](https://ithelp.ithome.com.tw/2021ironman)參賽題目「[寫一個列車抵站時間 Android App](https://ithelp.ithome.com.tw/users/20139666/ironman/4661)」的第 3 篇，你可到 iThome [查看原文](https://ithelp.ithome.com.tw/articles/10266378)。
>
> [文章目錄]({{< ref "2021-ithome-ironman" >}})

我們用到的 API endpoint 只有一個，就是用來取得港鐵機場快綫、東涌綫、屯馬綫及將軍澳綫最多四班即將到站列車的抵達時間。車站清單我們會直接寫死在 app 裏面。API 的使用方法可以在[資料一線通網站](https://data.gov.hk/tc-data/dataset/mtr-data2-nexttrain-data)（香港政府 open data 的網站）內找到。這個 API 是不需要額外申請 API key，只是一個普通的 HTTP GET request，然後 response body 是一個 JSON object。

打開「數據字典」PDF 檔，可以找到各車站及行車綫的代碼和 response body JSON object 的說明。當中我們主要用到的 property 大致上是：

- `plat` 月台編號
- `time` 時間（留意它的格式並非按照 [ISO 8601](https://zh.wikipedia.org/zh-hk/ISO_8601)）
- `dest` 目的地車站代碼
- `seq` 次序（我們要按照這個順序顯示班次）

再看「港鐵實時列車服務資訊 API 規範文件」PDF 檔，它提供了幾個 HTTP response status code 的意義和一些 response body 例子。例如當事故發生時那個 API 會提供一段文字讓我們顯示。

## Response data class

一般而言，我們都不會直接用 Android SDK 入面的 [org.json 套件](https://developer.android.com/reference/org/json/package-summary)（即是 `JSONObject` 那些 class）來將 API response 的 JSON 轉成 Java/Kotlin object。因為要為每個 property 逐個寫 code，費時失事，所以我們都會用其他 deserialization library 去做 JSON 和 Java/Kotlin object 轉換。

在使用那些 library 之前，我們要準備 API response JSON 對應的 Kotlin data class。那個 data class 基本上就是要跟 JSON object 的結構相同，即是 response 的那個 property 是 string 就要在 data class 開對應的 string property。另外，我們只對整個 response 入面的某幾個 property 感興趣，所以在準備 data class 只針對那幾個 property 而造，其餘的東西我們就略過。

下面就是完成後的 data class，最尾的 companion object 放了幾個 constant 方便之後處理。

```kotlin
data class EtaResponse(
    /**
     * system status code.
     */
    val status: Int = STATUS_ERROR_OR_ALERT,
    /**
     * Alert message.
     */
    val message: String = "",
    /**
     * URL for Special Train Services Arrangement case.
     */
    val url: String = "",
    /**
     * Indicate if the train is delayed.
     */
    val isDelay: String = IS_DELAY_FALSE,
    val data: Map<String, Data> = emptyMap()
) {
    data class Data(
        /**
         * Indicate the destinations of the train in the specific line (up trip).
         */
        val up: List<Eta> = emptyList(),
        /**
         * Indicate the destinations of the train in the specific line (down trip).
         */
        val down: List<Eta> = emptyList()
    )

    data class Eta(
        /**
         * Platform numbers for the departure / arrival train.
         */
        val plat: String = "1",
        /**
         * Estimated arrival time (or departure time) of the train.
         */
        val time: String = EMPTY_TIMESTAMP,
        /**
         * MTR Station Code in capital letters.
         */
        val dest: String = "",
        /**
         * The sequence of the 4 upcoming trains.
         */
        val seq: String = "0"
    )

    companion object {
        const val EMPTY_TIMESTAMP = "-"

        const val STATUS_NORMAL = 1
        const val STATUS_ERROR_OR_ALERT = 0
        const val IS_DELAY_TRUE = "Y"
        const val IS_DELAY_FALSE = "N"
    }
}
```

## 小結

我們已經準備好 response 的 data class。下一篇我們會介紹不同的 JSON deserialization library 並且將 `EtaResponse` 加上 deserialization library 的 annotation。

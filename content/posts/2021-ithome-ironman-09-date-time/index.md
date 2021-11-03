---
title: "2021 iThome 鐵人賽 Day 9：Date & time"
tags:
  - 2021 iThome 鐵人賽
  - Android
date: 2021-09-24T00:00:00+08:00
canonicalURL: https://ithelp.ithome.com.tw/articles/10270899
---

> 本篇文章是 [2021 iThome 鐵人賽](https://ithelp.ithome.com.tw/2021ironman)參賽題目「[寫一個列車抵站時間 Android App](https://ithelp.ithome.com.tw/users/20139666/ironman/4661)」的第 9 篇，你可到 iThome [查看原文](https://ithelp.ithome.com.tw/articles/10270899)。
>
> [文章目錄]({{< ref "2021-ithome-ironman" >}})

上一篇在實作 `EtaResponseMapper` 的時候我們用了 Java 8 開始有的 `Instant`、`LocalDateTime` 和 `ZonedDateTime`。它們都是跟日期時間相關的 class。但其實 Kotlin 都有 [kotlinx-datetime](https://github.com/Kotlin/kotlinx-datetime) 做類似的東西。但目前 kotlinx-datetime 還是在早期開發階段，有很多常用功能都未做到，例如我們這次需要用到的 formatter 目前仍需要用 Java time，所以還是用 Java time 算。

在 Java 8 之前，Java Standard Library 是有 `Calendar`、`Date` 之類的 class，但它們本身的設計是古古怪怪。例如 `Calendar` 一月是用 `0` 表示，還有是沒有考慮到時區問題。所以後來就出現了 [Joda Time](https://www.joda.org/joda-time/) 這個 library。受到 Joda Time 的啟發，之後就出了 [JSR-310](https://jcp.org/en/jsr/detail?id=310) 的提案，最終就在 Java 8 的 Standard Library 加入了 Java Time。Java Time 除了之前用過的 `DateTimeFormatter`、`LocalDateTime`、`ZonedDateTime` 和 `Instant` 之外，還有其他常用的東西例如 `Clock`、`Period`、`Duration` 等等。它們都是用來表達不同的東西和方便我們計算關於日期時間的問題。例如我們會用 `Instant` 而不是一個 `Long` 的 variable 表示某個時刻、用 `Duration` 表示時段，這樣會令 code 更易理解，不用再擔心那個 `Long` 的單位是秒還是毫秒。另一個例子是計算 N 天後是幾年幾月幾日就不用再刻意把時分秒清零再加天數，因為可以用 `LocalDate.now().plusDays(10)` 就可以了。

由於舊版 Android 並未支援 Java 8 的功能，所以我們需要靠 [desugaring](https://developer.android.com/studio/write/java8-support) 來為我們的 app 補上部分 Java 8 或以上的 language feature，當中包括大部分的 Java Time 功能。加入 desugaring library 的方法很簡單，就是在 app module 的 *build.gradle* 加上以下的東西：

```groovy
android {
    // ...
    compileOptions {
        coreLibraryDesugaringEnabled true
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }
    kotlinOptions {
        jvmTarget = '1.8'
    }
}

dependencies {
    coreLibraryDesugaring "com.android.tools:desugar_jdk_libs:$desugaringVersion"
    // ...
}
```

Desugaring 的原理其實就是把那些 Java 8 或以上能夠 backport 到的 language feature 在 compile 時幫你補上去 APK/AAB 內，所以不論 Android 版本是新還是舊都能行到那段 code，反正都是用你 APK/AAB 提供的 class 執行。

{{< figure src="apk-content.png" title="加入了 desugaring 的 DEX 檔案包含了 Java Time 的 class" >}}

在未有 desugering 之前，一般都會用 [ThreeTen Backport](https://www.threeten.org/threetenbp/) 代替 Java Time。

## 時區問題

另一樣想借本篇討論的是時區的問題。在台灣或者香港都是 UTC+08:00，而且沒有日光節約時間 (daylight saving time, DST)。所以很多時在設計系統都沒有認真考慮時區問題（或者沒有意識到有這個問題），全部時間儲存都一概使用 UNIX timestamp。但如果系統日後需要支援多時區時就很難改得動了。

首先我們要了解 Time Zone Database 和 time zone ID。[Time Zone Database](https://www.iana.org/time-zones) 是由 IANA 管理的時區資料庫（IANA 就是管理域名的那個機構），那個 database 就是要記錄世界各地以前至未來已知的時區資訊，當中包括日光節約時間的切換規則。

那 time zone ID 就是我們平常寫 code 看到的 `Asia/Hong_Kong`、`Asia/Taipei`、`Asia/Shanghai`、 `America/New_York` 之類的東西。這些 ID 就是找一些有代表性的地名來命名，代表性的意思是指以時區、政府、以往實行過的時區之類有獨特性。雖然 `Asia/Hong_Kong`、`Asia/Taipei`、`Asia/Shanghai` 在現在都是指向 UTC+08:00，但為了能順利地轉換以前的日期時間仍要保留三個不同的 ID 表示。如果你有下載過 Time Zone Database 的話，用普通的純文字編輯器打開就會看到它會紀錄每個 time zone ID 何時選用那個 UTC 偏移量 (offset) 和有沒有執行夏令時間的資訊，還有更多的是有間該 time zone ID 的相關文獻。如果想了解更多那個地方時區的歷史的話 Time Zone Database 是不容錯過。

{{< figure src="tzdb.png" title="Time Zone Database 香港部分" >}}

Time Zone Database 香港的部分

其中一個有趣的東西是 Time Zone Database 跟中國大陸相關的有好幾個 ID，這是因為[中國大陸曾經分被劃分為五個時區](https://zh.wikipedia.org/wiki/%E4%B8%AD%E5%9C%8B%E6%99%82%E5%8D%80)：

- `Asia/Shanghai` 上海 (UTC+08:00)
- `Asia/Urumqi` 烏魯木齊 (UTC+06:00)
- `Asia/Harbin` 哈爾濱，現在指向 `Asia/Shanghai`
- `Asia/Chongqing` 重慶，現在指向 `Asia/Shanghai`
- `Asia/Kashgar` 喀什，現在指向 `Asia/Urumqi`

這些重新指向的 ID 都是為了向後兼容，即是如果表達一個以前的當地時間我們仍可以配搭這些 ID 從而計算出 UNIX timestamp 或者反向計算出當地時間。

{{< figure src="windows-time-zone.png" title="Windows 10 UTC+08:00 時區" >}}

Windows 時區列表出現好幾個地名反映了以前那些地方都採用不同時區

除了一般的時區問題之外，部分地方會實行日光節約時間 (Daylight Saving Time, DST)。意思是一年會切換兩次時區：大約在春季左右會找一天把時間調快一小時；在秋季又會再把時間調慢一小時還原。用意是因為夏天的日照時間長，把日常活動都調快一小時就能接觸更多陽光，從而節省能源（例如開少一小時電燈）。當然去到今時今日還能不能節省能源已成疑問，加上切換時間那兩天對工作和生活都造成影響。所以歐盟有考慮過廢除日光節約時間，但目前尚未實行。

其實寫了那麼多 Time Zone Database 的東西都是想指出 UNIX timestamp 不能萬能，尤其是用於表達將來的時間。因為時區可以隨時因為各地政府的政策變更（例如會否在將來取消日光節約時間），所以如果要儲存幾年後的某月某日早上 9 時要開會的話，我們應該儲存當地時間及 time zone ID，不是 UNIX timestamp 或者當地時間及 UTC 偏移量。這樣即使政府改變時區都不會影響到儲存的資料（因為可以靠更新 Time Zone Database 以取得正確結果）。其中一個改變時區的例子是北韓，[在 2015 年 8 月 15 日至 2018 年 5 月 4 日期間改用 UTC+08:30，其後改回跟南韓一樣 UTC+09:00](https://www.storm.mg/article/431531)。但如果是儲存過去的日期時間的話用 UNIX timestamp 就沒有大問題，因為過去的東西不會再改。

## Time Zone Database 更新

IANA 會跟據各地政府對時區的變更更新 Time Zone Database，所以一年可能會發布好幾個版本。這個 database 在不少地方用到，例如大家平常使用的作業系統、[JRE](https://www.oracle.com/java/technologies/tzdata-versions.html) 等等。它們都會透過系統更新或 JRE 版本更新來更新它們內裏所用的 Time Zone Database。有關 Android 系統更新 database 的方法可以參考 [AOSP](https://source.android.com/devices/tech/config/timezone-rules) 網站。

如果很在乎 Time Zone Database 是不是最新版的話，目前似乎只有 [TickTock](https://github.com/ZacSweers/ticktock)（不是抖音）和用回 [ThreeTen Backport](https://www.threeten.org/threetenbp/)。

另外，在使用 Java Time 的 method 前，緊記檢查 [desugaring 支援的 class 和對 Java Time 的特別說明](https://developer.android.com/studio/write/java8-support-table)，因為 backport 始終有部分的實作仍依賴原系統的實作。

## 參考

- [ThreeTen - Home page and Documentation](https://www.threeten.org/) JSR-310 的介紹網站，入面有對 JSR-310 主要 class 的解說（JSR-310 就是現在的 Java Time）
- [Introduction to the Java 8 Date/Time API](https://www.baeldung.com/java-8-date-time-intro)
- [Introducing kotlinx-datetime by Ilya Gorbunov](https://www.youtube.com/watch?v=YwN0kAMNvXI) kotlinx-datetime 介紹，入面有講解各款 class 的用途，還有講解日光節約時間切換和為甚麼不能用 UNIX timestamp 表示將來發生的事件的問題。即使不是寫 Kotlin 都值得看

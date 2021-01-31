---
title: Java 量度單位 (JSR 363 Units of Measurement API)
tags:
  - Java
  - Android
date: 2020-08-13T23:50:59+08:00
---


在日常生活中，我們都會用到不同的量度單位。例如重量有時會用公斤 (kg)，有時會用磅 (lb)，有時又會用斤之類。如果在 Java 上表示這些數值，用 `int`、`float`、`double` 的話有時會令人理解錯誤，就好像 UNIX timestamp 般一時會用秒一時會用毫秒。為防止人們誤解這個 variable 或 method 的時間單位，通常都要在名稱加上 `millis` 之類的後綴。知名例子有 `System.currentTimeMillis`。如果處理時間的話，現成的有 [`TimeUnit`](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/TimeUnit.html)。不過如果去到之前的重量單位的話，JDK 就沒有現成的 class。

如果是 Android 的話可以用 [`android.icu.util.Measure`](https://developer.android.com/reference/android/icu/util/Measure)，但要 API level 24 (Android 7.0) 或以上才可使用，而且不包括單位轉換功能。

如果想在 Android API level 24 以前的裝置用到的話，可以考慮用 [JSR 363](http://unitsofmeasurement.github.io/pages/about.html) (Units of Measurement API)。它提供了一套 API 來表示量度單位和數值，而且包含單位轉換和格式化。

如果要用這個 library 的話，是需要這兩個 artifact：

{{< highlight groovy >}}
implementation "tech.units:indriya:2.0.2"
implementation("systems.uom:systems-common:2.0.2") {
    exclude group: 'jakarta.annotation', module: 'jakarta.annotation-api'
}
{{< /highlight >}}

`tech.units:indriya` 是 JSR 363 的 reference implementation。它其實是用了 `javax.measure:unit-api` 所定義的 interface 的一個實作。而 `systems.uom:systems-common` 就提供了一些主流的單位的定義，例如 `SI`（國際單位制）、`Imperial`（英制）和 `USCustomary`（美制）。

要去除 `jakarta.annotation-api` 是因為 Android Gradle plugin desugaring 時會加入和 `jakarta.annotation` 同名的 class，例如 `javax.annotation.Generated`。

下面是一個由公里轉成哩的例子：

{{< highlight kotlin >}}
import org.junit.Assert.assertEquals
import org.junit.Test
import si.uom.SI
import systems.uom.common.USCustomary
import tech.units.indriya.quantity.Quantities
import javax.measure.MetricPrefix
import javax.measure.Quantity
import javax.measure.quantity.Length

class MeasureTest {
    @Test
    fun kilometerToMile() {
        val distanceKm: Quantity<Length> = Quantities.getQuantity(12, MetricPrefix.KILO(SI.METRE))
        val distanceMile: Quantity<Length> = distanceKm.to(USCustomary.MILE)
        assertEquals(7.45645431, distanceMile.value.toDouble(), 0.00000001)
    }
}
{{< /highlight >}}

用了這個 library 就是要把數值包裝成 `Quantity<Length>`，然後就可以押後處理數值的實際單位，只需知道它是長度單位就可以了，直到需要輸出時才考慮單位的問題。如果需要為單位加上[詞頭](https://zh.wikipedia.org/zh-hk/%E5%9B%BD%E9%99%85%E5%8D%95%E4%BD%8D%E5%88%B6%E8%AF%8D%E5%A4%B4) (prefix) 的話，可以套上 `MetricPrefix` 的 `KILO`、`CENTI`、`NANO` 之類的 [decorator](https://refactoring.guru/design-patterns/decorator)。

如果需要輸出顯示的話，可以用它提供的 `NumberDelimiterQuantityFormat`，不過我覺得不太好用，在 Android 的話可以用 string resource 處理。

談到文章開首提及過的斤，`systems.uom:systems-common` 當然是沒有提供，但可以自己定義新的單位。根據香港法例第 68 章[《度量衡條例》](https://www.elegislation.gov.hk/hk/cap68!en-zh-Hant-HK?INDEX_CS=N&xpid=ID_1438403555032_004)的定義，1 斤 = 0.60478982 公斤。我們可以用 `si.uom.SI.KILOGRAM` 做參照來衍生出斤這個單位：

{{< highlight kotlin >}}
fun cattyToKilogram() {
    val catty = SI.KILOGRAM.multiply(60478982).divide(100000000)
    SimpleUnitFormat.getInstance().label(catty, "斤")

    val twoCatty = Quantities.getQuantity(2.0, catty)
    val kg = twoCatty.to(SI.KILOGRAM)
    assertEquals(1.209580, kg.value.toDouble(), 0.000001)
}
{{< /highlight >}}

值得一提，斤在不同地方有不同定義。中國的一斤是等於 0.5 公斤；台灣的一斤是等於 0.6 公斤。

## 參考

- [Units of Measurement](http://unitsofmeasurement.github.io/)
- [The First IoT JSR: Units of Measurement JSR-363](https://www.oracle.com/us/assets/lad-2015-ses16319-lima-compressed-2604608.pdf)
- [Introduction to javax.measure](https://www.baeldung.com/javax-measure)

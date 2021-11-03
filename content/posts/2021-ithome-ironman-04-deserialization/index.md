---
title: "2021 iThome 鐵人賽 Day 4：Deserialization"
tags:
  - 2021 iThome 鐵人賽
  - Android
date: 2021-09-19T00:00:00+08:00
canonicalURL: https://ithelp.ithome.com.tw/articles/10267288
---

> 本篇文章是 [2021 iThome 鐵人賽](https://ithelp.ithome.com.tw/2021ironman)參賽題目「[寫一個列車抵站時間 Android App](https://ithelp.ithome.com.tw/users/20139666/ironman/4661)」的第 4 篇，你可到 iThome [查看原文](https://ithelp.ithome.com.tw/articles/10267288)。
>
> [文章目錄]({{< ref "2021-ithome-ironman" >}})

JSON serialization/deserialization 應該是不少 Android app 都會做的事，基本上近乎每個 Android project 都會用了一個或幾個這些 library，而 Android 都有好幾個選擇。除了上一篇提過的 [org.json 套件](https://developer.android.com/reference/org/json/package-summary)之外，[Gson](https://github.com/google/gson)、[Moshi](https://github.com/square/moshi) 和 [Kotlin serialization](https://github.com/Kotlin/kotlinx.serialization) 都是熱門的選擇。

一般的 deserialization library 都是透過 Java/Kotlin class 的 property 名和 data type 來跟 JSON 做轉換，例如你在 Java class 有一個叫 `createdAt` 的 `String` property，deserialization library 就會找 JSON object 有沒有一個叫 `createdAt` 的 property。如果有就把它以 `String` 的方式讀取，然後塞進去那個 Java class 的 `createdAt` property 入面。如果是 serialization 的話，就是把這個動作掉轉來做，就是看到 Java class 的 `createdAt` property 是 `String`，然後跟 Java  `String` 最相近的 JSON data type 是 `string`，那 JSON object 就會出現 `{"createdAt": "2021-09-01"}`。

## Gson

[Gson](https://github.com/google/gson) 是由 Google 開發，是上面三個 library 最早出現的一個，功能亦都很強大。例如可以設定 JSON 和 Java class property 之間的命名轉換規則（[`FieldNamingPolicy`](https://www.javadoc.io/doc/com.google.code.gson/gson/2.8.5/com/google/gson/FieldNamingPolicy.html)），亦都支援 [streaming parsing](https://sites.google.com/site/gson/streaming)，適合處理巨大的 JSON 文檔。

## Moshi

Moshi 是 Square 開發的 JSON serialization/deserialization library。特色是它們功能上沒 Gson 那麼多，專注在主要的功能上，所以比 Gson 小巧。另外一大特色是它提供了 Kotlin codegen（一個 annotation processor 生成 Moshi 的 adapter），就是 Moshi 能感知你的 Kotlin class property 是不是 nullable 而決定是不是把那個 Kotlin class property 設定成那個 property 的預設值還是用 JSON document 找到的值。

Gson 是不會知道你那個 property 是不是 nullable，因為它是 Kotlin 的 language feature。所以在 Gson 做 deserialization 時的做法是：

1. 用 Java reflection 執行 default constructor（即是沒有參數的 constructor）建立那一個 object
2. 看到那 JSON 有一個 property，就用 Java reflection 找那個 class 有沒有對應的 property，如果有就把 JSON property 的值塞進去那個 Java object 入面

問題就是出現在第二步，即使你在 default constructor 入面把那些 property 塞了不是 null 的值（如果是 Kotlin data class 就是在定義 property 時提供了預設值），但是如果 JSON property 的 value 是 null 的話 Gson 仍會把 null 塞入去那個 property。如果那個是 Kotlin 的 non-null property 那就會去到調用 property 的地方突然出現 `NullPointerException`。同樣的情況如果是用 Moshi 配合 Kotlin 支援的話，它就會在 deserialization 拋出 `JsonDataException`，不會把問題延到其他地方才發現。

## Kotlin Serialization

Kotlin Serialization 是那三個 library 最新的一個，我們將會用這個 library 來做 API JSON response 的 deserialization。這個 library 的特色是它是用 Kotlin compiler plugin 來生成 visitor 的 code（就是類似 Moshi 生成 adapter 般）、它本身的設計是支援多種格式諸如 Protobuf、Properties 等等還有是支援 Kotlin multiplatform。

Moshi 有個問題是如果 property 是 non-null 但 JSON 對應的 property 是 null 的話它就會 throw exception，但 Kotlin serialization 經過設定後可以讓它設定那個 property 的預設值而不是 throw exception。

如果你有 Protobuf 的需求的話，可以看看你是在那些地方用。如果是只有 app 自己在用（例如是 [DataStore](https://developer.android.com/topic/libraries/architecture/datastore) 的話）用 Kotlin serialization 都是不錯的選擇。但如果是和其他地方（例如跟 backend 之間通訊）那就不如考慮其他的 library。因為 Kotlin serialization 的[野心](https://github.com/Kotlin/kotlinx.serialization/issues/477#issuecomment-498019273)是如果 backend、mobile 都是用 Kotlin 寫的話，那就只需要交換那個加了 Kotlin serialization annotation 的 data class 原檔就可以了，不需要再交換 proto 檔作中轉。另外，Kotlin serialization 是用 proto2，如果要用 proto3 還是用其他 library。

## Response data class

我們選好了 Kotlin serialization，那就為先前的 data class 補回 annotation。其實要補回的 annotation 只有兩個：

- `@Serializable` 放在 class 開頭
- `@SerialName` 放在 property 開頭，用以指明 JSON 的 property 名

```kotlin
@Serializable
data class EtaResponse(
    /**
     * system status code.
     */
    @SerialName("status") val status: Int = STATUS_ERROR_OR_ALERT,
    /**
     * Alert message.
     */
    @SerialName("message") val message: String = "",
    /**
     * URL for Special Train Services Arrangement case.
     */
    @SerialName("url") val url: String = "",
    /**
     * Indicate if the train is delayed.
     */
    @SerialName("isdelay") val isDelay: String = IS_DELAY_FALSE,
    @SerialName("data") val data: Map<String, Data> = emptyMap()
) {
    @Serializable
    data class Data(
        /**
         * Indicate the destinations of the train in the specific line (up trip).
         */
        @SerialName("UP") val up: List<Eta> = emptyList(),
        /**
         * Indicate the destinations of the train in the specific line (down trip).
         */
        @SerialName("DOWN") val down: List<Eta> = emptyList()
    )

    @Serializable
    data class Eta(
        /**
         * Platform numbers for the departure / arrival train.
         */
        @SerialName("plat") val plat: String = "1",
        /**
         * Estimated arrival time (or departure time) of the train.
         */
        @SerialName("time") val time: String = EMPTY_TIMESTAMP,
        /**
         * MTR Station Code in capital letters.
         */
        @SerialName("dest") val dest: String = "",
        /**
         * The sequence of the 4 upcoming trains.
         */
        @SerialName("seq") val seq: String = "0"
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

## R8 rules

不要忘記在 [proguard-rules.pro](http://proguard-rules.pro/) 加入[對應的 R8 rule](https://github.com/Kotlin/kotlinx.serialization#android)。如果不加就會有問題。

```text
-keepattributes *Annotation*, InnerClasses
-dontnote kotlinx.serialization.AnnotationsKt # core serialization annotations

# kotlinx-serialization-json specific. Add this if you have java.lang.NoClassDefFoundError kotlinx.serialization.json.JsonObjectSerializer
-keepclassmembers class kotlinx.serialization.json.** {
    *** Companion;
}
-keepclasseswithmembers class kotlinx.serialization.json.** {
    kotlinx.serialization.KSerializer serializer(...);
}

# Change here net.swiftzer.etademo
-keep,includedescriptorclasses class net.swiftzer.etademo.**$$serializer { *; } # <-- change package name to your app's
-keepclassmembers class net.swiftzer.etademo.** { # <-- change package name to your app's
    *** Companion;
}
-keepclasseswithmembers class net.swiftzer.etademo.** { # <-- change package name to your app's
    kotlinx.serialization.KSerializer serializer(...);
}
```

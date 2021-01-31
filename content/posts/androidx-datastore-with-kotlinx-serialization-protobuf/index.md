---
title: Jetpack DataStore 搭配 kotlinx.serialization Protobuf
date: 2020-11-14T18:52:34+08:00
tags:
  - Android
  - Kotlin
---


上月 [kotlinx.serialization](https://github.com/Kotlin/kotlinx.serialization) [出了 1.0 版](https://blog.jetbrains.com/kotlin/2020/10/kotlinx-serialization-1-0-released/)。除了支援 JSON 之外，還有支援 [Protocol Buffers (Protobuf)](https://developers.google.com/protocol-buffers)，而且還是跨平台支援。而在前一個月 Android 出了 [Jetpack DataStore](https://developer.android.com/topic/libraries/architecture/datastore)，它是一個用來取代 `SharedPreferences` 的 library。它有兩種用法：

1. Preferences DataStore：像 `SharedPreferences` 般以 key 存取資料，可以隨時加新 key，而且沒有特別的 type checking 處理，全靠讀取時指明 value 的 data type。
2. Proto DataStore：用 Protobuf 來儲存資料，存取時候都是直接經 Java/Kotlin class，所以和 `SharedPreferences` 及 Preferences DataStore 相比是 type safe。

既然要用 DataStore，那就當然要用 Proto DataStore。如果有看過 [Codelab](https://developer.android.com/codelabs/android-proto-datastore) 的話，它都會叫大家用 `com.google.protobuf:protoc` 和 `com.google.protobuf:protobuf-javalite` 把 proto 檔案生成對應的 Java class。就像處理 JSON 要找 Gson 之類的 library 做 serialization/deserialization 一樣，不過 Protobuf 的用法是先定義好一個 proto 檔（即是 schema），然後把這個檔案交給 protoc compiler 生成不同程式語言的 entity class，然後會有另一個 library 做 entity class 和 protobuf serialization/deserialization。但其實 DataStore 的 API 設計並沒有硬性規定要用 Google 的 Protobuf library，甚至無規定用 Protobuf 格式。

<!-- more -->

DataStore 的 API 就是提供一個 Kotlin Coroutine/Flow 的方式讀寫 object（Proto DataStore 的話）。這樣就可以逼大家把 I/O 動作放去其他 thread 執行（預設是用 `Dispatchers.IO`），又可以用 Flow observe 改動。相比起 `SharedPreferences`，`SharedPreferences` 的 API 大部分都是 synchronous，調用起來又很快，放在 UI thread 好像問題不大。但有時候改動 data 時用 `apply()` 後馬上讀取可能會讀不到最新值，用 `commit()` 又因為用 `SharedPreferences` 時未有考慮到寫入是 asynchronous 所以 linter 又出警告。所以 DataStore API 設計上就索性改用 Kotlin Coroutine/Flow。而用 Protobuf 儲資料是因為它既 type safe 又比 `SharedPreferences` 所用的 XML 細小。DataStore 的 Protobuf 存放位置跟 `SharedPreferences` 有點不同，它是放在 *files/datastore* 目錄內，而不是 *shared_prefs* 內。

{{< figure src="datastore-file-location.png" title="DataStore 存放位置" >}}

## 搭配 kotlinx.serialization

⚠️ 註：這篇文章是以 Jetpack DataStore 1.0.0-alpha03 來寫的。

如果想改用 kotlinx.serialization 來處理 Protobuf 的話，首先要準備 data class。Data class 要有 `@Serializable`，property 最好要有 `@ProtoNumber`，以便日後 data class 增減 property 後 Protobuf 能夠相容。

{{< highlight kotlin >}}
import kotlinx.serialization.Serializable
import kotlinx.serialization.protobuf.ProtoNumber

@Serializable
data class AssetConfig(
    @ProtoNumber(1) val version: String = "",
    @ProtoNumber(2) val path: String = "",
)
{{< /highlight >}}

不要忘記加入 `org.jetbrains.kotlin.plugin.serialization` Gradle plugin 和 `org.jetbrains.kotlinx:kotlinx-serialization-protobuf` 到 module 的 dependency。沒有這兩個東西是不能夠做到 serialization/deserialization。

之後要寫一個 DataStore `Serializer`。它是用來設定如果 file system 沒有那個 protobuf 時要回傳甚麼，還有是做 serialization 和 deserialization。由於我們的 data class 每個 property 都有預設值，所以 `defaultValue` 直接用 `AssetConfig()` 就算了。

而 `readFrom` 是用來 deserialize Protobuf。我們要將 `InputStream` 的內容交去 kotlinx.serialization。`InputStream` 不需要 close，因為 `androidx.datastore.core.SingleProcessDataStore` 的 `readData` 在 call `serializer` 時已經有用 [`use`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.io/use.html) 包住 `FileInputStream`，所以它會幫我們 close。`use` 就是 Java [try-with-resources](https://docs.oracle.com/javase/tutorial/essential/exceptions/tryResourceClose.html) 的替代品。下面是 `readData` 的 code，除了那個 `use` 之外，還看到 `defaultValue` 的用法：

{{< highlight kotlin >}}
private suspend fun readData(): T {
    try {
        FileInputStream(file).use { stream ->
            return serializer.readFrom(stream)
        }
    } catch (ex: FileNotFoundException) {
        if (file.exists()) {
            throw ex
        }
        return serializer.defaultValue
    }
}
{{< /highlight >}}

之後要再 override `writeTo`，就是 `readFrom` 的相反。同樣是不要 close `OutputStream`。這次 KDoc 有寫明不可以 close。

> Marshal object to a stream. writeTo should not close output, doing so will result in an exception.

下面就是完整的 `Serializer`：

{{< highlight kotlin >}}
import androidx.datastore.core.CorruptionException
import androidx.datastore.core.Serializer
import kotlinx.serialization.SerializationException
import kotlinx.serialization.decodeFromByteArray
import kotlinx.serialization.encodeToByteArray
import kotlinx.serialization.protobuf.ProtoBuf
import java.io.InputStream
import java.io.OutputStream

object AssetConfigSerializer : Serializer<AssetConfig> {
    override val defaultValue: AssetConfig
        get() = AssetConfig()

    override fun readFrom(input: InputStream): AssetConfig {
        return try {
            ProtoBuf.decodeFromByteArray(input.readBytes())
        } catch (e: SerializationException) {
            throw CorruptionException(e.message.orEmpty(), e)
        }
    }

    override fun writeTo(t: AssetConfig, output: OutputStream) {
        output.write(ProtoBuf.encodeToByteArray(t))
    }
}
{{< /highlight >}}

所以就是找到方法把 `InputStream`/`OutputStream` 和那個 object 互相轉換到就可以了，沒有限定要用那個 Protobuf library，甚至用其他格式都可以。

## 使用 DataStore

DataStore 使用上不太難用。首先是要取得 `DataStore`，這有點像 `SharedPreferences`，要提供 Protobuf 檔案名稱，另外要提供剛才做的 `Serializer`。

如果要讀取就用 `data`，它是 `Flow`，有改動時就會通知。變更的話就用 `updateData`，lambda 會提供目前的 object。由於我們的 data class 全部 property 都是用 `val`，所以用了 `copy`。lambda 的 return value 就是將會寫入 Protobuf 的內容。

{{< highlight kotlin >}}
import androidx.datastore.createDataStore

val dataStore: DataStore<AssetConfig> = context.createDataStore(
    fileName = "asset_config.pb",
    serializer = AssetConfigSerializer
)

dataStore.data.collect { config ->
    // new value of config
}

dataStore.updateData { config ->
    // config is the current value
    config.copy(version = "2.0")
}
{{< /highlight >}}

## 用不用 DataStore 好

用 Jetpack DataStore 前期準備好像比 `SharedPreferences` 複雜（因為要準備 .proto 檔 / data class 又要寫一個 `Serializer`），但就多了 type safe 特性。如果你會把 JSON string 塞入 `SharedPreferences` 的話，用 DataStore 會比 `SharedPreferences` 好。因為 `SharedPreferences` 背後是儲存在 internal storage 的 XML 檔。如果 value 是 JSON 的話，那就是要 serialize/deserialize 兩次（用 `SharedPreferences` 的 `getString` deserialize 一次，之後再用 Gson 之類又再 deserialize 一次）。改用 DataStore 就直接由 Protobuf deserialize 一次就行。所以如果是 nested object 的話用 DataStore 其實是很好的。至於一般 key value 的話要看你想不想轉用。

如果你有用 [AndroidX Preference](https://developer.android.com/guide/topics/ui/settings) 的話，要留意需要重寫設定頁。因為 `PreferenceFragmentCompat` 本來就是配 `SharedPreferences` 來用，如果想自訂儲存方式的話，本來是可以用 [`PreferenceDataStore`](https://developer.android.com/guide/topics/ui/settings/use-saved-values#custom-data-store)，但那些 method 都是 synchronous，所以現實上很難改到。不過大部分人都是自製設定頁，所以這部分問題不大。最大問題應該是要把全部本來用 `SharedPreferences` 的地方都要改做 asynchronous。如果當初寫的時候沒有把讀寫 `SharedPreferences` 的地方都有考慮到 asynchronous 的話那個改動就會超大，除非把 DataStore 的 Flow 和 Coroutine 都強行變成 `runBlocking`。

至於 kotlinx.serialization 方面，你可能會留意到我們從來沒有寫過 .proto 檔。這是因為 kotlinx.serialization 是靠 Kotlin class 和 annotation 定義 schema，有別於其他 library 要由 .proto 生成 Java/Kotlin class 才能用。如果那個 Protobuf 只會在 Android app 用到，那問題不大。但如果要處理的 Protobuf 是跨平台（例如 backend 拍板 schema，其他地方例如 Android、iOS、Web 要跟那個 schema 的話），用 kotlinx.serialization 會比較尷尬。因為人家給你的 .proto 檔不能直接用，要自己對住它用 Kotlin 寫一次 data class。但寫的時候不能保證你寫的跟人家的 .proto 定義一致。如果不能 serialize/deserialize 又要額外花時間 debug。因為 kotlinx.serialization 定位是以 Kotlin 為中心，只需要 backend frontend 共用那個 Kotlin file 就能解決問題，完全不需要交換 .proto 檔。不過如果你有跨平台的考量需要交換 .proto 檔的話，那可能用 Google 的 Protobuf library 或者 [Wire](https://square.github.io/wire/) 之類比較合適。但如果情況是 backend 用 JSON，想在 Android 用 DataStore Proto 儲存的話（可能是一些零碎需要 offline cache 的內容），直接用 kotlinx.serialization 或許比較方便，因為可以共用同一個 Kotlin data class，kotlinx.serialization 的 annotation 又是跨格式共用的。

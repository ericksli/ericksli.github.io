---
title: "2021 iThome 鐵人賽 Day 8：Data layer implementation (2)"
tags:
  - 2021 iThome 鐵人賽
  - Android
date: 2021-09-23T00:00:00+08:00
canonicalURL: https://ithelp.ithome.com.tw/articles/10270358
---

> 本篇文章是 [2021 iThome 鐵人賽](https://ithelp.ithome.com.tw/2021ironman)參賽題目「[寫一個列車抵站時間 Android App](https://ithelp.ithome.com.tw/users/20139666/ironman/4661)」的第 8 篇，你可到 iThome [查看原文](https://ithelp.ithome.com.tw/articles/10270358)。
>
> [文章目錄]({{< ref "2021-ithome-ironman" >}})

上一篇的 repository 還欠一個 mapper 把 `EtaResponse` 轉成 `EtaResult`。我們首先準備一個通用的 interface：

```kotlin
interface Mapper<T, R> {
    suspend fun map(o: T): R
}
```

有些 Android Clean Architecture 會每個 layer 都準備一個對應的 mapper interface，這次示範我們就簡化這一部分，全部 layer 都共用同一個 mapper interface，不論是由高層次 layer 去低層次 layer 還是由低層次 layer 去高層次 layer 都一樣。因為這個 mapper 都是為了在寫 unit test 時可以 mock 那個 interface 而不是 mock 那個 concrete implementation，所以它是不是共用 interface 問題不大。

以下是整個 mapper class 的 code：

```kotlin
private val TIMESTAMP_FORMATTER = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")

class EtaResponseMapper @Inject constructor() : Mapper<HttpResponse, EtaResult> {
    override suspend fun map(o: HttpResponse): EtaResult = when (o.status) {
        HttpStatusCode.OK -> mapResponse(o.receive())
        HttpStatusCode.TooManyRequests -> EtaResult.TooManyRequests
        HttpStatusCode.InternalServerError -> EtaResult.InternalServerError
        else -> EtaResult.Error(IllegalStateException("Unsupported HTTP status code ${o.status}"))
    }

    private fun mapResponse(response: EtaResponse): EtaResult = with(response) {
        when {
            status == EtaResponse.STATUS_ERROR_OR_ALERT -> EtaResult.Incident(
                message = message,
                url = url,
            )
            isDelay == EtaResponse.IS_DELAY_TRUE -> EtaResult.Delay
            else -> EtaResult.Success(
                schedule = sequence {
                    yieldAll(data.values.asSequence()
                        .flatMap { it.up }
                        .map { mapEta(EtaResult.Success.Eta.Direction.UP, it) })
                    yieldAll(data.values.asSequence()
                        .flatMap { it.down }
                        .map { mapEta(EtaResult.Success.Eta.Direction.DOWN, it) })
                }.toList()
            )
        }
    }

    private fun mapEta(direction: EtaResult.Success.Eta.Direction, eta: EtaResponse.Eta) =
        with(eta) {
            EtaResult.Success.Eta(
                direction = direction,
                platform = plat,
                time = try {
                    ZonedDateTime.of(
                        LocalDateTime.parse(time, TIMESTAMP_FORMATTER),
                        DEFAULT_TIMEZONE
                    ).toInstant()
                } catch (e: DateTimeParseException) {
                    Instant.EPOCH
                },
                destination = try {
                    Station.valueOf(dest)
                } catch (e: IllegalArgumentException) {
                    Station.UNKNOWN
                },
                sequence = seq.toIntOrNull() ?: 0,
            )
        }
}
```

雖然看起來很長，但做的東西其實很簡單：首先是看 HTTP response status code，如果是 `429` 和 `500` 可以馬上回傳 `EtaResult.TooManyRequests` 和 `EtaResult.InternalServerError`，不用再花時間看 response body。至於 `200` 就要看 response body 才能知道要回傳甚麼（即是 `mapResponse` 的部分）。

去到 `mapResponse`，我們先把易處理的東西處理，例如 `status` 和 `isDelay` 兩個 property。`EtaResult.Incident` 和 `EtaResult.Delay` 就是靠這兩個 property 來判斷。最後剩下的就是最平常的情況，那就是 response 有提供列車班次。

`mapEta` 入面是把 `EtaResponse.Eta` 轉換成 `EtaResult.Success.Eta`。Response JSON object 是用 `UP` 和 `DOWN` array 區別上下行班次，但我們把兩個 array 合併成一個 list，方便日後 UI 可以自訂排序。時間方面，我們把原來是疑似 [ISO 8601](https://zh.wikipedia.org/wiki/ISO_8601) 格式的 string 的日期時間轉成 `Instant`。由於它不是正式的 ISO 8601 格式，我們要針對這個格式寫一個 `DateTimeFormatter` (`TIMESTAMP_FORMATTER`) 轉換換成 `LocalDateTime` 然後再轉成 `ZonedDateTime` 最後轉換成 `Instant`。`DEFAULT_TIMEZONE` 是定義在另一個檔案：

```kotlin
val DEFAULT_TIMEZONE: ZoneId = ZoneId.of("Asia/Hong_Kong")
```

不要忘記還有一樣東西要做的是要在 `DataModule` 加回 `@Binds` 的 function，否則在用的時候 Dagger 會報錯：

```kotlin
@Binds
fun bindEtaResponseMapper(mapper: EtaResponseMapper): Mapper<HttpResponse, EtaResult>
```

到了這裏，data layer 的實作就完成了。完整的 code 可以直接去 [GitHub repo](https://github.com/ericksli/eta-demo) 查閱。

---
title: "2021 iThome 鐵人賽 Day 11：Data layer testing (2)"
tags:
  - 2021 iThome 鐵人賽
  - Android
date: 2021-09-26T00:00:00+08:00
canonicalURL: https://ithelp.ithome.com.tw/articles/10272245
---

> 本篇文章是 [2021 iThome 鐵人賽](https://ithelp.ithome.com.tw/2021ironman)參賽題目「[寫一個列車抵站時間 Android App](https://ithelp.ithome.com.tw/users/20139666/ironman/4661)」的第 11 篇，你可到 iThome [查看原文](https://ithelp.ithome.com.tw/articles/10272245)。
>
> [文章目錄]({{< ref "2021-ithome-ironman" >}})

今天會繼續寫 `EtaResponseMapperTest`。我們示範的 test case 是正常輸出班次的情景。首先是準備 response：

```kotlin
val response = mockHttpResponse(
    statusCode = HttpStatusCode.OK,
    etaResponse = EtaResponse(
        status = EtaResponse.STATUS_NORMAL,
        message = "successful",
        isDelay = EtaResponse.IS_DELAY_FALSE,
        data = mapOf(
            "TKL-TKO" to EtaResponse.Data(
                up = listOf(
                    EtaResponse.Eta(
                        plat = "1",
                        time = "2020-01-11 14:28:00",
                        dest = "POA",
                        seq = "1",
                    ),
                    EtaResponse.Eta(
                        plat = "1",
                        time = "2020-01-11 14:36:00",
                        dest = "LHP",
                        seq = "2",
                    ),
                ),
                down = listOf(
                    EtaResponse.Eta(
                        plat = "2",
                        time = "2020-01-11",
                        dest = "XXX",
                        seq = "",
                    ),
                ),
            ),
        ),
    ),
)
```

在正常的情況下 server 會提供上行及下行的班次。在我們的實作中，上下行其實是用相同方法處理，那我們就在上行的 array 放一些正常的內容然後在下行放有問題的內容，看看我們的 mapper 能否正常處理。

接着就是 assertion 的部分。我們先檢查輸出的結果是不是 `EtaResult.Success`，然後用 `and` 開了一個 lambda。

```kotlin
expectThat(mapper.map(response)).isA<EtaResult.Success>().and {
    // 入面的 this 就變了 AssertionBuilder<EtaResult.Success>
    // 這樣就可以繼續深入檢查 EtaResult.Success 的 property
}
```

之後我們就會按照 `EtaResult.Success` 的每一個 property 檢查。它只有一個 property `schedule` ，我們再可以開另一層的 lambda 檢查 `List<EtaResult.Success.Eta>` 入面的內容。

針對 collection 的部分，Strikt 提供了幾個 method 方便我們做 assertion：

- `isNotEmpty()` 檢查 collection 是不是空
- `hasSize(3)` 檢查 collection 是不是有 3 個元素
- `get(2)` 拿 index 第 2 的元素
- `first()` 就是上面 `get` 的特別版，只取第一個元素
- `containsExactly(1, 2, 3)` collection 入面必須要有這些元素，而且要按同樣次序出現
- `containsExactlyInAnyOrder(1, 2, 3)` 和上面差不多，只是沒規定先後次序

更多的用法可以參考 [Strikt 網站](https://strikt.io/wiki/collection-elements/)。之後我們可以再駁 `and` 來進入那一個元素作檢查。一般我們都是會先用 `get` 取得那個 property 的 `AssertionBuilder` 然後再接駁那些 `isEqualTo`、`isNull` 之類的 assertion。留意這個 `get` 不是 `get(0)` 那個，parameter 用數字是用來取得 collection 的某一個元素；但如果 parameter 是用 property/method reference 或者 lambda 就是用來取得那個 property。

以下是 property/method reference 和 lambda 寫法例子：

```kotlin
val subject: EtaResult.Success.Eta = EtaResult.Success.Eta(...)
expectThat(subject) {
    // property/method reference
    get(EtaResult.Success.Eta::platform).isEqualTo("1")
    // lambda
    get { platform }.isEqualTo("1")
}
```

兩者效果都是一樣，但 Strikt 網站指出用 lambda 寫法效能上比用 property/method reference 差。所以如果可以的話都盡量用 property/method reference。

以下是整個 assertion 的部分：

```kotlin
expectThat(mapper.map(response)).isA<EtaResult.Success>().and {
    get(EtaResult.Success::schedule).isNotEmpty().and {
        get(0).and {
            get(EtaResult.Success.Eta::direction).isEqualTo(EtaResult.Success.Eta.Direction.UP)
            get(EtaResult.Success.Eta::platform).isEqualTo("1")
            get(EtaResult.Success.Eta::time).isEqualTo(
                ZonedDateTime.of(
                    2020, 1, 11, 14, 28, 0, 0,
                    DEFAULT_TIMEZONE
                ).toInstant()
            )
            get(EtaResult.Success.Eta::destination).isEqualTo(Station.POA)
            get(EtaResult.Success.Eta::sequence).isEqualTo(1)
        }
        get(1).and {
            get(EtaResult.Success.Eta::direction).isEqualTo(EtaResult.Success.Eta.Direction.UP)
            get(EtaResult.Success.Eta::platform).isEqualTo("1")
            get(EtaResult.Success.Eta::time).isEqualTo(
                ZonedDateTime.of(
                    2020, 1, 11, 14, 36, 0, 0,
                    DEFAULT_TIMEZONE
                ).toInstant()
            )
            get(EtaResult.Success.Eta::destination).isEqualTo(Station.LHP)
            get(EtaResult.Success.Eta::sequence).isEqualTo(2)
        }
        get(2).and {
            get(EtaResult.Success.Eta::direction).isEqualTo(EtaResult.Success.Eta.Direction.DOWN)
            get(EtaResult.Success.Eta::platform).isEqualTo("2")
            get(EtaResult.Success.Eta::time).isEqualTo(Instant.EPOCH)
            get(EtaResult.Success.Eta::destination).isEqualTo(Station.UNKNOWN)
            get(EtaResult.Success.Eta::sequence).isEqualTo(0)
        }
    }
}
```

留意因為 `Instant` 有 override `equals` 和 `hashCode`，還有是 immutable，所以我們可以直接建構一個新的 `Instant` 跟輸出做比較，如果那個 class 沒有正確地 override `equals` 和 `hashCode` 的話，還是要逐個 property call getter 檢查比較安全。

你或許會問為甚麼我們不直接建構一個塞好我們期望的內容的 `EtaResult.Success` object 然後直接 `isEqualTo` 做比對。這是因為我想善用 Strikt 這類 assertion library 的優勢。Strikt 這類 assertion library 如果 assertion 出問題的話它會輸出詳細的錯誤訊息讓你看。

```kotlin
@Test
fun incident() = runBlockingTest {
    val response = mockHttpResponse(
        statusCode = HttpStatusCode.OK,
        etaResponse = EtaResponse(
            status = EtaResponse.STATUS_ERROR_OR_ALERT,
            message = "Special train service arrangements are now in place on this line.",
            url = "https://www.mtr.com.hk",
        ),
    )
    expectThat(mapper.map(response)).isA<EtaResult.Incident>().and {
        get(EtaResult.Incident::message).isEqualTo("Special train service arrangements are now in place on this line.")
        get(EtaResult.Incident::url).isEqualTo("https://www.mtr.com.hk/alert/alert_title_wap.html")
    }
}
```

```text
▼ Expect that Incident(message=Special train service arrangements are now in place on this line., url=https://www.mtr.com.hk):
  ✓ is an instance of net.swiftzer.etademo.domain.EtaResult$Incident
  ▼ value of property message:
    ✓ is equal to "Special train service arrangements are now in place on this line."
  ▼ value of property url:
    ✗ is equal to "https://www.mtr.com.hk/alert/alert_title_wap.html"
            found "https://www.mtr.com.hk"
```

如果直接用 `isEqualTo` 的話，那 assertion 會寫成：

```kotlin
expectThat(mapper.map(response)).isA<EtaResult.Incident>().isEqualTo(
    EtaResult.Incident(
        message = "Special train service arrangements are now in place on this line.",
        url = "https://www.mtr.com.hk/alert/alert_title_wap.html",
    )
)
```

之後輸出的錯誤訊息都是把那個 class 的 `toString` 放給你自己看，但 property 一多你就很難檢查：

```text
▼ Expect that Incident(message=Special train service arrangements are now in place on this line., url=https://www.mtr.com.hk):
  ✓ is an instance of net.swiftzer.etademo.domain.EtaResult$Incident
  ✗ is equal to Incident(message=Special train service arrangements are now in place on this line., url=https://www.mtr.com.hk/alert/alert_title_wap.html)
          found Incident(message=Special train service arrangements are now in place on this line., url=https://www.mtr.com.hk)
```

`EtaResponseMapperTest` 的其他 test case 其實寫法都大同小異，所以我就不逐一介紹。有興趣的話可以直接[去 GitHub 看 code](https://github.com/ericksli/eta-demo/blob/main/app/src/test/java/net/swiftzer/etademo/data/EtaResponseMapperTest.kt)。

下一篇會寫 `EtaRepositoryImplTest`。

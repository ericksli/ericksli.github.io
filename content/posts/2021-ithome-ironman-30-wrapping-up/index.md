---
title: "2021 iThome 鐵人賽 Day 30：Wrapping up"
tags:
  - 2021 iThome 鐵人賽
  - Android
date: 2021-10-15T00:00:00+08:00
canonicalURL: https://ithelp.ithome.com.tw/articles/10281769
---

> 本篇文章是 [2021 iThome 鐵人賽](https://ithelp.ithome.com.tw/2021ironman)參賽題目「[寫一個列車抵站時間 Android App](https://ithelp.ithome.com.tw/users/20139666/ironman/4661)」的第 30 篇，你可到 iThome [查看原文](https://ithelp.ithome.com.tw/articles/10281769)。
>
> [文章目錄]({{< ref "2021-ithome-ironman" >}})

終於來到最後一篇了！不經不覺已經寫了三十篇文章。我們由 Ktor client 接駁 API 一直講到 UI，然後再做 ViewModel 的 unit testing。中間加插了時間處理、Flipper 和 proxy server 的內容。其實這些內容有部分是以前工作上跟 Android 同事定期會議分享的內容。那時已經有想法把內容放到自己的 blog 上，但最後只是放了小許。現在有這個機會就加插這些內容進去。除了用來填滿三十篇之外，就是把一些不會直接在 Android 開發教學找到但又實用的東西放進去。我在八月尾才決定題目，然後開始寫開首的文章，並且準備示範 project 的 code。初初寫的時候以為三十篇是很多，所以開頭寫的內容不夠充實。但到了多 code snippet 的部分就發覺光是 code 就很長，要分拆做好幾篇。所以三十篇入面光是不同的 unit testing 都佔了十篇。由於我是一邊寫文一邊準備示範 project，所以內容分配是頭輕尾重，尤其是後段做 UI 的部分一篇的長度比開首的文章長幾倍。還有是內容可能有時會跟前一兩篇重覆（好像 Dagger 某些內容有重覆提及）。本來還打算加插 Compose 的內容但發覺剩餘篇數太少而且寫完都不夠完整，所以改做 ViewModel 測試作罷。

現在回顧那個示範 project 有個地方做錯的是改變排序不應該觸發 API call。要做到這個效果的話可以把排序的動作搬到 ViewModel 做，或者是 use case 是有自己的狀態（即是會保留之前取得的 response）。前者的話就令到 domain layer 是多餘；後者的話我又覺得班次這件事不會有其他東西觸發它改變，把它的狀態保留又有點怪。如果我們做的功能是類似 Facebook news feed，這樣 domain layer 保留一份資料就很合理。因為可以在 domain layer 整合 push notification，當有 push notification 的時候就用 push payload 更新本地現有的 news feed，然後把 news feed 外露成 `Flow`。這樣 ViewModel 就不用知道有甚麼情況那個 news feed 會被更新，只需要訂閱 news feed 就可以了。

如果把排序的 logic 放到 ViewModel，這樣 use case 就只是左手交右手（把 data layer 的東西轉交予 presentation layer）。但我好難找到一個有充實 business logic 而且做完之後又可以抄到我自己的 side project 用的情景（因為選即時班次是因為我的 side project [MetroRide app](https://play.google.com/store/apps/details?id=net.swiftzer.metroride) 要做這個功能）。其實不是沒有，只是 business logic 太複雜，講解它都用不少篇幅，變相整個系列不是在示範 Android 的東西。如果個別 use case 沒有甚麼特別的 logic 的話，[為了令整個 app 的做法統一還是建議寫 use case](https://proandroiddev.com/why-you-need-use-cases-interactors-142e8a6fe576)。

另外一個做得不好的地方是倒數分鐘只是靠每十秒 call API 後更新而不是按實際時間更新。解決方法可以是再另加一個每秒觸發的 `Flow` 來重新計算倒數分鐘值，但因為我不想弄得太複雜而沒有做到（因為寫完又要示範寫 test case，本身每十秒自動更新都要寫一大堆 test case）。

如果要談完成鐵人賽有甚麼東西學到的話，我想主要是試用了一次 Ktor client、Kotlin Coroutine scope 和 Flow 測試快進時間。以前做的項目都是用 OkHttp 加 Retrofit，如果是要測試快進時間的話都是用 RxJava。Kotlin Coroutine 和 Flow 感覺上比 RxJava 易學，最起碼沒有 `subscribeOn` 和 `observeOn` 傻傻分不清楚的問題，而且 Coroutine 的 suspending function 可以令整個寫法看起來像平常寫 code 的形式，不用費神想如何把它們串聯成一條 RxJava 的 `Single`/`Completable` 之類。加上現在 Android Jetpack 都有足夠的配套，像是 lifecycle 的 scope、data binding 支援等等。用過之後就回不了 LiveData 和 RxJava 去。

整個系列有三分之一是做測試，這是我特別花篇幅做的。因為身邊不少同事都不太有寫自動化測試的經驗，亦不明白為甚麼要用 dependency injection library。我覺得沒有寫過自動化測試的人是很難明白為甚麼要弄一大堆 interface，又要用 constructor injection。因為這些東西本來就可以用 singleton 就能做到。但如果你了解到寫測試時需要 [everything's under control](https://youtu.be/FiW503Q-pP0?t=68) 的話就自然明白到為甚麼要搞這麼多東西，為的就是能在測試時控制到自己想要的情景。還有是揀選 architecture 時可以用能否容易寫 unit test 做參考準則，如果發覺很難寫或者寫得很古怪的話那個 architecture 應該都有點問題。現在有了 Dagger Hilt，Android Jetpack 又有配套，就算不完全明白全部 Dagger 的東西也可以輕鬆做到 dependency injection。

最後，感謝各位花時間閱讀我的文章。如果想看其他的東西可以到我的 [blog](https://eric.swiftzer.net/)，如果有用 Medium 的話可以 follow 我的 [Medium](https://ericksli.medium.com/)。

---

順帶一提，有一個有趣的東西是我在做示範 app 的時候無意中發現港鐵的抵站時間 API 居然會時光倒流。最初看到 UI 還以為我計算倒數出錯所以顯示不到倒數，但檢查 API response 才發現他們 backend 的班次不會把日期進位，一直停留在同一日，直至全部班次都是翌日才會輸出正確的日期。我想他們應該是把日期和時間分開處理，之後臨到輸出 response 時才把它們合併一起。

{{< figure src="time-travel.png" title="港鐵的抵站時間 API 班次時間計算錯誤" >}}

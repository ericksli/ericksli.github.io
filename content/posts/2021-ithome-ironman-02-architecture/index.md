---
title: "2021 iThome 鐵人賽 Day 2：Architecture"
tags:
  - 2021 iThome 鐵人賽
  - Android
date: 2021-09-17T00:00:00+08:00
canonicalURL: https://ithelp.ithome.com.tw/articles/10265616
---

> 本篇文章是 [2021 iThome 鐵人賽](https://ithelp.ithome.com.tw/2021ironman)參賽題目「[寫一個列車抵站時間 Android App](https://ithelp.ithome.com.tw/users/20139666/ironman/4661)」的第 2 篇，你可到 iThome [查看原文](https://ithelp.ithome.com.tw/articles/10265616)。
>
> [文章目錄]({{< ref "2021-ithome-ironman" >}})

## Architecture Components

以前 Android Developers 網站沒有特別提及過寫 Android app 應該用甚麼 architecture，直至到近年 Google 因應社群的要求才[建議使用 MVVM](https://developer.android.com/topic/architecture)並推出了對應 MVVM 的 Architecture Components library。下圖是 Android Developers 網站建議的 architecture：

{{< figure src="android-architecture.png" title="Android Developers 網站建議的 architecture" >}}

它主要特色是：

- Activity/Fragment 負責 UI 部分
- Repository 是外露資料處理的 method 讓 ViewModel call，使 ViewModel 無須得知資料的實質來源地
- ViewModel 是把 Activity/Fragment 和 Repository 粘合在一起。它外露一些 method 觸發資料的處理（例如一個按鈕的 `onClick` listener 會 call ViewModel 的一個 method，ViewModel 會 call Repository 發送資料去 server），而 Repository 處理資料後的結果會透過 ViewModel 外露的 `LiveData` 然後在 Activity/Fragment observe 並按所收到的東西再更新 UI

如果 app 是一般 CRUD (Create, Read, Update, Delete) 類型的話這個架構應該足以應付。但如果 app 有很多 business logic 要處理的話（以計算地鐵車費為例，buiness logic 就是把 SQLite database 取得的由 A 站到 B 站的原車費按不同的情景加工處理，然後輸出折實價給 UI 顯示），這些 business logic code 可能會塞去 ViewModel 或 Repository，結果令那些 class 塞了太多 code。所以就出現了我們常常聽到的 Clean Architecture。在 Android app 一般的做法是把上圖的 Repository 和 ViewModel 加插 Use Case 或 Interactor 來塞 business logic，那些 use case 或 interactor 就是 Domain layer；Activity/Fragment 至 ViewModel 是 presentation layer；repository 至 SQLite/Retrofit 是 Data layer。

## Modularization

Clean Architecture 沒有特定標準，所以有些比較講究的人會開設不同的 Gradle module 放置不同部分的 code 而不是把所有的 code 都放到同一個 module 內。至於如何界開不同的 module（即是「modularization」）亦都是一個不會有結論的話題。

最簡單的方法是：

- Presentation（放 UI 相關的東西，即是 Activity/Fragment/ViewModel）
- Domain（放 Use case 的 interface 和 concrete implementation、Repository interface）
- Data（放 Repository 的 concrete implementation、供 Local 和 Remote implement 的 data source interface）
- Local（處理本機資料來源，例如存取 Shared Preferences、Data Store、SQLite 之類並 implement data source interface 供 Data 使用）
- Remote（處理遠端資料來源，例如 HTTP request 和 Web socket 之類並 implement data source interface 供 Data 使用）

好處是可以盡量把 module 設定成純 Kotlin/Java module，例如是 Domain、Data 和 Remote。那就減少亂寫 code 的情況（比如說是在 HTTP response error 時直接彈出 Toast 而不是在在 Presentation layer 決定用甚麼形式顯示 error）。還有是因為 Gradle module 之間的 dependency 能控制 module 內的 class 能否接觸到另一 module 的 class，所以比單純以 Java/Kotlin package 分隔不同 layer 更有保證。如果你打算做 Kotlin Multiplatform Mobile (KMM) 的話，刻意分割成純 Kotlin module 是必要的，因為 KMM 共用的 code 必須要是純 Kotlin 寫的，連調用 Java Standard Libaray 都不可以。

缺點是如果要修改某一個 feature 的話，你需要到每一個 module 修改 code，如果初接觸這種 modularization 方式的話很容易會被搞到頭昏腦脹，同時亦做不了 Play Feature Delivery。

另一種 modularization 方式是單純的按 feature 劃分，一個 feature 一個 module。一個 module 包含了該 feature 全部 layer 的 class，通常都是會用 Java/Kotlin package 劃分，這樣就可以做到 Play Feature Delivery。但缺點是要決定那些東西是跨 feature 共用，例如 SQLite database 是不是共用同一個，如果不共用如何保證 ACID？或者是兩個 feature 都要 call 同一個 API endpoint，那應該是共用同一套 code 還是把那些 code 抄到各自的 feature module 內？當出現了 common module 放這些跨 feature 的 code 後，common module 最終會不會變成另一個垃圾崗？另外，跨 module 的 navigation 亦是一大難題，即使出了 [Navigation Component](https://developer.android.com/guide/navigation) 仍未有完整方案，你不能在 feature module 的 project 使用 Navigation Component 的全部功能。這可能是你不能用到 safe args、IDE 會出現紅字、要自己寫額外 code 去封裝 Navigation Component 之類。

如果想同時做到按 feature 及 layer 分 module，可以每個 feature 都有對應的 layer module。即是Feature A 會有 Feature A 的 presentation、domain 和 data module。但這樣就會有太多 module。折中方案可以是 presentation 和 domain layer 就按 feature 劃分，data layer 就所有 feature 共用同一套 module。

我覺得如果是打算把現有 project 做 modularization 的話，按 layer 劃分或者是折中方案或許比較好。因為很多時候開始做的時候都是對全局沒有太深入的了解，未能為意那些地方是好多個 feature 都會同時用到。當你把舊有的 code 改成新寫法之後，對整個 app 有更深入的理解。那時候才決定最終用那款 modularization 還未算遲。因為新 code 都或多或少按 layer 和 feature 劃分，而且應該有 interface 分隔 layer 之間的 dependency，那時候再把 code 搬去另一 module 應該不用再大改現有的 code。

還有另一樣東西要留意是要不要每個 layer 都有對應的 data class 表達要處理的 data。相信大家都看過一些 project 是使用同一個 class 去接收 API response、表達 SQLite database table 的 row、加上 `Parcelable` 用來開啟 Activity/Fragment。如果  API response 的式樣跟你最後使用的需要是一致的話還好，但如果 API response 的式樣需要特別處理才能用的話這類多用途 data class 會很難理解。

## 示範 App

這個示範 app 都是用 MVVM，但就不會做 modularization，Layer 都是用 package 區分，layer 之間都是會有 interface 分隔。而因為列車抵站時間是每分鐘都在變，所以這個 app 不會做 local cache，只需要定時 call API 就可以了，那就把原本 data 和 remote layer 合併成 data layer。如果我們打算由一開到 Fragment 就 call backend API 並將 response 顯示在 UI 上，大概會是這樣：

{{< figure src="demo-app-architecture.png" title="示範 app 的 architecture" >}}

數字是步驟，橙色 Fragment 有個箭頭指向 Layout XML 和 ViewModel 意思是 Fragment 會建立 Binding 和 ViewModel instance。

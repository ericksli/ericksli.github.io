---
title: Missing parts when designing and implementing Android UI
date: 2024-11-24T15:20:00+08:00
tags:
  - Android
  - Jetpack Compose
cover:
  image: cover.jpg
  alt: Missing parts when designing and implementing Android UI
---

昨天出席了 [GDG Hong Kong](https://gdg.community.dev/gdg-hong-kong/) 舉辦的 [DevFest 2024 Hong Kong](https://gdg.community.dev/events/details/google-gdg-hong-kong-presents-devfest-2024-hong-kong/) 並分享了「Missing parts when designing and implementing Android UI」。

這個題目大致分為三個部分：

1. 一般在準備 Figma mockup 及由 mockup 實作成 Android app UI 時的常見問題，並以一 [Figma Community 免費範本](https://www.figma.com/community/file/1116708627748807811/coffee-shop-mobile-app-design)作例子
2. Jetpack Compose accessibility
3. Jetpack Compose UI testing，示範 Compose testing 搭配 Robolectric 及支援 Appium 的貼士

<!--more-->

本身提交題目時只是想講第一部分，但後來大會回覆話不夠技術內容，要我再改。所以才在後面補上 Compose accessibility 和 UI testing 的內容，最後就通過了。但當初報名做講者時還未準備投影片，所以簡介部分寫得很普通（其實整個簡報是由我以前在公司跟同事內部分享時所造的投影片改裝出來，只是那時還未決定要抽多少內容。）

## 內容概要

### 常見問題

第一部分是講準備 Figma mockup 時經常被忽略的地方。這個題目我一直都想拿出來 present，原因是我在不同公司跟不同的 UI/UX designer 合作時幾乎全部 designer 做出來的 mockup 都會找到那些問題：

- 沒有考慮不同 edge case（對那些 domain 了解不夠深入，沒想到這個位置都可以出 error）、屏幕尺寸（即是 responsive/adaptive layout）
- 不了解 Android 的特性（因為他們自己都是用 iOS，沒有花時間閱讀 Android 和 iOS 的文檔）
- 按鈕尺寸過小、顏色對比度不足等 accessibility 問題
- 沒有考慮到 internationalization/localization 問題，例如文本長度能不能塞進去 UI、貨幣、日期時間、度量衡單位、當地文化習慣之類
- 欠缺 UI/UX design 的基礎知識，例如不知道 bitmap/vector graphic 的分別、一般 design principle 例如應把相關的東西放在一起 ([Proximity principle](https://www.nngroup.com/articles/gestalt-proximity/))
- 之前 design team 訂了 design system 的 component 但又不跟從
- 對產品本身 domain 認識不夠深，例如產品是做股票 app 但對買賣股票流程不熟識（這個比較少見，因為這個比較嚴重）

以前看到 mockup 有問題時都有試過跟 designer 解釋一次，但有時候解釋完他們又聽不懂，要我或者其他 frontend developer 同事即場在 Figma 上示範才懂，甚至看完示範都不明白我們的意思。最後我乾脆自己 duplicate 那些 Figma file 到自己帳號再調整到方便自己輸出和計算物件尺寸的樣子，還有是因為沒有編輯權限所以用不到 Figma 的 plugin（那些[生成 vector drawable 的 plugin](https://www.figma.com/community/plugin/797369763563831541/android-vector-drawable) 比起 Figma 內置的輸出功能更方便）。

我覺得問題是 designer 的培訓出了問題，我懷疑有那些經驗或者是有 sense 的 designer 都是經歷過稿件不斷被回退後靠自己摸索到而不是靠 senior 同事指導學懂這些東西。

當然 developer 那邊都會有問題，最常見是不處理 Android 的 configuration change 和 process death 問題。前者有了 AndroidX ViewModel 已經有所改善，但後者就仍然很少人會理會。另一個經典問題是他們用的測試機屏幕尺寸比較大，當他們砌 UI 時內容在一個屏幕內能完全顯示的話就不去包 `ScrollView` 之類，結果用戶在系統設定字體大小到最大時就不能看全部內容。還有是 UI 在 split screen 或 pop-up view 時沒有處理好 `onSaveInstanceState` 所以不能用。以前在公司處理用戶投訴時用戶說找不到按鈕，原因是字體被設成最大但 UI 沒有包 `ScrollView`，所以用戶就找不到那個按鈕，最後客服回覆說要用戶把字體大小改為正常就能看到。但整個客服流程已經花了幾天，用戶等了幾天得來的回覆就是一個這麼廢的內容。

### Jetpack Compose accessibility

Accessibility 是大家經常忽略的部分。隨着歐盟的 [European accessibility act](https://ec.europa.eu/social/main.jsp?catId=1202) 在明年六月生效，相信大家會開始重視 accessibility。這個部分簡單地介紹了一些 accessibility 常用的 modifier 和示範了 checkbox 和 radio button 如何做 accessibility。重點是要用那些 modifier 標註那些 UI component 的 state，例如那個 checkbox 是不是已經勾選了。另外亦分享了使用 Compose 1.7 新出的 [`LinkAnnotation`](https://developer.android.com/reference/kotlin/androidx/compose/ui/text/LinkAnnotation) 在 TalkBack 上遇到的 bug 和解決方法。

其實 Compose accessibility 這個題目是可以講[大半個小時](https://www.youtube.com/watch?v=cgm8qQpoDt0)，我提過的內容只是一小部分。當然還未計算單純 [WCAG](https://www.w3.org/WAI/standards-guidelines/wcag/) 內非 code 實作的部分。如果想了解更多 WCAG 的內容可以到香港政府數字政策辦公室下載[無障礙網頁手冊](https://www.digitalpolicy.gov.hk/tc/our_work/digital_government/digital_inclusion/accessibility/promulgating_resources/handbook/)。

### Jetpack Compose UI testing

這個部分承接了 Compose accessibility，因為 UI testing 主要是 assert UI component 的 state，而那些 state 包含了 checkbox 是不是勾選之類。所以我們要用那些 modifier 把 UI component 的 state 正確地表達出來而不是隨便放一個有剔號的圖片就叫做 checkbox，一來方便做 UI testing，二來又可以給 TalkBack 讀出來。這部分用了上一部分做 checkbox 的 composable function 加工補上 `testTag` 來示範 UI test 的寫法，另外亦示範了如何為 `LazyColumn` 寫 test。

示範完 Compose UI testing 後就簡單地介紹 Compose UI 如何做 Appium UI test。因為 Compose 是用 canvas 畫一次 UI，所以要特別加入 `testTagsAsResourceId = true` 的 `semantics` modifier 把 Compose 內的 `testTag` 轉化成 resource ID，以便在 UI Automator 中能順利找到那些 UI component。最後解釋了 UI Automator Viewer 跟 Android Studio 內置的 Layout Inspector 有甚麼分別和示範了 UI Automator Viewer 的使用方法。

註：[Appium Inspector](https://github.com/appium/appium-inspector) 內看到的 view hierarchy 跟 UI Automator Viewer 看到的是一樣，因為 Appium 是建基於 UI Automator，而 UI Automator 是建基於 accessibility service，所以才說 accessibility 跟 UI testing 有關連。

## 錄影及投影片

{{< youtube ct1b_Xr7IGQ >}}

{{< speakerdeck id="4e00bbedfc0b422e814053153fd5b413" ratio="1.7772511848341233" >}}

[Google Slides](https://docs.google.com/presentation/d/1TXMHUOKZUaMulhDjHIy8HmjCoTpn7yLG62DFy6kq-Bc/edit)

## 後記

這次 DevFest 編排日程編得很奇怪，他們將幾個 Jetpack Compose/Android 相關的 talk 都排在同一個時段。我自己都想聽舊同事 Dharmendra 講導入 Kotlin Multiplatform (KMP) 和 [Yulianti](https://www.linkedin.com/in/yulianti-oenang/) 比較 Flutter 跟 KMP。完了自己的 talk 之後踫到他們聊了一會大家都說想聽對方的 talk 但自己要講 talk 不能離開。

---

## 參考及閱讀更多

### UI layout

- [Support different screen sizes — Android Developers](https://developer.android.com/guide/topics/large-screens/support-different-screen-sizes)
- [Large screen app quality — Android Developers](https://developer.android.com/develop/ui/compose/layouts/adaptive/support-different-screen-sizes)
- [Multi-window support — Android Developers](https://developer.android.com/develop/ui/compose/layouts/adaptive/support-multi-window-mode)
- [Display content edge-to-edge in your app — Android Developers](https://developer.android.com/develop/ui/views/layout/edge-to-edge)
- [UI Design — Android Developers](https://developer.android.com/design/ui)
- [Layout — Apple Developer Documentation](https://developer.apple.com/design/human-interface-guidelines/layout)
- [Add auto layout to a design — Figma learn](https://help.figma.com/hc/en-us/articles/5731482952599-Add-auto-layout-to-a-design)
- [Figma Tutorial: Auto Layout — Master Auto Layout in 15 Minutes](https://www.youtube.com/watch?v=42uQGucQA9o)
- [Migrating from the ClickableText composable to LinkAnnotation](https://joebirch.co/migrating-from-the-clickabletext-composable-to-linkannotation/)

### Internalization and localization

- [Styling internationalized text in Android](https://medium.com/androiddevelopers/styling-internationalized-text-in-android-f99759fb7b8f)
- [Chinese Mobile App UI Trends](https://dangrover.com/blog/2014/12/01/chinese-mobile-app-ui-trends.html)
- [More Chinese Mobile UI Trends](https://dangrover.com/blog/2016/01/31/more-chinese-mobile-ui-trends.html)
- [How Chinese Apps Handled Covid-19](https://dangrover.com/blog/2020/04/05/covid-in-ui.html)
- [中国人和外国人用的 App，为什么差别那么大？](https://www.ifanr.com/app/822846)
- [Test your app with pseudolocales — Android Developers](https://developer.android.com/guide/topics/resources/pseudolocales)
- [Android pseudolocale]({{< ref "android-pseudolocale" >}})

### Accessibility

- [W3C Accessibility Standards Overview](https://www.w3.org/WAI/standards-guidelines/)
- [Web Accessibility Handbook — Digital Policy Office, HKSARG](https://www.digitalpolicy.gov.hk/en/our_work/digital_government/digital_inclusion/accessibility/promulgating_resources/handbook/)
- [European accessibility act](https://ec.europa.eu/social/main.jsp?catId=1202)
- [Google Accessibility](https://www.google.com/accessibility/)
- [Accessibility in Jetpack Compose — Android Developers](https://developer.android.com/develop/ui/compose/accessibility)
- [GitHub: cvs-health/android-compose-accessibility-techniques](https://github.com/cvs-health/android-compose-accessibility-techniques)

### UI testing

- [Test your Compose layout — Android Developers](https://developer.android.com/develop/ui/compose/testing)
- [Testing Compose Multiplatform UI — Kotlin Multiplatform Development](https://www.jetbrains.com/help/kotlin-multiplatform-dev/compose-test.html)

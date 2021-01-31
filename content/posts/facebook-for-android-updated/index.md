---
title: Facebook for Android 出了更新了！
tags:
  - Android
  - Facebook
  - i18n
categories:

date: 2012-12-14T13:11:57+08:00
---

[早前](http://chinese.engadget.com/2012/09/12/zuckerberg-html-5-facebook-mobile-app-mistake/) Facebook  創辦人暨現任 CEO Mark Zuckerberg 於訪談中說到以 HTML5 編寫 [Facebook Android app](https://play.google.com/store/apps/details?id=com.facebook.katana) 是公司決策上最大的錯誤，今日他們終於不再用 WebView + PHP 顯示貼文了！速度的確快了不少。不過郤引來[一眾香港用戶投訴](http://unwire.hk/2012/12/14/android-facebook-no-more-broken-chinese/software/)，指界面會轉用簡體中文，甚至連[《蘋果日報》](http://hk.apple.nextmedia.com/realtime/finance/20121214/51184200)都有報道，指 Facebook app 出現簡體中文是要打中內地市場。

如果你的 Android Facebook app 界面是以簡體中文顯示的話，請檢查你的設備是否使用「中文（香港）」、「中文（香港特別行政區）」之類的語言。如果是的話，請切換到「中文（繁體）」、「中文（台灣）」之類的語言，問題就會迎刃而解。

{{< figure src="Screenshot_2012-12-14-12-16-44.png" title="使用 zh_TW 時會以繁體中文顯示" >}}

為甚麼使用「中文（台灣）」就沒有問題？**原因是 [Android 根本就沒有 zh_HK 這個語言](http://developer.android.com/reference/java/util/Locale.html)，zh_HK 只是一眾手機平板廠商自作聰明加上去的。**Android SDK 的 emulator 都只有「中文（繁體）」和「中文（簡體）」。但廠商自行加了 zh_HK 後，郤沒有做好如果 app 沒有 zh_HK 的界面語言的話要怎樣 fallback。結果用戶將設備語言設成「中文（香港）」後，只有原生的 app 才會出現繁體中文界面，其他 app 甚至是內置的 Google apps（例如 Google Play、Gmail）只會出現英文界面。Sony 更將中文輸入法與語系綑綁，如果要輸入香港字的話，就必須要將系統語言設為「中文（香港）」，使用「中文（台灣）」就不能輸入。

{{< figure src="Screenshot_2012-12-14-12-17-18.png" title="Sony Xperia ion 的語言設定" >}}

正因為 Android 本身就沒有 zh_HK，相信只有香港的 Android app developer 才會知道香港用戶有機會會使用 zh_HK 這個語系，開發 app 時會加上 zh_HK 界面語言，所以大多數的 Android app 就算有配上 zh_TW 的界面語言，如果用戶將自己的系統語言設成 zh_HK 的話都很可能只看到英文，又或者會像 Facebook app 一樣出現簡體中文。

可能當時 Android 的開發團隊當時未有考慮到部分地區例如香港會使用多過一種語言，不像美國般只用英文，所以將系統的地區和語言合併來設定。如果系統將地區和語言是分開來設定的話，那就不會出現今日 Facebook app 的情況，廠商又不用自作聰明地加上 zh_HK 了。在目前的情況，如果想用繁體中文介面的話，我還是建議大家用 zh_TW，不要再用 zh_HK。（我自己都是用 zh_TW 的）

_12 月 15 日補充：本地 Android app 開發者小倫在其網誌[小倫角落](http://gamehk.blogspot.hk/2012/12/android-locale-problem-for-hong-kong.html "Android app locale settings suggestions")分享了如何整理 Android app 的界面語言。_

---
title: 萬用 Preprocessor Compiler：Prepros
tags:
  - CSS
  - LESS
  - SASS
categories:
date: 2013-06-08T12:53:35+08:00
---

[Prepros](http://alphapixels.com/prepros/) 是一個能支援多款 CSS、JavaScript 和 HTML Preprocessor 的 compiler，它支援 LESS、Sass、Scss、Stylus、Jade、Slim、CoffeeScript、Haml 和 Markdown，有些名我還是第一次聽。有了它就不用再裝 Ruby 之類的東西，因為它已經內置 compile 時所需的程式。它可以在 compile 的時候亦可以 minify code（minify 是指將檔案中不必要的空白字元、註解清除，令檔案變小，加快網頁載入的時間），亦支援近年流行的 Live Reload 功能（在檔案儲存時瀏覽器會即時重新載入有關的網頁，方便即時看到剛才修改過的效果，只需要安裝 Prepros 的 extension 和在 Prepros 開啟這個功能就可以了）。它支援的 Preprocessor 數量和功能可以媲美 Mac 的 [CodeKit](http://incident57.com/codekit/)。不過 Prepros 不能直接 minify 最原始的 JavaScript 檔案，亦沒有優化圖片的功能，但都足夠一般的使用。重點的是 Prepros 是免費的！

<!--more-->

{{< figure src="6-8-2013-11-07-40-AM.png" title="Prepros 的界面簡潔，布局還有幾分像 CodeKit" >}}

我自己只是 CSS 才用 preprocessor，因為寫傳統的 CSS 如果 style 一多檔案就會好長，而且那些 CSS selector 不時都會重複使用，令到 CSS 變得很累贅。用了 Preprocessor 之後因為它們有 Variable、Mixin、Nesting 等等的功能，令到寫 CSS 有寫 program 的感覺。它令到寫 CSS 和日後修改設計變得更方便，例如將網頁的主色都寫到 variable，日後改顏色的話就只需要改 variable。Mixin 即是類似 programming 的 function，可以有 parameter。你可以將那些 CSS3 的新 properties 寫入一個 mixin，只要一調用這個 mixin 就會為你加上 W3C、WebKit、Mozilla、IE 對應的 properties，這樣就不需要每次都要同時寫幾個 vendor-specific properties。（其實 vendor-specific properties 的 mixin 是不用自己寫的，因為已經有現成。Sass / Scss 有 [Compass](http://compass-style.org/)；LESS 有 [LESS Hat](http://lesshat.com/)）如果是做 Responsive Layout 的話，一般的 CSS Preprocessor 可以讓你將 `@media` 寫到 selector 的 `{ }` 入面（稱為 `@media` bubbling），這樣可以將相關的東西集中起來，方便修改，不用再自己另外找個位放 `@media` 的部分。要解決 CSS 檔過長的問題，可以將不同的部分儲存成一個個 LESS / Sass / Scss 等 Preprocessor 檔案。之後再開一個 Preprocessor 檔案將其餘的檔案用 `@import` 來加入到這一個檔案中。最後用 compiler 將這一個檔案轉成 CSS。這樣做就可以生成到一個包含了所有 CSS Rule 的檔案，而 [Twitter Bootstrap](http://getbootstrap.com) 也是用這個方法。

我自己試過用 Scss + Compass 和 LESS + [Twitter Bootstrap](http://getbootstrap.com)，兩者其實都是差不多。初次使用 CSS Preprocessor 可以考慮使用 Scss 和 LESS，因為它們都是直接衍生自傳統 CSS，syntax 和傳統 CSS 最相近。一個正確的傳統 CSS 檔案本身已經是一個正確的 Scss / LESS 檔案。如果想了解 Sass、Scss 和 Compass 的話，可以看看 Brandon Mathis 的 [Sass &amp; Compass: The future of stylesheets now](http://speakerdeck.com/u/imathis/p/sass-compass-the-future-of-stylesheets-now)。用過 CSS Preprocessor 之後保證愛不釋手，不會再想寫傳統的 CSS 了！

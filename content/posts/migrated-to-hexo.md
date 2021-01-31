---
title: 轉用 Hexo
tags:
- Hexo
date: 2016-03-21T00:17:40+08:00
---


之前一直都想用 static site generator 來取代沿用的 WordPress。現在終於由 WordPress 轉到 Hexo，並且將網站改用 GitHub Pages 架站。

Hexo 是一個用 Node 寫的 static site generator，它的定位是用來造 blog，不過用來做一些簡單的文檔網站都可以。近年開始流行如 Jekyll 之類的 static site generator，主要的特色除了是可以將網站放到普通的 web hosting 外，就是可以用 Markdown 寫文章。用 Markdown 的好處就是語法比 HTML 簡單，尤其適合寫一些文字為主，不需要特別排版的文章。還有就是寫一些夾雜着程式碼的文章。Static site generator 通常都會提供 syntax highlight 功能，而且還會將生成的 syntax highlight HTML 直接匯出，無須在 client side 做 syntax highlight。之前在 WordPress 寫夾雜着不少程式碼的文章時就要不時切換 HTML 和 WYSIWYG editor 來補加 `<code>` 之類的 tag，用了不少時間才寫完。

寫文章方面，因為匯出 HTML 需要先安裝 Hexo，所以不能像 WordPress 般方便。我的做法是在 Cloud9 開一個 workspace 來寫文。這樣就不用每部機都要裝一次 Hexo，而且下一次開啟時還能保留 editor 和 shell 的 context。寫文時亦可以開啟 server 來預覽網站。完成後就可以 deploy 到 GitHub Pages。如果嫌沒有網頁寫作介面的話可以安裝 [hexo-admin](https://github.com/jaredly/hexo-admin) plugin。

轉移過程最麻煩的地方就是要人手逐篇文章去檢查轉換完的 Markdown。雖然 Hexo 有提供 WordPress 轉移 plugin，但因為它的轉換插件只是作簡單的轉換，沒有特別去 escape Markdown 語法的字符。加上它沒有將上載到 WordPress 的圖片下載一次。所以要逐張圖片人手處理。有些文章實在有太多圖片，我在轉移的時候只保留部分的圖片。留言方面，因為先前已經轉用 Disqus，所以只需要在設定檔填上 Disqus ID 就可以在文章顯示以前的留言。

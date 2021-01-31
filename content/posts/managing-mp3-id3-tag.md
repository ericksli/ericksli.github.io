---
title: 整理 MP3 ID3 tag
date: 2015-05-17T22:19:27+08:00
tags:
- MP3
---

之前有整埋 MP3 的 ID3 tag，以下是我的小分享。

如果要整理 MP3 檔的 ID3 tag 的話，可以用 [Mp3tag](http://www.mp3tag.de/en/)。在設定頁，揀選 Tags | Mpeg，在 Write 部分只剔選 ID3v2，而下面就只揀選 ID3v2.3 UTF-16。而在 Remove 部分則剔選 ID3v1 及 APE。這樣就應該不會再有亂碼。

如果要移除其他 tag（例如歌詞、iTunes 遺留下來的 tag），可以在 Mp3tag 的檔案欄的項目按右鍵，然後點選 Extended tags。歌詞的 tag 是 `UNSYNCEDLYRICS`。

iTunes 都可以整理 MP3 的 ID3 tag，但預設是不會將 tag 的改動寫回 MP3 檔內，如果沒有特別處理過的話在其他 player（例如電話）播放就未必見到改動過的 ID3 tag。

如果要補回專輯的 cover art 的話，除了 Google image search 之外，iTunes Store 本身都有提供大量 cover art。除了用 iTunes 來自動補回 cover art 之外，也可以用 [iTunes Artwork Finder](http://bendodson.com/code/itunes-artwork-finder/)。只需揀選 Album，輸入專輯名並選擇 iTunes Store 地區就可以開始搜尋。網站會提供 600 px 和 1500 px 的圖像。只需右鍵複製再在 Mp3tag 的 Cover 欄右鍵貼上就能替歌曲補回 cover art。

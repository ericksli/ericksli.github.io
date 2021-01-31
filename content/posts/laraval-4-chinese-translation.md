---
title: Laraval 4 中文語系翻譯
tags:
  - Laravel
  - PHP
  - i18n
date: 2013-06-06T21:57:20+08:00
---

Laravel 4 不再附帶英文以外的 validataion error message 語系翻譯，如果要用其他語系的話，就要自己另外下載。Caouecs 已經在 GitHub 開了一個 [repository](https://github.com/caouecs/Laravel4-lang) 收集不同的翻譯，我自己也譯了繁體中文，大家可以下載需要的翻譯。如果是打算做中文介面的話，最好也要將各欄位的中文名加到 _app/lang/XX/validation.php_ 內的 `attributes` array 內。因為 Laravel 本身是用 input 的 name 來做 attribute 的名稱，所以如果不自訂 `attributes` array 的話顯示錯誤訊息時會中英夾雜。

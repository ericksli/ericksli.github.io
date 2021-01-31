---
title: ConEmu
tags:
  - Git
date: 2013-12-29T23:17:40+08:00
---

最近開始用了一些需要用 command 執行的工具（Grunt 和 PHPUnit），發現一個可以支援顏色顯示的 console 會比較方便。[ConEmu](https://code.google.com/p/conemu-maximus5/) 是一個在 Windows 上使用而且有分頁功能的 console，和 Mac 的 iTerm 有點相似。它的特色就是可以讓你自由地設定各項細節。由字型、字體大小到啟動時開啟那一個目錄都能設定到。它除了可以調用 Windows 的 Command Prompt (cmd.exe) 之外，還可以調用 PowerShell、Bash 等等的 shell。如果用 Bash 的話可以用 Git 提供的版本，這個版本可以支援基本的 Bash 指令，用來執行 PHP CLI 程式、Grunt、PHPUnit 都有顏色顯示。

{{< figure src="8-26-2014-9-20-38-PM.png" title="在 ConEmu 內使用 Git Bash" >}}

ConEmu 的基本設定可以參考 [scotch.io](http://scotch.io/) 的[《Get a Functional and Sleek Console in Windows》](http://scotch.io/bar-talk/get-a-functional-and-sleek-console-in-windows)，內有介紹如何設定一個使用 Git Bash 的 Task（即是設定檔之類的東西）和介面。

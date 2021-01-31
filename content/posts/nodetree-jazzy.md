---
title: nodetree 和 jazzy
tags:
- iOS
date: 2016-12-07T23:27:14+08:00
---


這次想介紹兩個 documentation 相關的工具﹔[nodetree](https://www.npmjs.com/package/nodetree) 和 [jazzy](https://github.com/realm/jazzy)。

nodetree 是用來生成 ASCII 樹狀目錄結構圖。[先前一篇有關 iOS 的文章](/2016/09/cocoapods-framework-pod)就是用 nodetree 生成相關的樹狀目錄結構圖。它的用法非常簡單，只需要輸入 `nodetree` 指令就能印出目前目錄的結構。

而 jazzy 就是一個 Swift 和 Objective-C 的文檔生成器。它和 Java 的 JavaDoc 功能相近，都是按照源碼檔案中以特定格式輸入的註解來生成 HTML 網頁。如果是 Objective-C 的話，註解形式和 JavaDoc 相近。而生成出來的 HTML 網頁外觀和 Apple 官方的 API documentation 非常相似。如果不喜歡的話還可以自訂範本。

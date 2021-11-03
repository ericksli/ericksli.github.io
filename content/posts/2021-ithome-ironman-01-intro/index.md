---
title: "2021 iThome 鐵人賽 Day 1：Intro"
tags:
  - 2021 iThome 鐵人賽
  - Android
date: 2021-09-16T00:00:00+08:00
canonicalURL: https://ithelp.ithome.com.tw/articles/10263963
---

> 本篇文章是 [2021 iThome 鐵人賽](https://ithelp.ithome.com.tw/2021ironman)參賽題目「[寫一個列車抵站時間 Android App](https://ithelp.ithome.com.tw/users/20139666/ironman/4661)」的第 1 篇，你可到 iThome [查看原文](https://ithelp.ithome.com.tw/articles/10263963)。
>
> [文章目錄]({{< ref "2021-ithome-ironman" >}})

早陣子（2021 年 6 月 27 日）[港鐵屯馬綫](https://zh.wikipedia.org/zh-hk/%E5%B1%AF%E9%A6%AC%E7%B6%AB)全綫通車，當日有電視台訪問了一名鐵路迷，他受訪時調寄家傳戶曉的英國民謠《綠䄂子》即興唱了一句自創歌詞「[屯馬開通真的很興奮](https://evchk.wikia.org/zh/wiki/%E5%B1%AF%E9%A6%AC%E9%96%8B%E9%80%9A%E7%9C%9F%E7%9A%84%E5%BE%88%E8%88%88%E5%A5%AE)」。那我們就以港鐵實時列車抵站時間做例子，示範一些現在 Android 開發流行的東西。

App 會有兩頁：選擇車站頁和抵站時間頁。選擇車站頁就是列出某條行車綫的車站，讓用戶點選查閱該站的列車抵站時間；抵站時間頁就是顯示某行車綫及車站的列車抵站時間。

{{< figure src="wireframe.png" title="Wireframe" >}}

選擇用這個 app 做示範是因為我本身就想做這個功能（因為我[本身有一個 app 是做這東西](https://play.google.com/store/apps/details?id=net.swiftzer.metroride)），另一個原因是顯示抵站時間頁不是單純的直接顯示伺服器傳回的內容，要有一些後期的處理和在用戶介面上需要分幾個狀態顯示內容。

在介紹如何做的同時我會順帶補充選用這個做法的原因和一些個人心得，藉此能提供一些官方使用說明沒有講的東西。

## 大概會包含的組件

- Architecture Components
- Data binding
- RecyclerView
- Navigation Components
- Kotlin Coroutines
- Kotlin Flow
- Ktor Client
- Kotlin serialization
- Dagger Hilt

如果後段時間允許的話，或許會再加上 Compose 相關的內容。

## 對象

這個文章系列適合對 Kotlin 和 Android 開發有基本認識的人，因為文章不會每樣東西都解釋，亦不會每個步驟都附有截圖。簡單來說就是能寫到一個簡單的 Android app。（本身已經懂得 Activity、Fragment、RecyclerView 這些東西）

如果本身聽過上面那堆組件名稱但又不太知道實際怎樣用的話，這系列或許能幫到你。

## GitHub Repository

示範 app 的源碼已經放在 [GitHub](https://github.com/ericksli/eta-demo)。

## 註

由於這篇文的讀者絕大部分都不是香港人，所以在這裏補充一下。「屯馬開通真的很興奮」之所以變成香港網路上的 meme 是因為調寄《綠䄂子》。《[綠䄂子幻想曲 (Fantasia on Greensleeves)](https://www.youtube.com/watch?v=Y9lailBPbhU)》是香港中文和英文科公開試聆聽考試會聽到的間場音樂，所以很多人每次聽到這首音樂都有不安情緒。

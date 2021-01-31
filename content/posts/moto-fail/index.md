---
title: Moto Fail
tags:
  - Android
  - Motorola
date: 2011-04-22T14:30:31+08:00
---

那時買了 [Milestone](http://www.motorola.com/Consumers/HK-ZH/Consumer-Product-and-Services/Mobile+Phones/Motorola-MILESTONE-HK-ZH?localeId=133) 用了個多月後就發現顯示屏入塵。本來顯示屏入塵幾乎是每部手提電話的通病，而且問題亦不算嚴重到影響使用。但聽聞有用家試過 [Moto](http://www.motorola.com/Consumers/HK-ZH/Home) 會免費清除顯示屏內的塵，所以前天將 Milestone 拿去 Moto 去除顯示屏內的塵，順道去問問 Moto 為何用 Moto 官方出的 [Android](http://www.android.com/) 2.2 更新會收不到 [Google Chrome to Phone](https://market.android.com/details?id=com.google.android.apps.chrometophone) 的 Push notification 和 Moto 輸入法的問題。

{{< figure src="moto-milestone.jpg" title="Milestone 刷了 CM7 的 Home screen" >}}

去到 Moto 維修中心，櫃枱上就有一張 Android 2.2 更新的通告。說明 Milestone 用 2.2 會較以前用 2.1 慢和應用程式容易出現 Force Close。不過我還是問那個服務員為甚麼會特別容易 Freeze（Milestone 官方的 2.2 更新[幾乎全部地區版本](http://and-developers.com/sbf:milestone221)都會有 [<abbr title="Display Serial Interface">DSI</abbr> kernel bug](https://supportforums.motorola.com/message/349764)），他說是因為 Milestone 本身 RAM 比起其他用 2.2 的機不夠多，所以特別容易出現這個問題。而 Chrome to Device 用不到的問題就說是其他應用程式的問題，所以不會回應。但我懷疑是 Milestone 官方 2.2 的 [C2DM Framework](http://code.google.com/android/c2dm/index.html) 出問題，因為之前用 [CyanogenMod](http://www.cyanogenmod.com/) 6 和 CyanogenMod 7 都能收到 Chrome to Phone 的 Push，但用官方 2.2 就永遠都收不到。那個服務員就說會找師傅除塵和更新系統，說要第二天才能取回手機。其實早就知道 Moto 出的 2.2 本身就是爛，所以都不期望更新系統後會得到甚麼改善。

昨日拿回手機後，發現都是亞太版 (SHOLS_U2_05.26.4) 的 2.2，即是說 <abbr title="Display Serial Interface">DSI</abbr> kernel bug 仍然會出現。回家刷了 CM7 後，由於 Android 2.3 的界面有特別多黑色，發現顯示屏還是有幾點塵，不過比除塵前相比少了很多。

今早無意中看見顯示屏背面突然發現多了道花痕。Milestone 出廠時顯示屏背面是有張保護貼，很多人買回來之後都會保留這張保護貼，並將藍色角位剪走以避免保護貼吸塵。這是因為有不少用家發現多次趟動鍵盤的話顯示屏背面的金屬板會有兩條路軌痕。誰不知 Moto 維修時會再用相信是界刀的工具界多一次那個角位，就連表面黑色油漆都界走。在寫這篇文時又再發現鏡頭的環多了兩道壓痕。

{{< figure src="back.jpg" title="機背的花痕" >}}

{{< figure src="camera.jpg" title="相機鏡頭的兩道壓痕" >}}

其實我早就對 Moto 香港失望，Milestone 的 2.2 更新一拖再拖，拖到有歐洲用家要做一個[倒數網站](http://www.motocountdown.eu/)來倒數 Moto 出 2.2 更新的限期。Facebook 的 Moto Europe Fan page 更是充斥着用家的的投訴。Moto 的產品本身就不太差，但市場推廣、售後服務就奇差。Moto 只着重北美及中國內地市場，香港開售的產品比美洲和歐洲都來得遲，來到香港賣都變成「二手科技」。[Atrix](http://www.motorola.com/Consumers/US-EN/Consumer-Product-and-Services/Mobile-Phones/Motorola-ATRIX-US-EN) 和 [XOOM](http://www.motorola.com/Consumers/US-EN/Consumer-Product-and-Services/Tablets/ci.MOTOROLA-XOOM-US-EN.overview) 到現在仍未在香港開售，就算開售但其定價還比美國的貴。加上 Android 系統更新比其他品牌還要遲，Milestone 升級 2.2 都要拖成一年（美國的 [Droid](http://www.motorola.com/Consumers/US-EN/Consumer-Product-and-Services/Mobile-Phones/Motorola-DROID-US-EN) 就很快有更新），拖了一年的 2.2 還要是 2.2.1 而不是 2.2.2。Moto 自己不更新還要鎖 Bootloader，讓其他社區開發團隊都不能修改核心（如果 Moto 不再出更新的話，<abbr title="Display Serial Interface">DSI</abbr> kernel bug 將永遠存在），Moto 客戶經理更口出狂言指[要刷 ROM 就別買 Moto](http://www.androidnoodles.com/2011/01/motorola-said-if-want-to-brush-a-third-party-to-buy-another-mobile-phone-rom/)。現在竟然連維修都有問題！Moto 想靠 Android 來翻身，但第一個翻身作 Milestone 的 2.2 升級就一拖再拖，最後出來還問題多多，令消費者對 Moto 失去信心。真是成也蕭何，敗也蕭何！

---
title: 使用 Exchange ActiveSync 同步 Outlook.com 郵箱
tags:
  - Android
  - Outlook
categories:

date: 2013-01-12T22:51:13+08:00
---

微軟的 [Outlook.com](http://outlook.com) 已經開放了一段時間，如果想在 Android 上收發 Outlook.com 的電郵除了用傳統的 SMTP+POP 或者使用[官方的專用 app](https://play.google.com/store/apps/details?id=com.outlook.Z7) 之外，其實還可以用 Exchange ActiveSync。雖然 Outlook.com 沒有交待過 Exchange ActiveSync 的設定方法，但做法其實不難，只需要用手機內置的電郵 app（不是 Gmail app）再使用本文提供的設置就可以了（方法同樣適用於 Hotmail 郵箱）。

<!--more-->

以下圖片所顯示的界面可能會因為設備生產廠商不同而和你的設備有所出入，但應該大同小異。

首先在電郵 app 裡新增一個帳戶。輸入你的 Outlook.com / Hotmail 電郵地址和密碼，然後按「手動設定」。（留意不是按「下一步」，因為按「下一步」是會用 SMTP/POP 收發電郵。）

{{< figure src="Screenshot_2013-01-12-19-30-38.png" title="輸入電郵地址和密碼" >}}

下一步就是按「Exchange ActiveSync」。

{{< figure src="Screenshot_2013-01-12-19-30-49.png" title="揀選「Exchange ActiveSync」" >}}

之後會轉到內收設定頁，它自動替你填上設定。但我們是需要改動它們才可以用 Exchange ActiveSync。

* **網域\使用者名稱：**你的 Outlook.com / Hotmail 完整的電郵地址（不是電子郵件別名）
* **密碼：**你的登入密碼
* **伺服器：**m.hotmail.com
然後將「使用安全連線 (SSL)」打勾，如下圖所示：

{{< figure src="Screenshot_2013-01-12-19-31-23.png" title="輸入使用者名稱、密碼和伺服器" >}}

完成後按「下一步」，再按照程式的指示完成餘下的步驟就可以新增你的 Outlook.com / Hotmail 郵箱了。

使用 Exchange ActiveSync 除了可以用 Push 的方式接收新郵件外，還可以同步 Outlook.com / Hotmail 的通訊錄和行事曆。不過我只是用來同步郵箱，沒有試過同步我的 Outlook.com 通訊錄和行事曆，相信是可以同步到的。除此之外，使用 Exchange ActiveSync 可以免郤安裝 Outlook.com 專用 app，那個 app 的界面非常普通，還是維持住 Android 2.3 的設計。如果你需要接收其他郵箱（例如 Yahoo、ISP）的話，使用內置的電郵程式可以在合併收件箱閱讀所有郵件，郵件會按時間排列好。你還可以為不同的帳號設定好代表顏色，方使在合併收件箱辨認。

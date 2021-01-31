---
title: 資料一線通
tags:
  - Open data
  - API
  - PTES
date: 2011-04-11T12:38:40+08:00
---

香港政府近年來積極推廣電子政府。除了推出「[香港政府一站通](http://www.gov.hk/)」、「[地理資訊地圖](http://www.map.gov.hk)」、「[公共交通查詢服務網站 (PTES)](http://ptes.td.gov.hk/)」之外，最近還推出「[資料一線通](http://www.gov.hk/tc/theme/psi/)」服務，將公共資料公開讓大眾使用。

資料一線通目前提供公共設施的地理參考數據和主要道路的實時交通資料，供市民和機構免費下載使用，就算商業使用都是免費。地理參考數據就是各公共設施（學校、醫院、文娛康樂設施等）的 CSV 格式位置資料，例如經緯度、電話、地址等。而實時交通資料就有運輸署提供的行車速度圖、平均過海行車時間及特別交通消息，實時交通資料更提供 XML 格式供開發人員使用。網站還提供了開發說明和 Java 示範程式供開發人員參考。

其實資料一線通就是為開發人員提供 [API (Application Programming Interface)](http://zh.wikipedia.org/wiki/%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F%E6%8E%A5%E5%8F%A3)。API 是 [Web 2.0](http://zh.wikipedia.org/wiki/Web_2.0) 的關鍵元素之一，不少網上服務和機構早已開始提供 API 供人免費使用。例如 [Google Maps](http://code.google.com/apis/maps/index.html)、[Twitter](http://dev.twitter.com/)、[Facebook](http://developers.facebook.com/)、[倫敦交通局 (Transport for London, TfL)](http://www.tfl.gov.uk/businessandpartners/syndication/) 等。但香港政府和機構就慢人數倍。以運輸署的 PTES 和地政總署的地理資訊地圖為例，網站載入速度太慢、介面設計過時又難用，兩者的使用者經驗與 Google Maps 等相類似的 Web 2.0 服務相差甚遠。唯一的好處可以說是它們的資訊是第一手資料，比起 Google Maps 要用地圖王、MapABC 等二手資料準確得多。現在政府免費提供資料供人使用就可以無須自行開發這些服務網站 / 應用程式。只要有人開發的話，不論是不同平台的電腦、手機、平板還是網上應用程式都可以有。完全不用擔心政府提供的服務追不上科技潮流。

但要擔心的是資料一線通所使用的伺服器能否處理大量的查詢。PTES 初初啟用時經常出現錯誤就已經是一個先例。倫敦交通局在去年初初推出即時地鐵路線服務消息、列車抵站消息、Journey Planner 的 Data Feed 時就出現伺服器負荷過大而要暫停公開 Data Feed。後來轉用 [Windows Azure](http://www.windowsazure.com/) 雲端運算平台[才能繼續提供服務](http://blogs.msdn.com/b/windowsazure/archive/2010/12/13/transport-for-london-moves-to-windows-azure.aspx)。如果資料一線通並非使用如 Azure、[Amazon EC2](http://aws.amazon.com/ec2/)、[Google App Engine](http://code.google.com/appengine/) 之類的雲端運算平台的話，很有可能會重蹈 PTES 和 TfL 的覆轍。

最後，緊記填寫資料一線通的[網上意見調查](http://www.gov.hk/tc/theme/psi/feedback/)，向政府表達在 18 個月的試驗計劃後要繼續免費開放更多資料和 API，方便使用資料的市民和開發人員。

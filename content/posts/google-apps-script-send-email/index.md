---
title: 用 Google Apps Script 發送電郵
tags:
  - Google
date: 2019-04-14T15:08:36+08:00
---


在[上一篇]({{< ref "posts/google-apps-script-create-calendar-event" >}})為大家介紹了如何用 [Google Apps Script](https://developers.google.com/apps-script/) 建立 Google Calendar event。這一次就示範用 Google Apps Script 發送電郵（即是 mail merge）。

<!-- more -->

## 準備內容

上次我們用 2019 年香港公眾假期作例子，這次我們用得獎名單做例子。

{{< figure src="mail-list.png" title="得獎名單" >}}

第二及第三欄是我們會在電郵內文用到；Sent By 和 Sent At 兩欄是用來判別這封電郵是否發送過。如果已經發送過就會標明是由誰人執行和執行時間。

## 準備內文範本

和之前一樣，都是用 Google Apps Script 的 HTML template 功能。

{{< highlight html >}}
<p>Dear <?= name ?>,<p>

<p>Congratulation! You won our smartphone giveaway. Your <?= prize ?> will be shipped to you within one month.</p>
{{< /highlight >}}

## 簽名檔 / 加入圖片（非必須）

使用 `GmailApp` 發送電郵是不會把你 Gmail 所設的簽名檔加到電郵內。如果你想在電郵內出現簽名檔的話，需要在 HTML template 內加入簽名檔的 HTML code。留意電郵軟件未必支援新式的 HTML/CSS，所以電郵排版都是用 HTML `<table>` 和用 inline CSS。

如果想加入圖片的話，有兩個方法：

1. 直接連結寄存在其他地方的圖片
2. 將圖片變成電郵附件，再顯示在電郵的內文

第一個方法和平時做網頁差不多，第二個方法比較複雜。如果需要用第二個方法，可以參考下面的做法。

首先，將圖片上載到 Google Drive。接着就用 script 把這些圖片檔變成 [`Blob`](https://developers.google.com/apps-script/reference/base/blob)。以下的 function 示範了如何抽取在 `EMAIL_SIGNATURE_IMAGE_FOLDER_ID` 入面的檔案的 `Blob`。Folder ID 其實就是用網頁版 Google Drive 開啟 folder 後在網址見到的一串 ID。

{{< highlight js >}}
function extractEmailSignatureBlobs() {
  var result = {}
  var files = DriveApp.getFolderById(EMAIL_SIGNATURE_IMAGE_FOLDER_ID).getFiles()
  while (files.hasNext()) {
    var file = files.next()
    result[file.getName()] = file.getBlob()
  }
  return result
}
{{< /highlight >}}

在 HTML template 內，可以在 `<img>` 的 `src` 加上 `cid:` 前綴來顯示 inline 圖片：

{{< highlight html >}}
<img src="cid:imageKey" />
{{< /highlight >}}

`imageKey` 就是剛才 `extractEmailSignatureBlobs` return 的 object 的 key。例如在 Google Drive 的 folder 有一個 *foo.png*，那 `src` 就會是 `cid:foo.png`。

## 發送郵件或建立草稿

`createDraft` 會在你 Gmail 草稿箱內新增一封電郵，不會把它發出。

{{< highlight js >}}
GmailApp.createDraft(rowValues[0], subject, html, {
  htmlBody: html,
  inlineImages: inlineImages
});
{{< /highlight >}}

`inlineImages` 就是 `extractEmailSignatureBlobs` 的 return value。如果沒有加入圖片就不用加。

如果想馬上發送郵件，那就把 `createDraft` 換成 `sendEmail`。

下面是發送郵件的程式碼：

{{< highlight js >}}
function sendEmail() {
  var sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName('Mail merge');
  var range = sheet.getRange(2, 1, sheet.getMaxRows() - 1, 5);
  var values = range.getValues();
  var template = HtmlService.createTemplateFromFile('email');
  
  for (var row in values) {
    var rowValues = values[row];
    var rowIndex = parseInt(row) + 2;
    
    // Early termination when no email address found in a row
    if (isCellEmpty(rowValues[0])) {
      return;
    }
    
    // Skip sent row
    if (!isCellEmpty(rowValues[4])) {
      continue;
    }
    
    template.name = rowValues[1];
    template.prize = rowValues[2];
    
    var html = template.evaluate().getContent();
    var subject = rowValues[1] + ', you are the winner!'
    GmailApp.createDraft(rowValues[0], subject, html, {
      htmlBody: html
    });
    
    // Write the user's email address and timestamp to that row
    sheet.getRange(rowIndex, 4).setValue(Session.getActiveUser().getEmail());
    sheet.getRange(rowIndex, 5).setValue(new Date());
  }
}
{{< /highlight >}}

基本上就是一個 loop 逐行檢查，如果需要發電郵就用 `GmailApp` 製造郵件放到自己 Gmail 的草稿箱。

執行完成後會看到 Sent By 和 Sent At 兩欄經已被填妥。下次再執行時程式會檢查 Sent At 是否為空，那就不會重覆發送郵件。

{{< figure src="sent-mail-list.png" title="執行完成後的得獎名單" >}}

## 完整程式碼

Google Apps Script 程式碼、HTML 和試算表內容已經放到 [Gist](https://gist.github.com/ericksli/905e026e3db37955584d9845599b7b71) 上。

## 下一步

之後可以考慮加入[自訂選單](https://developers.google.com/apps-script/guides/menus)。有了自訂選單就不用每次都要進入 Script editor 才能執行程式。

## 結語

上一篇和這一篇分別介紹了 Google Apps Script 操作 Spreadsheet、Gmail、Drive 和 HTML template 功能。你可以結合一起使用，提昇工作效率。例如 Spreadsheet 內容由 Form 形式輸入、使用 Maps Service 做 Geocoding 等等。此外，Google Apps Script 的 function 也可以[當 spreadsheet 的 function 使用](https://developers.google.com/apps-script/guides/sheets/functions)。如果你常常使用 Google Drive 的辦公室軟件功能的話 Google Apps Script 實在不容錯過。

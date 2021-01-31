---
title: 用 Google Apps Script 建立 Google Calendar event
tags:
  - Google
date: 2018-10-17T14:49:46+08:00
---


[Google Apps Script](https://developers.google.com/apps-script/) 是一套以 JavaScript 造的 API，可以讓你寫程式控制 Google Apps 內 Docs、Spreadsheet、Gmail、Drive 等等的功能。這次介紹如何用 Google Apps Script 建立 Google Calendar 的 event。我們會把部分建立 event 時所需要的資料放入 spreadsheet 內（會以 2019 年香港公眾假期作例子），然後用 Google Apps Script 讀取 spreadsheet 的內容再建立 event，效果就像 mail merge 般。

<!-- more -->

## 準備內容

首先，去 Google Drive 建立一個 Spreadsheet。

{{< figure src="holiday-sheet.png" title="我們會將這些假期加到 Google Calendar 內" >}}

## 找出 Calendar ID

因為每人的 Google Calendar 可以有多於一個 calendar，所以第一件事就是要找到你想加入 event 那一個行事曆的 calendar ID。按 Tools > Script editor 進入 Google Apps Script 的編輯工具。

{{< figure src="spreadsheet-menu-script-editor.png" title="進入 Google Apps Script 的方法" >}}

這個編輯工具左邊是檔案列表，右邊是讓你修改程式碼。

我們首先試試下面的一個 function：

{{< highlight js >}}
function listAllCalendars() {
  var calendars = CalendarApp.getAllCalendars();
  calendars.forEach(function (c) {
    Logger.log(c.getName() + " >>> " + c.getId())
  })
}
{{< /highlight >}}

在工具列，有一個選單，是用來選擇執行的 function。

{{< figure src="run-script.png" title="執行 function 的方法" >}}

選擇「listAllCalendars」，然後按 Run，你首先會看到 Google 要求授權窗口，授權後你就會看到「Running function listAllCalendars...」。這表示 `listAllCalendars` 正在執行中。

{{< figure src="running-script.png" title="執行中" >}}

執行完成後它會自動消失。消失後你可以按 <kbd>Ctrl</kbd> + <kbd>Enter</kbd> 開啟 Logs 窗口，你可以看到經由 `Logger.log` 記錄下來的 log。在 `listAllCalendars` 中，我們把 calendar 的名稱和對應的 ID 都記錄下來。因為之後建立 calendar event 是要利用到 calendar ID，所以你要把 calendar ID 抄起來備用。

{{< figure src="apps-script-log.png" title="Logs 窗口" >}}

## 準備 event 描述範本

接着就返回 Script editor。Google Calendar 的 event 描述可以用一些基本的 HTML 標籤（例如 `<a>`、`<b>`、`<br>`、`<ul>`、`<li>`）。這次我們會用到 Google Apps Script 的 HTML template 功能。

在選單中揀選 File > New > HTML file，之後輸入名稱「Calendar Template」。

{{< figure src="apps-script-create-html.png" title="建立 HTML template" >}}

[`HtmlService`](https://developers.google.com/apps-script/reference/html/html-service) 的 template 功能主要是用 `<?  ?>` 和 `<?=  ?>` 兩款 tag。如果要用到 `if`、`for` 的話，就可以這樣做：

{{< highlight html >}}
<? if (foo == 1) { ?>bar<? } ?>
{{< /highlight >}}

如果要輸出 `bar` 的內容（連 HTML escape），可以用：

{{< highlight html >}}
<?= bar ?>
{{< /highlight >}}

我們會在每個 event 的描述顯示假期的中英文名稱。雖然看起來沒有甚麼用，但可以簡單示範這個功能。

{{< highlight html >}}
<b>英文名：</b> <?= enTitle ?>
<br><b>中文名：</b> <?= zhTitle ?>
{{< /highlight >}}

留意一點就是 Google Calendar 的 event 描述如果用 HTML 的話，它對空白字符的處理方式和網頁瀏覽器不同。網頁瀏覽器會將多於一個空白字符當成一個空白字符顯示（除非用 `&nbsp;`）；但 Google Calendar 就會把所有空白字符顯示。所以如果講究排版效果的話就可能會令 HTML template 的 HTML code 混亂一點。

## 讀取試算表內容

做好 HTML template 之後我們就要寫一個 function 把開首準備好的內容讀取並變成 JavaScript 的 object array。

{{< highlight js >}}
function extractHolidays() {
  var sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName('Holidays')
  var values = sheet.getRange(2, 1, sheet.getMaxRows() - 1, 3).getValues()
  var results = []
  for (var row in values) {
    var rowValues = values[row]
    // early termination if empty row is found
    if (rowValues[0] === '' || rowValues[1] === '' || rowValues[2] === '') {
      break
    }
    results.push({
      zhTitle: rowValues[0],
      enTitle: rowValues[1],
      date: rowValues[2],
    })
  }
  return results
}
{{< /highlight >}}

`SpreadsheetApp.getActiveSpreadsheet().getSheetByName('Holidays')` 其實就是取得當前檔案中的「Holidays」試算表的 [`Sheet`](https://developers.google.com/apps-script/reference/spreadsheet/sheet)。拿到 `sheet` 之後就可以讀寫試算表內的 cell。

下一步就是要讀取指定範圍的 cell，那就是用 [`getRange(row, column, numRows, numColumns)`](https://developers.google.com/apps-script/reference/spreadsheet/sheet#getRange(Integer,Integer,Integer,Integer)。

- `row` 填 `2` 是因為我們第一行是標題，第二行開始才是內容
- `column` 填 `1` 是因為我們在 Column A 開始就是內容（由 1 開始數起）
- `numRows` 填 `sheet.getMaxRows() - 1` 即是我們要 sheet 的全部行，但扣除第一行標題（實際內容沒有那麼多行，所以之後會有 early termination 的部分）
- `numColumns` 填 `3` 是因為我們要讀取的內容是由 Column A 至 C，共三列

之後就做一個 JavaScript object 來儲起每筆記錄，方便之後使用。留意 Column C 我們的格式是 `DateTime`，Google Apps Script 會自動取得 JavaScript `Date` object。時區可參考 Spreadsheet 的 Spreadsheet settings 及 Script editor 的 Project properties。最後把生成的 array return 就行了。

## 把 event 加到 calendar 內

最後的部分就是建立 Google calendar 的 event。

{{< highlight js >}}
var CALENDAR_ID = 'xxxxxxxxxx@group.calendar.google.com'


function createCalendarEvents() {
  var holidays = extractHolidays()
  
  var template = HtmlService.createTemplateFromFile('Calendar Template');
  
  holidays.forEach(function (holiday) {
    var title = holiday.zhTitle
    template.zhTitle = holiday.zhTitle
    template.enTitle = holiday.enTitle
    var html = template.evaluate().getContent()
    CalendarApp.getCalendarById(CALENDAR_ID).createAllDayEvent(title, holiday.date, {
      description: html
    })
  })
}
{{< /highlight >}}

`CALENDAR_ID` 就是最初抄下來的 calendar ID；`holidays` 就是剛才讀取 spreadsheet 的部分；`template` 就是建立之前準備好的 HTML template。

然後就為每筆記錄（公眾假期）套進 HTML template 並生成 HTML code，之後就用 [`createAllDayEvent(title, date, options)`](https://developers.google.com/apps-script/reference/calendar/calendar-app#createalldayeventtitle-date-options) 來建立 all-day event（因為公眾假期是全日的，如果你的 event 不是全日的話就改用 [`createEvent(title, startTime, endTime, options)`](https://developers.google.com/apps-script/reference/calendar/calendar#createEvent(String,Date,Date,Object)) 或其他 method）。

HTML template 要把 template 內用過的 variable（即是 `zhTitle` 和 `enTitle`）設定好才可以生成 HTML code（`template.evaluate().getContent()` 這一句）。

最後執行 `createCalendarEvents` 就可以建立 event。

{{< figure src="google-calendar-event-created.png" title="成品效果" >}}

{{< figure src="edit-calendar-event.png" title="編輯 event 會看到 HTML 格式的內容" >}}

留意每執行一次 `createCalendarEvents` 就會建立新的 event 而不是修改現有的 event。

如果你想在再執行時跳過已建立過的 event 的話，可以在 spreadsheet 加多一個 column 標記來記錄這一筆資料是否已經建立過 event。下面兩句可供參考（記錄當前用戶電郵地址、時間）：

{{< highlight js >}}
sheet.getRange(row, column).setValue(Session.getActiveUser().getEmail())
sheet.getRange(row, column).setValue(new Date())
{{< /highlight >}}

## 完整程式碼

Google Apps Script 程式碼、HTML 和試算表內容已經放到  [Gist](https://gist.github.com/ericksli/662e2edf7f5bd327990960ed03029bdf) 上。

[下一篇]({{< ref "posts/google-apps-script-send-email" >}})會講解利用 Google Apps Script 發送電郵（即是 mail merge）。

---

## 補充

在處理時間時難免會想用 [Moment.js](https://momentjs.com/) 來格式化、計算時間，又或者用 [Lodash](https://lodash.com/) 之類的 JavaScript library。Google Apps Script 是有機制 (App Script library) 讓你調用這些 library。詳情可以參考 [Stack Overflow](https://stackoverflow.com/a/16928369) 的答案。

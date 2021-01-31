---
title: Linkify 自動轉換成網址
tags:
  - Android
date: 2018-03-10T11:18:56+08:00
---


最近工作需要將不完整的網址變成網址，但輸入的 string 可以是普通的文字，亦可以是一個沒有 `http://` 或 `https://` 的網址，亦可以一個完整的網址。但 TLD 有太多，如果自己寫 regular expression 做檢查的話那句 regular expression 就會好長，而且要定時補上日後新推出的 TLD。

之後查過原來 Android 有 [`Linkify`](https://developer.android.com/reference/android/text/util/Linkify.html)/[`LinkifyCompat`](https://developer.android.com/reference/android/support/v4/text/util/LinkifyCompat.html) 這個 class：

> Linkify take a piece of text and a regular expression and turns all of the regex matches in the text into clickable links. This is particularly useful for matching things like email addresses, web URLs, etc. and making them actionable. Alone with the pattern that is to be matched, a URL scheme prefix is also required. Any pattern match that does not begin with the supplied scheme will have the scheme prepended to the matched text when the clickable URL is created. For instance, if you are matching web URLs you would supply the scheme `http://`. If the pattern matches example.com, which does not have a URL scheme prefix, the supplied scheme will be prepended to create `http://example.com` when the clickable URL link is created.

看起來應該合用，不過一般用法都是用來將 `TextView` 的合適文字變成可點擊的超連結。下面是我的做法：

{{< highlight kotlin >}}
val query = "example.com"
val spannable = SpannableString(query)
LinkifyCompat.addLinks(spannable, Linkify.WEB_URLS)
val linkifyUrl = spannable.getSpans(0, spannable.length, URLSpan::class.java)
    .filter { urlSpan -> spannable.getSpanStart(urlSpan) == 0 && spannable.getSpanEnd(urlSpan) == spannable.length }
    .map { urlSpan -> urlSpan.url }
    .firstOrNull()
{{< /highlight >}}

首先將疑似網址的 string 變成 `SpannableString`。之後用 `LinkifyCompat#addLinks` 來檢查整個 string 有沒有網址。如果有的話，`SpannableString` 中的網址部分會有 span 包住。

這次的要求是整個 `query` string 最多只有一個網址，如果沒有網址或出現多於一個網址的話，就當作沒有網址處理。所以要加上 `filter` 來篩走出現多於一個網址的情況。

最後 `linkifyUrl` 會是 `http://example.com`；如果 `query` 沒有網址的話，那 `linkifyUrl` 會是 `null`。

至於 `LinkifyCompat` 會不會定期更新 TLD 清單，我不太清楚，但當初出現 `LinkifyCompat` 就是用來[補回新出的 TLD](https://www.reddit.com/r/androiddev/comments/52ye0h/heads_up_on_a_new_linkifycompat_that_brings_the/)。

---
title: Inline CSS
date: 2015-03-11T23:04:29+08:00
tags:
- CSS
- Gulp
---

最近需要寫一個發送 email 的 program。如果要用 HTML 的話，最好都是找到現成的 HTML email framework。（例如 Zurb 的 [Ink](http://zurb.com/ink/)，其他的可以參考 [Responsive Email Resources](http://responsiveemailresources.com/) 網站）因為 HTML email 和平時做網頁分別很大，例如：

* CSS 要 inline，`<style>` 並不是全部 email client 都能支援
* 排版要用 `<table>`
* 圖片視乎需要當成附件，Base64 未必能被 email client 支援
但設計 email 時如果要寫 inline style 是非常難寫，所以都是用工具轉。我試過用 [Gulp](http://gulpjs.com/) 配 [gulp-inline-css](https://www.npmjs.com/package/gulp-inline-css) 和 [gulp-inline](https://www.npmjs.com/package/gulp-inline)，效果不錯。<!--more-->

以下是 Gulp 做 inline CSS 的寫法：

{{< highlight javascript >}}
var gulp = require('gulp');
var minifyHTML = require('gulp-minify-html');
var inlineCSS = require('gulp-inline-css');
var inline = require('gulp-inline');

gulp.task('email', function () {
    return gulp.src(['templates/email/src/*.html'])
        .pipe(inline({
            base: 'templates/email/src'
        }))
        .pipe(inlineCSS({
            applyStyleTags: true,
            applyLinkTags: true,
            removeStyleTags: true,
            removeLinkTags: true
        }))
        .pipe(minifyHTML())
        .pipe(gulp.dest('templates/email'));
});
{{< /highlight >}}

`inline` 是用來將 HTML 用 `<style>` 外連的 CSS 檔變成內篏在 HTML 檔的 CSS。之後 `inlineCSS` 會將 `<style>` 的 CSS 變成 inline 在每一個 HTML tag 的 `style` attritube。最後 `minifyHTML` 是用來將多餘的空格清走，令檔案變得更細。將它抽走都不會影響 inline CSS 的效果。如果不用 gulp-inline 而直接用 gulp-inline-css 的話會令到原先用 `<style>` 內篏在 HTML 檔的 CSS 不能 override 外連 CSS 檔的 rule。

有一樣東西要留意：`inline` 和 `inlineCSS` 會將 HTML 檔內本應要 escape 的字符 escape 掉。所以如果 HTML 檔是要配合 template engine 用的話，要留意 template engine 所用的字符會否被 escape 掉。

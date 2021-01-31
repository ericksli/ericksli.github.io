---
title: 在 Windows 刪除路徑名太長的檔案
date: 2015-09-21T23:20:51+08:00
tags:
  - Node.js
  - JavaScript
---

通常寫 JavaScript project 都會用到 NPM，NPM 和其他 package manager 不同之處就是每個 package 的 dependency 都會放在其 package 內的 node_modules 資料夾，而不會將所有 dependency 放去同一個資料夾內。這個做法的好處是不同的 package 即使用了相同的 module 但不同版本都不會衝突。但壞處是當一個 module 有 dependency 時，而這些 dependency 自己本身都有 dependency 時，便會令 project 的 node_modules 資料夾內有非常多層的資料夾。如果用 Windows 的話，有可能因路徑名太長不能刪除資料夾和檔案。

但原來解鈴還須繫鈴人，有人做了一個簡單的 Node 工具 [rimraf](https://github.com/isaacs/rimraf) 可以順利地移除這些資料夾。

首先，用 NPM 安裝 rimraf：

{{< highlight dos >}}
npm install -g rimraf
{{< /highlight >}}

然後，將要刪除的資料夾傳到 rimraf 即可：

{{< highlight dos >}}
rimraf C:\Users\Eric\Documents\proj
{{< /highlight >}}

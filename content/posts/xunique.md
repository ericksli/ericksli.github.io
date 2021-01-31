---
title: xUnique
tags:
  - iOS
  - Git
date: 2016-09-24T10:58:43+08:00
---

Xcode project 有個特色就是 VCS unfriendly。例子有 project.pbxproj 會有自動產生的 hash 和開啟 xib、storyboard 時不論有沒有修改過內容都好 Xcode 還是自動會將 macOS 和 Xcode 版本寫入。如果是 xib、storyboard 的版本問題就不難解決，因為比較難出現 conflict。但 project.pbxproj 就很容易 conflict。這是因為 Xcode project 和 Java 之類的 project 不同，Xcode 是可以讓 user 自訂 project 檔案排序，還有就是可以讓 user 決定那些檔案是屬於 project 的一部分，不是純綷只定義 file system 的某幾個資料夾就是 project 的一部分。然而這個特色卻容易出現 merge conflict。只要你和另一個組員在 Xcode file tree 同時將檔案排列次序改變，那就會在 merge 時出現 conflict。尤其是在前期開發時因為不時要開新檔案，會令 conflict 更容易出現。而解決 conflict 又特別麻煩，因為 project.pbxproj 的設計是讓 Xcode 讀，不是讓人去閱讀。

而解決 project.pbxproj 容易出現 VCS conflict 的方法就是用 [xUnique](https://github.com/truebit/xUnique)。它是一個 Python 寫的 script，它會將 project.pbxproj 的所有項目的 UUID 都改用 MD5 hash；而項目都會以英文字母順序排列好。這會確保 UUID 的產生方式和檔案排列次序一致，不會再有個人色彩。這樣就能減低 merge conflict 的機會。

留意 xUnique 要全部組員一齊用才會有效。使用方法可以將它設成 Git 的 pre-commit hook 又或者設定在 Xcode build 時執行。xUnique 暫時發現的缺點就是在執行完 xUnique 後如果令 project.pbxproj 有改動的話 Xcode file tree 的資料夾都會被摺疊，之後要再人手展開才可以回復先前的狀態。

註：在寫這篇文時看到解決 storyboard merge conflict 可以用 [StoryboardMerge](https://github.com/marcinolawski/StoryboardMerge)。

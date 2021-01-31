---
title: Conventional Commits 和 commitlint
tags:
  - Git
date: 2020-08-24T23:27:07+08:00
---


[Conventional Commits](https://www.conventionalcommits.org/) 是一個簡單的 Git commit message 約定，用來規定 commit message 的寫法。Git 本身就沒有規定 commit message 的內容格式，所以不同人會有不同的做法。如果 repository 只是有一個或幾個人用的話，那問題就不大。但如果在開源軟件或者公司這類多人同時參與的 repository 時，不同風格的 commit message 會令人花額外的時間來了解 repository 的變動。如果再加上一部分的 commit message 內容空洞的時候（例如只寫「fix」、「commit」、「修正錯誤」等等），情況就會失控。

<!-- more -->

一般的 Git client 因為界面空間和方便檢索，在顯示紀錄時都是顯示 commit message 的第一行。所以第一行是最重要。

```text
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

Conventional Commits 亦是這樣處理。上面是 Conventional Commits 的 commit message 結構。第一行是 header。Header 分為三部分：type、scope 和 subject。Type 就是分類，例子有 `fix`、`feat`，分別表示修正和新功能。如果按 [Angular](https://github.com/angular/angular/blob/22b96b9/CONTRIBUTING.md#-commit-message-guidelines) 的約定，還有 `build`、`ci`、`docs`、`perf`、`refactor`、`style`、`test` 可供選擇。值得一提是 [`chore` 已經被刪走](https://github.com/angular/angular/commit/dff6ee32725197bdb81f3f63c5bd9805f2ed22bb)，原因是因為太多人濫用。緊接 type 的就是 scope，它是用來提供除 type 之外額外的脈絡資訊，例如是用來注明改動和那個組件或功能相關。而 header 最重要的部分 subject 就是為這個 commit 提供一個簡短的總結。

因為 Git client 顯示 commit message 第一行的空間有限，如果 subject 空間有限而未能完全表達的話（例如是講解變動的原因），可以寫在 body 部分。Body 可以多於一段，用空行分隔。

而 footer 就是用來放 issue tracker 的 issue ID、pull request 或 merge request ID、審核者、重大變更之類，一行一個詮釋。

除了規範格式方便其他人閱讀 Git log 以外，Conventional Commits 的另一個主要作用是用來控制版本號碼的變更。具體來說就是跟 [Semantic Versioning (SemVer)](https://semver.org/) 連動。如果 Git log 有 `fix` 的 commit 就對應 SemVer 的 `PATCH` release；`feat` 就對應 `MINOR` release；`BREAKING CHANGE` 就對應 `MAJOR` release。

另外，可以配合對應 Conventional Commits 的工具自動生成 CHANGELOG，毋須再人手整理，減少遺漏項目的機會。

Conventional Commits 還有其他的規範，我不花時間逐一說明。可以到他們的[網站](https://www.conventionalcommits.org/)閱讀，網頁已被翻譯成不同的語言。

-----

有了 Conventional Commits 這個規範都未能完全解決 commit message 混亂的問題，因為總有人不按規範寫 commit message。所以就有 Conventional Commits 的 linter，例好 [commitsar](https://commitsar.tech/) 和 [commitlint](https://commitlint.js.org/)。如果用 commitlint 的話，它有一個 [@commitlint/config-conventional](https://github.com/conventional-changelog/commitlint/tree/master/%40commitlint/config-conventional) 的設定讓你做 lint rule 基底。它是按照 Conventional Commits 規範來設定，但因為規範要適用於不同的 project，所以不能寫得太嚴格。如果想要更細緻的規範，commitlint 提供了額外的 rule 可供選用。

為方便大家了解 @commitlint/config-conventional 的 rule，下面我準備了一張 cheat sheet（[另有 PDF 檔案](commitlint-config-conventional-cheatsheet.pdf)）。希望這張 cheat sheet 可以令你更容易向你的團隊推廣 Conventional Commits。

{{< figure src="commitlint-config-conventional-cheatsheet.png" title="Cheat sheet" >}}

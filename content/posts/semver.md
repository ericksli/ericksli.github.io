---
title: SemVer
tags:
- Kotlin
date: 2017-06-16T00:59:11+08:00
---


剛剛為了方便做 force update app 功能的版本號碼比對就寫了一個 [Semantic Versioning (SemVer)](http://semver.org/) 的 Kotlin data class。這個 class 有 implement `Comparable`，是參照 SemVer 規範要比對 major、minor、patch 和 pre-release version，但 `equals` 就會再比對 build metadata（即是 Kotlin data class 的預設做法）。

<!-- more -->

{{< gist ericksli 7f2e7457234b6edc144fc8b69c825efd >}}

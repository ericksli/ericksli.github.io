---
title: Parcelable & Intent extra
tags:
  - Android
  - Kotlin
date: 2017-09-16T12:03:49+08:00
---

Android 如果想將自己寫的 data type 的 object 傳到其他 `Activity`、`Fragment` 之類的地方的話，就要用 [`Parcelable`](https://developer.android.com/reference/android/os/Parcelable.html) 來做 serialization/deserialization。`Parcelable` 有點像 Java 本身的 `Serializable`，不過 `Parcelable` 是 Android SDK 內專為 Android 而特設的，所以會快過 `Serializable`。

最近寫 Android app 時無意中發現 `Intent` 的 `getParcelableExtra` return 出來的 object 是會重用的。如果我有個 object 用 Parcelable intent extra 在 Activity A 傳去 Activity B，而在 Activity B 用 `getParcelableExtra` 取回這個 object 然後再改一下 object 的 field，之後返回 Activity A 再傳相同的 object 去 Activity B，用 `getParcelableExtra` 取回這個 object 是會取得先前在 Activity B 改動過的 object，而不是 Activity A 那個原本的 object。

所以如果打算會改動 `getParcelableExtra` 傳回的 object 的話，最好都是複製一個來改，不要直接改傳回的 object。如果是 [Kotlin data object](https://kotlinlang.org/docs/reference/data-classes.html) 的話，可以用 [`copy`](https://kotlinlang.org/docs/reference/data-classes.html#copying) 這個 method。

另外，[Kotlin 1.1.4](https://blog.jetbrains.com/kotlin/2017/08/kotlin-1-1-4-is-out/) 的 Android Extensions plugin 新增了 [`@Parcelize` annotation](https://github.com/Kotlin/KEEP/blob/master/proposals/extensions/android-parcelable.md) 來自動生成 `Parcelable` 相關的 code。但它是實驗功能，還未有正式說明文檔。如果不想用實驗功能的話，可以用 [PaperParcel](https://grandstaish.github.io/paperparcel/) 之類的 library。

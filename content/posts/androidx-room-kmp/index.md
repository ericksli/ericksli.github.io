---
title: "AndroidX Room KMP"
date: 2025-05-17T14:00:00+08:00
tags:
  - Android
  - Kotlin
---

近期 Google 開始將 AndroidX library 支援 [Kotlin Multiplatform (KMP)](https://developer.android.com/kotlin/multiplatform)。而 [Room](https://developer.android.com/training/data-storage/room/room-kmp-migration) 是其中一個支援 KMP 的 library。要將 Room 變成 multiplatform library，他們將原先直接調用 Android 系統的 SQLite 變成每個 app 自帶 SQLite 的 native library（即是 bundled SQLite），這樣就可以統一每個平台所用的 SQLite 版本，確保效果一致。

要改用 bundled SQLite，首先要加入 `androidx.sqlite:sqlite-bundled` 這個 artifact。然後在 `Room.databaseBuilder` 指定使用 `BundledSQLiteDriver`：

```kotlin
const val APP_DATABASE_NAME = "metroride.db"

Room.databaseBuilder<AppDatabase>(
    context = context,
    name = context.getDatabasePath(APP_DATABASE_NAME).absolutePath,
)
    .fallbackToDestructiveMigration(dropAllTables = true)
    .setDriver(BundledSQLiteDriver())
    .setQueryCoroutineContext(ioDispatcher)
    .build()
```

`Room.databaseBuilder` 的寫法和過往的比較有一些不同。例如可以指定 coroutine dispatcher。另一樣不同的是如果用了 bundled SQLite 的話就不能在 `Room.databaseBuilder` 用 `setQueryCallback`。

另一個沒那麼明顯的位置是關於 transaction 的部分。以往的寫法是 `withTransaction` 但現在[改為 `useWriterConnection`](https://developer.android.com/training/data-storage/room/room-kmp-migration#migrate_from_support_sqlite_to_sqlite_driver)。我按照指示改用 `useWriterConnection` 但發現 app crash 了。後來才發現本來的 code 在 transaction lambda 內寫了 `runBlocking` 導致 app crash。這個部分改一改就沒問題了。

## SQLite 版本

我用了目前最新版的 bundled SQLite (`androidx.sqlite:sqlite-bundled:2.5.1`)，它其實是用了 SQLite 3.46.0；如果改用 Android 內置的 SQLite 的話在 Android 15 (API Level 35) 實機的 SQLite 版本為 3.44.3。不過看 SQLite [release history](https://www.sqlite.org/changes.html) 話 3.46.0 都不算很新，但都比 Android 內置的新。如果想知道各 Android 版本所提供的 SQLite 版本可以到 [`android.database.sqlite` 的 JavaDoc](https://developer.android.com/reference/android/database/sqlite/package-summary) 查閱對照表（[實機的 SQLite 版本會比對照表的新](https://stackoverflow.com/a/4377116)）。

如果想知道 SQLite 版本的話可以用 `SELECT sqlite_version()` 這句 query：

```kotlin
withContext(ioDispatcher) {
    val path = context.getDatabasePath(APP_DATABASE_NAME).absolutePath
    BundledSQLiteDriver().open(path).use { conn ->
        conn.prepare("SELECT sqlite_version()").use { stmt ->
            while (stmt.step()) {
                Timber.d("sqlite version ${stmt.getText(0)}")
            }
        }
    }
}
```

## 總結

如果你的 project 有用到 AndroidX Room 而又打算 multiplatform 的話，非 Android target 是用 bundled SQLite。至於 Android app 用不用 bundled SQLite 就視乎需要。如果想確保效果一致和要用到新版 SQLite 的功能就用 bundled SQLite。留意如果用了 bundled SQLite 的話 Android Studio 的 App Inspection 內的 Database Inspector 不能打開 database，它只會顯示 closed。或許可以在 debug build 加個切換功能，這樣就可以在開發 SQLite 相關的功能時用到 Database Inspector 和 `setQueryCallback`。

{{< figure src="databases.webp" title="Database Inspector 無法檢示 database 內容" >}}

改用了 bundled SQLite 後相信可以改用自己的電腦跑跟 SQLite 相關的 test（例如 database migration），因為 SQLite 版本已經鎖定，不會有測試環境跟實際環境不一致的問題。另一樣可以用的是在新版 SQLite 才有的功能（例如 3.45.0 開始才支援的 [JSONB](https://www.sqlite.org/json1.html)）。可惜 Room 到目前為止還是[不支援 FTS5](https://issuetracker.google.com/issues/146824830)，最高只到 FTS4，除非是自己手寫 SQL。

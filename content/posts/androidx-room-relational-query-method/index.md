---
title: "AndroidX Room Relational Query Method"
tags:
  - Android
date: 2022-01-29T00:15:00+08:00
---

最近為 [MetroRide](https://play.google.com/store/apps/details?id=net.swiftzer.metroride) 做新功能，剛好有個地方可以用到 [Room](https://developer.android.com/training/data-storage/room) 2.4 的新功能：Relational Query Method。這個功能可以把平常 table 之間的 relationship 用 `Map` 一次過 return 出來，不用像以前般要特製一個專門的 data class 來做 DAO query method 的 return type（正式名稱叫做 [intermediate data class](https://developer.android.com/training/data-storage/room/relationships#data-class)）。

<!-- more -->

{{< youtube i5coKoVy1g4 >}}

## Relational Query Method

看過他們的 YouTube 影片，這個功能的用法很簡單。就是在平常的 DAO method 的 return type 改成 `Map` 就可以了。

```kotlin
@Query(
    """
    SELECT rail_stations.*, rail_lines.* FROM rail_stations
    INNER JOIN rail_line_station ON rail_stations.id = rail_line_station.rail_station_id
    INNER JOIN rail_lines ON rail_lines.id = rail_line_station.rail_line_id
    WHERE rail_stations.id = :stationId 
        OR rail_stations.canonical_station_id = :stationId 
        OR rail_stations.id = (SELECT canonical_station_id FROM rail_stations WHERE id = :stationId)
    ORDER BY sequence ASC
"""
)
fun getLinesAndStationsByStationId(stationId: Long): Flow<Map<RailStation, List<RailLine>>>
```

我的 SQLite database 有 `rail_stations` 和 `rail_lines` 兩個 table，對應車站和路綫。由於車站和路綫是 many-to-many relationship，所以有了 `rail_line_station` table 連接着。

之後試試效果，當 `stationId` 調景嶺站看看它會不會 return 調景嶺站有觀塘綫和將軍澳綫，大概效果是這樣：

```text
RailStation(調景嶺)
├ RailLine(觀塘綫)
└ RailLine(將軍澳綫)
```

那個 `Map` 的 key 的確是 `RailStation`，value 又真是 `List<RailLine>`，但仔細看它們的值就發現是亂搭一通，雖然 type 是 `RailStation` 但它的內容卻是 `RailLine`。

{{< figure src="dao.png" title="DAO method return 出來的 Map" >}}

不過如果直接把 DAO method 的 SQL statement 直接放在 SQLite database 內執行又沒有異樣。

{{< figure src="query-result.png" title="在 SQLite 執行這句 query 的結果" >}}

好奇找找 Room 做了甚麼，節錄由 Room annotation processor 生成出來的 `getLinesAndStationsByStationId`：

```java
final Cursor _cursor = DBUtil.query(__db, _statement, false, null);
try {
    final int _cursorIndexOfId = CursorUtil.getColumnIndexOrThrow(_cursor, "id");
    final int _cursorIndexOfTitleZh = CursorUtil.getColumnIndexOrThrow(_cursor, "title_zh");
    final int _cursorIndexOfTitleEn = CursorUtil.getColumnIndexOrThrow(_cursor, "title_en");
    final int _cursorIndexOfInitial = CursorUtil.getColumnIndexOrThrow(_cursor, "initial");
    final int _cursorIndexOfLrlFareZone = CursorUtil.getColumnIndexOrThrow(_cursor, "lrl_fare_zone");
    final int _cursorIndexOfLrlStopCode = CursorUtil.getColumnIndexOrThrow(_cursor, "lrl_stop_code");
    final int _cursorIndexOfSystem = CursorUtil.getColumnIndexOrThrow(_cursor, "system");
    final int _cursorIndexOfCanonicalStationId = CursorUtil.getColumnIndexOrThrow(_cursor, "canonical_station_id");
    final int _cursorIndexOfLatitude = CursorUtil.getColumnIndexOrThrow(_cursor, "latitude");
    final int _cursorIndexOfLongitude = CursorUtil.getColumnIndexOrThrow(_cursor, "longitude");
    final int _cursorIndexOfGeohash = CursorUtil.getColumnIndexOrThrow(_cursor, "geohash");
    final int _cursorIndexOfPrimaryColor = CursorUtil.getColumnIndexOrThrow(_cursor, "primary_color");
    final int _cursorIndexOfKeywords = CursorUtil.getColumnIndexOrThrow(_cursor, "keywords");
    final int _cursorIndexOfId_1 = CursorUtil.getColumnIndexOrThrow(_cursor, "id");
    final int _cursorIndexOfTitleZh_1 = CursorUtil.getColumnIndexOrThrow(_cursor, "title_zh");
    final int _cursorIndexOfTitleEn_1 = CursorUtil.getColumnIndexOrThrow(_cursor, "title_en");
    final int _cursorIndexOfInitial_1 = CursorUtil.getColumnIndexOrThrow(_cursor, "initial");
    final int _cursorIndexOfSystem_1 = CursorUtil.getColumnIndexOrThrow(_cursor, "system");
    final int _cursorIndexOfId_2 = CursorUtil.getColumnIndexOrThrow(_cursor, "id");
    // 略……
    final Map<RailStation, List<RailLine>> _result = new LinkedHashMap<RailStation, List<RailLine>>();
    while (_cursor.moveToNext()) {
        // 略……
    }
    return _result;
} finally {
    _cursor.close();
}
```

出現了好幾次 `id`、`title_zh`……那就開始明白為甚麼那個 `Map` 會亂搭，原因是因為 `rail_stations` 和 `rail_lines` table 各自都有 `id`、`title_zh` 之類的 column。在 SQL query 結果出現了好幾個 `id`、`title_zh` column，然後 Room 生成的 code 就用錯了 column 來建構 `RailStation`、`RailLine` entity object。如果不做 join table 的話就不會有撞名的問題。要解決的話可以在 query 時幫 column 名改名（即是 `SELECT title_zh AS rail_line_title_zh` 之類），但 DAO 的 return type 就要特別為這句 query 再造一套，因為 `@ColumnInfo` 的名字跟原先不同。另一個方法是索性把 column 名改掉，但就要同時改動 DAO、index，又要處理 migration 問題。

## Auto Migration

同樣在介紹 Room 2.4 的影片有提及到的新功能，Room 現在能按照 database schema 的 JSON 檔案來生成 migration 的 SQL。但如果是 column 易名的話是要加上 `@RenameColumn` annotation 提示才能生成。

```kotlin
@RenameColumn(
    tableName = "rail_lines",
    fromColumnName = "id",
    toColumnName = "rail_line_id",
)
@RenameColumn(
    tableName = "rail_lines",
    fromColumnName = "title_zh",
    toColumnName = "rail_line_title_zh",
)
@RenameColumn(
    tableName = "rail_lines",
    fromColumnName = "title_en",
    toColumnName = "rail_line_title_en",
)
class RenameLineColumn : AutoMigrationSpec
```

然後在 `RoomDatabase` 的 `@Database` 加上 `autoMigrations`。

```kotlin
@Database(
    entities = [
        RailLine::class,
    ],
    version = 2,
    exportSchema = true,
    autoMigrations = [
        AutoMigration(from = 1, to = 2, spec = RenameLineColumn::class),
    ],
)
@TypeConverters(TimestampConverter::class)
abstract class AppDatabase : RoomDatabase() {
    // 略……
}
```

但到了開 app 時會 crash：

```text
FATAL EXCEPTION: main
Process: net.swiftzer.metroride.dev, PID: 22364
android.database.sqlite.SQLiteException: foreign key mismatch - "rail_line_station" referencing "rail_stations" (code 1 SQLITE_ERROR): , while compiling: PRAGMA foreign_key_check(`rail_line_station`)
    at android.database.sqlite.SQLiteConnection.nativePrepareStatement(Native Method)
    at android.database.sqlite.SQLiteConnection.acquirePreparedStatement(SQLiteConnection.java:1047)
    at android.database.sqlite.SQLiteConnection.prepare(SQLiteConnection.java:654)
    at android.database.sqlite.SQLiteSession.prepare(SQLiteSession.java:590)
    at android.database.sqlite.SQLiteProgram.<init>(SQLiteProgram.java:62)
    at android.database.sqlite.SQLiteQuery.<init>(SQLiteQuery.java:37)
    at android.database.sqlite.SQLiteDirectCursorDriver.query(SQLiteDirectCursorDriver.java:46)
    at android.database.sqlite.SQLiteDatabase.rawQueryWithFactory(SQLiteDatabase.java:1546)
    at android.database.sqlite.SQLiteDatabase.rawQueryWithFactory(SQLiteDatabase.java:1521)
    at androidx.sqlite.db.framework.FrameworkSQLiteDatabase.query(FrameworkSQLiteDatabase.java:183)
    at androidx.sqlite.db.framework.FrameworkSQLiteDatabase.query(FrameworkSQLiteDatabase.java:172)
    at androidx.room.util.DBUtil.foreignKeyCheck(DBUtil.java:136)
    at net.swiftzer.metroride.local.datasource.db.AppDatabase_AutoMigration_1_2_Impl.migrate(AppDatabase_AutoMigration_1_2_Impl.java:28)
```

再查查那段生成出來的 migration code，column 改名的 migration 做法是：

1. 建立一個新 table，table 名跟之前幾乎一樣，但前面加了個 `_`，內裏是用新的 column 名
2. 把舊 table 的內容倒進新 table
3. 把舊 table 刪除
4. 把新 table 名刪走 `_`
5. 如果有改動到 foreign key，就會用 `DBUtil.foreignKeyCheck` 檢查 foreign key constraint

```java
class AppDatabase_AutoMigration_1_2_Impl extends Migration {
    private final AutoMigrationSpec callback = new RenameLineStation();

    public AppDatabase_AutoMigration_1_2_Impl() {
        super(1, 2);
    }

    @Override
    public void migrate(@NonNull SupportSQLiteDatabase database) {
        database.execSQL("CREATE TABLE IF NOT EXISTS `_new_rail_line_station` (`rail_line_id` INTEGER NOT NULL, `rail_station_id` INTEGER NOT NULL, `rail_station_sequence` INTEGER NOT NULL, PRIMARY KEY(`rail_line_id`, `rail_station_id`), FOREIGN KEY(`rail_line_id`) REFERENCES `rail_lines`(`rail_line_id`) ON UPDATE CASCADE ON DELETE CASCADE , FOREIGN KEY(`rail_station_id`) REFERENCES `rail_stations`(`rail_station_id`) ON UPDATE CASCADE ON DELETE CASCADE )");
        database.execSQL("INSERT INTO `_new_rail_line_station` (`rail_line_id`,`rail_station_sequence`,`rail_station_id`) SELECT `rail_line_id`,`sequence`,`rail_station_id` FROM `rail_line_station`");
        database.execSQL("DROP TABLE `rail_line_station`");
        database.execSQL("ALTER TABLE `_new_rail_line_station` RENAME TO `rail_line_station`");
        database.execSQL("CREATE INDEX IF NOT EXISTS `index_rail_line_station_rail_station_id` ON `rail_line_station` (`rail_station_id`)");
        database.execSQL("CREATE INDEX IF NOT EXISTS `index_rail_line_station_rail_station_sequence` ON `rail_line_station` (`rail_station_sequence`)");
        DBUtil.foreignKeyCheck(database, "rail_line_station");

        // 其他 table 的 migration（略）

        callback.onPostMigrate(database);
    }
}
```

現在它在 `DBUtil.foreignKeyCheck(database, "rail_line_station")` 時 crash，內裏實際上是執行了 SQLite 的 ``PRAGMA foreign_key_check(`rail_line_station`)``。檢查 foreign key constraint 本來就沒有問題，問題在於變更 table 的順序，上面那個 exception 是因為 `rail_line_station.rail_line_id` 和 `rail_lines.rail_line_id` 有關連，但當時 `rail_lines` 只有 `id` column 而尚未易名導致 foreign key constraint 檢查報錯。

## Kotlin Symbol Processing

除了上面提及過的功能之外，Room 2.4 支援 [Kotlin Symbol Processing (KSP)](https://github.com/google/ksp)。但說明文檔沒有完整地展示如何設定。其實大體上跟之前用 KAPT 和 Java annotation processor 差不多，只是寫法換了小許。

```kotlin
plugins {
    // KSP plugin，留意版本前部分是 Kotlin 版本
    // 因為 KSP 本身是 Kotlin compiler plugin，所以 KSP 版本會跟 Kotlin 版本掛勾
    id("com.google.devtools.ksp").version("1.6.0-1.0.2")
    // 其餘的 plugin 略過
}

dependencies {
    // Room KSP
    ksp("androidx.room:room-compiler:2.4.1")
    // 其餘的 dependency 略過
}

ksp {
    // 向 KSP plugin 傳入參數
    // 如果要用 Room 的 migration 功能需要設定輸出 JSON 格式的 database schema 描述
    arg("room.schemaLocation", "$projectDir/schemas")
}
```

## 總結

準備 database schema 時要留意將來 join table column 撞名問題。以前用過其他 backend 的 ORM library 它們都會為你處理好這些問題。其實這個問題本來是可以在 SQL statement 為 column 名補上 alias 解決，不過用 Room 都是為了方便，如果效能不是差太遠的話相信大家都不會特別為每句 query 都做一個專用的 `@Entity` data class。

另一樣要留意的地方是 auto migration 有時候都不能做到你心目中的效果。在這次的清況如果真的要做 migration 的話可能要分幾次進行，確保順序是合乎自己的期望。另一個做法是把 annotation processor 生成出來的 code 抄去手動 migration class 內。

至於轉用 KSP 有沒有變快其實比較難感受到，因為 module 還有 Dagger Hilt 要用 KAPT，所以未能完全感受到 KSP 的速度。

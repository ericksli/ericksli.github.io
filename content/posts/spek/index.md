---
title: Spek
tags:
  - Spek
  - Kotlin
date: 2017-04-22T23:25:52+08:00
---


之前一直都有留意 [Kotlin](https://kotlinlang.org/) 這個程式語言在 Android app 開發的應用。最近試用 [Spek](http://spekframework.org/) 來做 Android project 的 local test。Spek 是一個用 Kotlin 寫的 testing framework，用法和 Ruby 的 [RSpec](http://rspec.info/) 差不多。對比 Android project 預設用的 JUnit 4，Spek 的寫法會比較清楚。因為 JUnit 4 只靠 class 和method 來為 test 分類，不能 nested（JUnit 5 才支援）。Spek 就用 nested 的方式來把 test 分類，還有就是用 string 來定義 test 名，比起 JUnit 4 用 method 名較易閱讀。

Spek 有提供 [IntelliJ IDEA/Android Studio plugin](https://plugins.jetbrains.com/plugin/8564-spek)，而且還有 JUnit platform engine。所以在 Android project 上面使用都沒有太大問題。

<!--more-->

## 安裝方法

其實 Spek 的網站有介紹過安裝方法，不過在 Android project 上使用要有些改變。

首先你的 Android project 要加入 Kotlin。這個部分 IDE 可以自動為你完成。之後就是加入 Spek。在 app 的 *build.gradle* 的 `dependencies` 加入以下部分：

{{< highlight groovy >}}
dependencies {
    testCompile "org.jetbrains.kotlin:kotlin-reflect:$kotlin_version"
    testCompile("org.jetbrains.spek:spek-api:$spek_version") {
        exclude group: 'org.jetbrains.kotlin'
    }
    testCompile "org.jetbrains.spek:spek-subject-extension:$spek_version"
    testCompile("org.jetbrains.spek:spek-junit-platform-engine:$spek_version") {
        exclude group: 'org.junit.platform'
        exclude group: 'org.jetbrains.kotlin'
    }
    testCompile 'org.junit.platform:junit-platform-runner:1.0.0-M3'
    testCompile "org.jetbrains.kotlin:kotlin-test-junit:$kotlin_version"
}
{{< /highlight >}}

留意 `spek_version` 是一個自訂變數，可以參考 `kotlin_version` 變數的定義方法。Spek 的最新版本請留意官方網站。

之後你的 test class 要加上 `@RunWith(JUnitPlatform::class)`：

{{< highlight kotlin >}}
import net.swiftzer.metroride.checkfare.Station
import org.jetbrains.spek.api.Spek
import org.jetbrains.spek.api.dsl.describe
import org.jetbrains.spek.api.dsl.it
import org.junit.platform.runner.JUnitPlatform
import org.junit.runner.RunWith
import java.math.BigDecimal
import kotlin.test.*


@RunWith(JUnitPlatform::class)
object FareTripLegSpec : Spek({
    describe("a fare trip leg node") {
        it("should have copy all other properties when from is given") {
            val originalNode = FareTripLeg(Station.CEN, Station.KOT, BigDecimal("10.0"))
            val newNode = originalNode.copy(from = Station.SYP)
            assertEquals(Station.SYP, newNode.from)
            assertEquals(Station.KOT, newNode.to)
            assertEquals(BigDecimal("10.0"), newNode.fare)
            assertNull(newNode.fareHandler)
        }
    }
})
{{< /highlight >}}

Spek 官方網站的例子有時用 class 有時用 object class，其實都是一樣的。Assertion framework 可以用 `org.jetbrains.kotlin:kotlin-test-junit`，即使你是用 Kotlin 寫 JUnit unit test 都可以用。

## 小問題

Spek 在一般使用（在 IDE 和 Gradle）都沒有太大問題，但仍有一些小問題。

- Spek IntelliJ IDEA plugin 功能未有 JUnit 般完善。例如 double click test case 名稱本應可以自動跳到 test 的 source code 對應位置，但 Spek 的 test 就不能。
- 在 Structure 面版看不到 Spek test 的結構，只可以看到 Kotlin class 的基本 method（即是 `equals`、`hashCode`、`toString` 那些），如果 test 寫得長會難跳到想去的位置。
- Gradle report 顯示會有些怪，可能是因為用 JUnit 4。（下面會再介紹）
- 在 IntelliJ IDEA/Android Studio 執行 Spek test 時需要特別設定 Run/Debug Configuration，否則可能會報錯或者 run 過一次 test 之後改 Spek test source code 再 run 時會用了舊 code 來測試。（下面會再介紹）

### Gradle HTML Report

如果你會查閱 Gradle 生成的 HTML test report 的話，會發現自己寫的 Spek test class 是空白的：

{{< figure src="empty-report.png" title="空白的 Spek test class" >}}

但其實 test class 的內容會被放到 default-package 內：

{{< figure src="actual-test-result.png" title="實際的結果是放在 default-package 內" >}}

不過在 [BuddyBuild](https://buddybuild.com) 看又會是正常，可能是他們特別處理過。

{{< figure src="buddybuild.png" title="過在 BuddyBuild 看又會是正常" >}}

### IntelliJ IDEA/Android Studio 的 Run/Debug Configuration

如果你的 Run/Debug Configuration 是 Spek 的話，緊記要在 Before launch 加入 `assembleDebugUnitTest`。（資訊由 Ranie Jade Ramiso 提供）

{{< figure src="run-debug-config.png" title="Run/Debug Configuration" >}}

如果是 Android JUnit 的話就不用特別設定。

## 小結

最後，要留意 Spek 並不是 Kotlin 指定的 testing framework（雖然 GitHub repo 是放在 JetBrains 名下），你可以使用其他的 framework（例如 JUnit）。還有 Kotlin 本身是支援 method 名中間有空白字符（Android Studio 會有 linter 警告，但是仍可以 compile 到的）。所以你仍可以使用 JUnit 來做 unit testing。下面是一個例子：

{{< highlight kotlin >}}
@file:Suppress("IllegalIdentifier")

class FareTripTest {
    @Test
    @Throws(Exception::class)
    fun `test get from one child node`() {
        val fareTrip = FareTrip()
        val fareTripLeg = FareTripLeg(Station.SHM, Station.KOT, BigDecimal("12.0"))
        fareTrip.addChildNode(fareTripLeg)
        assertEquals(Station.SHM.toLong(), fareTrip.from.toLong())
    }
}
{{< /highlight >}}

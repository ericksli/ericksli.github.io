---
title: Kotlin for Android
tags:
  - Android
  - Java
  - Kotlin
date: 2017-06-12T23:43:16+08:00
---


在四月開始轉用 [Kotlin](https://kotlinlang.org/) 來寫自己的 Android app。其實上年八月左右已經留意到 Kotlin 這個 JVM 語言能在 Android app 開發時使用，不過那時因為沒有太多時間所以只是看了少許官方教學和一些外國網誌就作罷，沒有真正拿來寫 Android app。到了最近看到愈來愈多人開始轉用 Kotlin 所以才真正開始轉用。到了現在 Kotlin 更成為 Android first-class support language。

初初轉用時都有些地方不明白，需要經常查閱文檔和 Google 例子。但其實 Kotlin 都不算太難學，syntax 上和 Java 有不同但差異不算太大，再加上一些當代語言常見的特性。所以如果本身有學過其他語言的話會很快上手。Kotlin 誕生的原因是 JetBrains 用 Java 開發 IDE 時發現到 Java 的不足而令他們決心做一個新的語言，所以骨子裏有着 Java 的影子，而 Kotlin 本身都是 JVM language（即是 Kotlin 原碼會變成 JVM bytecode 然後用 JVM 來執行）。現在 Kotlin 除了 compile 成 JVM bytecode 之外，還可以轉換成 JavaScript 和 native（即是直接在作業系統上執行，不需要 JVM/Node）。

<!-- more -->

## `val` & Nullable

Kotlin 的 nullable 和 `var`/`val` 特性令 declare variable 時要考慮到它是不是 mutable 和能否 null（其實 `val` 只是代表只可以 assign 一次，並不完全代表它是 immutable）。這些特性在開發時很有用，因為在使用這些 variable 時就能肯定它會不會 null 和會不會中途被改變 reference。而它會在 compile 時會檢查你的 code 是不是合符 variable 的 nullable 和 `var`/`val` 定義，那就避免在執行時才出現 `NullPointerException`。如果它是 nullable 的話，Kotlin 有 `?.` (safe navigation operator)、`?:` (null coalescing operator/Elvis operator) 和 `if` 來處理，比起 Java 要加大量的 `if` 和 Java 8 的 `Optional` 更加簡潔。

## Lambda

Lambda 可能是在 Android 未正式公布 Java 8 support 前轉用 Kotlin 的主要原因。因為 Android 有不少 callback 是 SAM-type (single abstract method)，加上近期流行的 RxJava 都要用到大量 SAM-type，如果每次都要寫完整的 anonymous class 就會變得很長（雖然 IDE 可以將 code 摺起讓它看起來像 lambda）。Java 8 的 lambda 正好就能簡化這種寫法。在未有 Java 8 support 時，就要用 [Retrolambda](https://github.com/orfjackal/retrolambda) 來令 Java 7 可以用 Java 8 lambda syntax。不過現在 Android 都開始支援在舊 Android 版本用 lambda，lambda 現在未必能吸引 Android developer 轉用 Kotlin。不過 Kotlin 和 Android Java 不同之處就是 Kotlin 不是靠 Android 版本來決能你能否使用新的語言特性，所以你不用等 Android API level 24 成為 project 最低支援版本時才能使用 `map`、`reduce` 之類的功能，只需要轉用 Kotlin 就能馬上使用。日後 Kotlin 有新功能時只需要改一下 Gradle file 就能用新功能。

## Java Interop

開發 Android app 必定要調用 Java class（Android SDK 和其他第三方 library），所以 Kotlin/Java 能否混合使用是非常重要。基本上在 Android 上用 Kotlin 都沒有發現有大問題。因為 Kotlin 本身就是可以和 Java 混合使用，你可以照常在 project 使用 Java 寫的 library。在 Kotlin 用 Java 的 class 是不需要太多特別處理，只是在 override method 時 parameter 可能會全部都當成 nullable（即是 parameter type 加了 `?`）。這是因為這個 Java class 的 method parameter 未有加上 `@Nullable` 和 `@Nonnull` annotation，所以 IDE 會假設全部 parameter 都是可以 null 的。如果你確定它不會 null 的話可以把 `?` 拿走。[現在 Square 出的 open source library 會開始加上 `@Nullable` annotation](https://medium.com/square-corner-blog/rolling-out-nullable-42dd823fbd89)，期望 Android SDK 日後都會補回 `@Nullable`，同時方便用 Java 和 Kotlin 的人。如果是在 Java 用 Kotlin 的 class/function 的話，就需要留意是否需要[補回 annotation](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.jvm/index.html#annotations) 來控制 Java 一方所見到的效果（例如 class 名、是否需要用 static、constructor 是否需要窮舉所有 parameter 組合等等）。此外，Kotlin 有 `kapt` 做 annotation processor，在 Java 用的 `apt`、`annotationProcessor` 可以用 `kapt` 代替。

## Extension

Kotlin 的 extension 是用來為現成的 class 加入新 method 和 attribute。Extension 是用來取代 Java programming 時常寫的 Utils static method。Extension 比 Utils 較佳是 extension 有 IDE 提示和用法較自然。

舉個例子：`FirebaseAnalytics` 本來是有一個 `logEvent` 的 method，不過第二個 parameter 是 `Bundle`，使用時要預先準備好 `Bundle` 比較不方便。參考了 [Anko](https://github.com/Kotlin/anko) `startActivity` 的[做法](https://github.com/Kotlin/anko/wiki/Anko-Commons-%E2%80%93-Intents)，做了一個 extension function。

{{< highlight kotlin >}}
fun FirebaseAnalytics.logEvent(name: String, vararg params: Pair<String, Any>) {
    check(params.size <= 25) { "An event can have up to 25 parameters" }

    if (params.isEmpty()) {
        logEvent(name, null)
    } else {
        val bundle = Bundle()
        params.forEach { (key, value) ->
            when (value) {
                is Int -> bundle.putInt(key, value)
                is Long -> bundle.putLong(key, value)
                is CharSequence -> bundle.putCharSequence(key, value)
                is String -> bundle.putString(key, value)
                is Float -> bundle.putFloat(key, value)
                is Double -> bundle.putDouble(key, value)
                is Char -> bundle.putChar(key, value)
                is Short -> bundle.putShort(key, value)
                is Boolean -> bundle.putBoolean(key, value)
                else -> throw UnsupportedOperationException("Unsupported type ${value.javaClass.name} in params")
            }
        }
        logEvent(name, bundle)
    }
}
{{< /highlight >}}

這個 extension 的用法是在第二個 parameter 改為傳入多個 [`Pair`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/-pair/)。`Pair` 是 Kotlin standard library 的一個 data class，用來放兩個 value。然後將 `Pair` 變成 `Bundle` 對應的 `put` method，最後將生成的 `Bundle` 傳到 `FirebaseAnalytics` 原本的 `logEvent`。

開首的 [`check`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/check.html) 也是 Kotlin standard library 的現成 function，當條件不符合時就會 throw `IllegalStateException`。

在 extension 入面，`this` 會變成 `FirebaseAnalytics`，但只可以存取到 `public` 的東西。

下面就是 extension function 的用法：

{{< highlight kotlin >}}
// Inside Activity
val firebaseAnalytics = FirebaseAnalytics.getInstance(this)
firebaseAnalytics.logEvent(Event.SELECT_CONTENT,
    Param.CONTENT_TYPE to "setting",
    Param.ITEM_ID to "about")
{{< /highlight >}}

除了 extension function 之外，還有 extension property。只需要自訂 getter/setter 就可以當成 property 使用。

{{< highlight kotlin >}}
val Context.myApp: MyApplication
    get() = applicationContext as MyApplication

val Fragment.myApp: MyApplication
    get() = activity.applicationContext as MyApplication
{{< /highlight >}}

這兩個 extension property 可以省卻每次在 `Activity` 和 `Fragment` 使用 `MyApplication` 時都要做 type casting。

## Data Class

Data class 主要用來替代平時儲 data 的 POJO。Compile 時會自動生成 getter/setter、`toString`、`hashCode`、`equals`、`componentN` function、`copy`，令行數大大減少。在 Google I/O 宣布 Kotlin 成為 Android first-class support language 時不少網站都以 data class 作例子，所以不詳細介紹了。

## DSL

Kotlin 有一個「得意」的功能，就是可以做 DSL (domain-specific language)，Kotlin 官方文件稱為 [type-safe builder](https://kotlinlang.org/docs/reference/type-safe-builders.html)。例子有 [Anko](https://github.com/Kotlin/anko)（用來取代 XML 做 Android UI layout）、[Spek](http://spekframework.org/)（用來寫 testing specification，類似 RSpec）、[kotlinx.html](https://github.com/Kotlin/kotlinx.html)（用來生成 HTML 的 DSL）等等。不過又不需要 librray 級的 project 才要做 DSL，有時 project 中需要做一些巢狀的 object（比如類似樹狀的 object），那時用 DSL 就可以用直觀的方式建立整個 tree 而不需要寫一大堆 create object 再 add to list 的 code。

做 DSL 主要是用到 Kotlin 以下的特性：

- [function literal with receiver](https://kotlinlang.org/docs/reference/lambdas.html#function-literals-with-receiver)
- [operator overloading](https://kotlinlang.org/docs/reference/operator-overloading.html)
- [infix function](https://kotlinlang.org/docs/reference/functions.html#infix-notation)
- [extension function](https://kotlinlang.org/docs/reference/extensions.html)
- [`DslMarker` annotation](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/-dsl-marker/)

官方文件對 DSL 沒有太深入的講解，反而 [Kotlin in Action](https://www.manning.com/books/kotlin-in-action) 第 11 章有詳細講解如何做 DSL，而這一章正好提供免費試讀。有需要的話可以看看。

## 不足的地方

基本上都沒發現太大的問題，只是配套上有所不足。例如 IntelliJ IDEA/Android Studio 內置的 code coverage runner 在計算 branch coverage 會不準確。

另一個問題是 data class 和 JSON/XML serialization/deserialization library 未能全面配合，其實這個問題不太關 Kotlin 事。Data class 容許定義 default value、nullable，不過有不少 JSON/XML serialization/deserialization library 都是用 Java reflection 來做轉換。有些 library 會支援 Kotlin data class（例如 [moshi-kotlin](https://github.com/square/moshi) 和 [jackson-module-kotlin](https://github.com/FasterXML/jackson-module-kotlin)），但需要用到 kotlin-reflect 這個 dependency（這個 dependency 有過萬個 method，對 Android 來說是很大）。

## 小貼士

- 可以善用 IDE 的「Show Kotlin bytecode」功能，看看生成的 Java bytecode。不懂看的話可以按一下「Decompile」，IDE 會將 bytecode 變成 Java。可以用這個功能來看看生成的 Java code。亦可以用這個功能來對比 Kotlin 和 Java 的分別。
- IDE 有「Kotlin REPL」，可以用來簡單試一下 Kotlin 的用法，不用等待 IDE build project。
- 目前 [Google Samples](https://github.com/googlesamples) 的 GitHub 開始有同時提供 Java 和 Kotlin 的 demo project，不過 Kotlin 版好像是自動轉換出來的，寫得不太好，所以目前都是看 Java 版比較好。
- Android 的 Activity、Fragment、Service 等等都不是用 constructor 來 initialize variable，但又不想將它變成 nullable 的話，可以善用 `lateinit`、`Delegates.notNull` 來押後 initialize variable。
- 如果是想用於 Android 開發而沒有學過 Java 的話，還是先學 Java 才學 Kotlin。因為 Android SDK 是 Java 而且第三方 library 都是 Java 居多。即使有提供 Kotlin 版大多只是提供 extension function 而不是完全用 Kotlin 重寫一遍。

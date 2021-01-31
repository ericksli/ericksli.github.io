---
title: Kotlin Annotation Processor
tags:
  - Kotlin
date: 2020-04-27T23:34:08+08:00
---


如果有做過 Android 開發的話應該都有用過 annotation processor（又稱 codegen），即是在 *build.gradle* 入面要用 `annotationProcessor` 或者 `kapt` 的那些 dependency。用法大概是在 code 上加上一些 `@` 開頭的 annotation，然後 build 出來就會自動幫你生成相關的 class。簡單來說 annotation processor 就是用 code 來讓 Java compiler 生成 code。通常都是用來生成一些內容重覆的 code 來代替自己人手寫。

自己做一個功能不多的 annotation processor 其實都不太難。難的地方是 debug 時不能像平常般加 breakpoint，加上我們要生成 Kotlin class 所以跟平常找到的教學會有輕微出入（因為大部分教學都是討論生成 Java class）。

<!-- more -->

這篇文章會示範做一個生成 feature flag 的 class。這個 feature flag 其實就是一個 interface，入面有不同的 method，一個 method 代表一個 feature flag，它們都是會回傳 boolean。把它做成 interface 而不直接定義一堆 boolean constant 的原因是方便寫 test case。我們可以 mock 那個 interface 就能測試那個 flag 是開和關的情境。下面是一個 interface 例子：

{{< highlight kotlin >}}
package com.example.annotation

import com.example.annotation.annotation.FeatureFlag
import com.example.annotation.annotation.FeatureFlagGroup

@FeatureFlagGroup
interface MyFeatureFlagsB {
    @FeatureFlag(key = "feature_b1", defaultValue = false)
    fun featureB1(): Boolean

    @FeatureFlag(key = "feature_b2", defaultValue = false)
    fun featureB2(): Boolean
}
{{< /highlight >}}

`@FeatureFlagGroup` 和 `@FeatureFlag` 是我們做的 annotation，我們會寫一個 annotation processor 來讀取這些被標注的 class 和 method，然後生成 implement 這個 interface 的 class。下面就是完成品：

{{< highlight kotlin >}}
package com.example.annotation.sample

import javax.annotation.Generated
import javax.inject.Inject
import kotlin.Boolean
import kotlin.String
import kotlin.collections.Map

/**
 * Concrete implementation of [MyFeatureFlagsB].
 */
@Generated(value = ["com.example.annotation.codegen.FeatureFlagCodegen"])
class MyFeatureFlagsBImpl @Inject constructor() : MyFeatureFlagsB {
  override fun featureB1(): Boolean = false

  override fun featureB2(): Boolean = false

  companion object {
    /**
     * Default value map
     */
    val defaultValues: Map<String, Boolean> = mapOf(
        "feature_b1" to false,
        "feature_b2" to false
        )
  }
}
{{< /highlight >}}

要做一個 annotation processor，我們要準備三個 module：

1. 放 annotation（`@FeatureFlagGroup` 和 `@FeatureFlag`）的 module (`:annotation`)
2. 放 annotation processor 的 module (`:codegen`)
3. 試用那個 annotation processor 的 module，這個會用到上面兩個 module (`:sample`)

{{< figure src="project-structure.png" title="這三個 module 的內容" >}}

## Annotation Module

下面就是剛才看到那兩個 annotation。`FeatureFlagGroup` 是放在 interface 的，所以 `@Target` 是 `AnnotationTarget.CLASS`；而 `FeatureFlag` 是放在那些 method，用來標註它的名稱和回傳值。所以 `@Target` 是 `AnnotationTarget.FUNCTION`。

{{< highlight kotlin >}}
package com.example.annotation.annotation

@Retention(AnnotationRetention.SOURCE)
@Target(AnnotationTarget.CLASS)
@MustBeDocumented
annotation class FeatureFlagGroup
{{< /highlight >}}

{{< highlight kotlin >}}
package com.example.annotation.annotation

@Retention(AnnotationRetention.SOURCE)
@Target(AnnotationTarget.FUNCTION)
@MustBeDocumented
annotation class FeatureFlag(val key: String, val defaultValue: Boolean)
{{< /highlight >}}

## Codegen Module

這個是我們的戲玉。要寫一個 annotation processor，首先是要 extend `javax.annotation.processing.AbstractProcessor`。入面有三個 method 需要 override：

1. `getSupportedAnnotationTypes`：把這個 processor 用到的 annotation 名稱列舉出來，即是 `FeatureFlagGroup::class.java.name` 和 `FeatureFlag::class.java.name`。
2. `getSupportedSourceVersion`：回傳支援的 Java 版本。我們簡單些回傳 `SourceVersion.latestSupported()` 就算了。
3. `process`：最重要的地方，我們會在入面讀取 annotation 然後生成 Kotlin code。

在 override `process` method 之前，我們要在這個 process class 加上 `@AutoService` 和 `@SupportedOptions` 兩個 annotation。第一個是 [AutoService Processor](https://github.com/google/auto/tree/master/service)，這個 processor 會幫你自動生成 *META-INF/services/javax.annotation.processing.Processor* 檔案。這個檔案是做 annotation processor 一定要有的東西。你可以自己手動加，但最方便都是用 `@AutoService`。

第二個 annotation `@SupportedOptions` 是用來標明我們會用 [Kotlin annotation processing (kapt)](https://kotlinlang.org/docs/reference/kapt.html)。另外開了一個 `KAPT_KOTLIN_GENERATED_OPTION_NAME` constant 是因為另一個地方都會用到。

{{< highlight kotlin >}}
private const val KAPT_KOTLIN_GENERATED_OPTION_NAME = "kapt.kotlin.generated"

@AutoService(Processor::class)
@SupportedOptions(KAPT_KOTLIN_GENERATED_OPTION_NAME)
class FeatureFlagCodegen : AbstractProcessor()
{{< /highlight >}}

之後 `process` 入面就是這樣：找出外層那個 `@FeatureFlagGroup`，然後在它的 class 入面找用了 `@FeatureFlag` 的 method，之後就交由 `generateImpl` 生成 implement feature flag interface 的 class。

{{< highlight kotlin >}}
override fun process(annotations: MutableSet<out TypeElement>, roundEnv: RoundEnvironment): Boolean {
    if (roundEnv.processingOver()) return false
    roundEnv.getElementsAnnotatedWith(FeatureFlagGroup::class.java)
        .filter { it.kind == ElementKind.INTERFACE }
        .forEach { featureFlagGroupElement ->
            val featureFlagElements = featureFlagGroupElement.enclosedElements
                .filter { it.getAnnotation(FeatureFlag::class.java) != null && it.kind == ElementKind.METHOD }
            val packageName = processingEnv.elementUtils.getPackageOf(featureFlagGroupElement).toString()
            generateImpl(packageName, featureFlagGroupElement, featureFlagElements)
        }
    return roundEnv.getElementsAnnotatedWith(FeatureFlagGroup::class.java).any { it.kind == ElementKind.INTERFACE }
}
{{< /highlight >}}

-----

`generateImpl` 比較長，所以都是分開幾段貼出來。如果要看完整的 code 可以到文末找到 Gist。

首先是一段檢查 [kapt](https://kotlinlang.org/docs/reference/kapt.html)，如果沒有就出錯誤。這亦示範了輸出錯誤的方法。`processingEnv.options["kapt.kotlin.generated"]` 放的就是我們要把 code 寫進去的目錄。放在這目錄的檔案之後會連同本身的 source code 一齊 compile。`KAPT_KOTLIN_GENERATED_OPTION_NAME` 是我們在開首所定義的 constant。在生成新的檔案前要先建立這個目錄。

{{< highlight kotlin >}}
val generatedSourcesRoot = processingEnv.options[KAPT_KOTLIN_GENERATED_OPTION_NAME].orEmpty()
if (generatedSourcesRoot.isEmpty()) {
    processingEnv.messager.printMessage(
        Diagnostic.Kind.ERROR,
        "Can't find the target directory for generated Kotlin files."
    )
    return
}

val implClassName = "${featureFlagGroupElement.simpleName}Impl"
val file = File(generatedSourcesRoot)
file.mkdir()
{{< /highlight >}}

之後我們就正式開始生成 code 了。簡單來說我們就是寫一個 program 來把一些 Kotlin source code 寫入到剛才那個目錄。雖然 Kotlin 有 `"""` 那款 [raw string](https://kotlinlang.org/docs/reference/basic-types.html#string-literals)，亦都有 string template，但還是用 [KotlinPoet](https://square.github.io/kotlinpoet/) 比較方便。如果是寫 Java 的話可以用 [JavaPoet](https://github.com/square/javapoet)。

下面我們直接看生成 class 的骨架，之後會補回未提及的 code block。

{{< highlight kotlin >}}
FileSpec.builder(packageName, implClassName)
    .addType(
        TypeSpec.classBuilder(implClassName)
            .addAnnotation(
                AnnotationSpec.builder(Generated::class.java)
                    .addMember("value = [%S]", FeatureFlagCodegen::class.java.name)
                    .build()
            )
            .addKdoc(CodeBlock.of("Concrete implementation of [%T].", className))
            .primaryConstructor(
                FunSpec.constructorBuilder()
                    .addAnnotation(ClassName("javax.inject", "Inject"))
                    .build()
            )
            .addSuperinterface(className)
            .addFunctions(funSpecs)
            .addType(
                TypeSpec.companionObjectBuilder(null)
                    .addProperty(
                        PropertySpec.builder(
                            "defaultValues",
                            Map::class.asClassName()
                                .parameterizedBy(String::class.asClassName(), Boolean::class.asClassName())
                        )
                            .initializer(defaultValuesMapCodeBlock)
                            .addKdoc("Default value map")
                            .build()
                    )
                    .build()
            )
            .build()
    )
    .build()
    .writeTo(file)
{{< /highlight >}}

首先是看到 `FileSpec.builder` 然後才看到 `TypeSpec.classBuilder`。因為 Kotlin 是可以一個 *.kt* 檔案內包含多於一個 class。

緊接 `TypeSpec.classBuilder` 是 `addAnnotation`。我們會在這個 class 加上 [`@Generated`](https://docs.oracle.com/javase/8/docs/api/javax/annotation/Generated.html)，讓人知道這個 class 是自動生成出來而不是人手寫的。不過留意的是，[加了這個 annotation 不代表 JaCoCo 能自動忽略這個 class](https://github.com/jacoco/jacoco/issues/831)，因為 `javax.annotation.Generated` 的 retention policy 是 `SOURCE`，被 JaCoCo 看到之前這個 annotation 已被抽走。如果想被 JaCoCo 忽略的話還是要另外做一個 retention policy 是 `RUNTIME` 或 `CLASS` 的 `@Generated` annotation。

接着就替這個 class 加上 [KDoc](https://kotlinlang.org/docs/reference/kotlin-doc.html)（即是 Kotlin 寫 comment 的格式，類似 JavaDoc）。我們會用到 KotlinPoet 的 `CodeBlock`。`CodeBlock` 是一個 KotlinPoet 經常用到的東西，這是用來寫一句句 statement 用的。留心看的話會見到入面有個 `%T`，看起來好像平時 `String.format` 用到的 formatter。但其實這是 [KotlinPoet 專用的 formatter](https://square.github.io/kotlinpoet/#t-for-types)。`%T` 就是代表 Type。KotlinPoet 能自動幫你生成對應的 `import` statement。下面是 `className` 的內容（就是透過 annotation 來取得被標註的 interface 名稱）：

{{< highlight kotlin >}}
val className = ClassName(
    (featureFlagGroupElement.enclosingElement as PackageElement).qualifiedName.toString(),
    featureFlagGroupElement.simpleName.toString()
)
{{< /highlight >}}

之後就開始寫 default constructor。這個 constructor 沒有任何參數，但要加上 `@Inject` annotation。由於這個 codegen module 沒有加到 javax.inject，所以我們直接寫它的全名。如果 codegen module 有加到 javax.inject 的話，那就可以寫成 `addAnnotation(Inject::class.java)`。但其實 codegen module 是不用加生成出來的 code 會用到的 dependency，只要知道它們全名就可以了。最重要的是用 codegen 那個 module 要加入這些 dependency。

加了 default constructor 之後就是幫這個生成的 class 標明 superclass。就是用 `addSuperinterface(className)` 這一句。

現在來到另一個主要的部分：`addFunctions(funSpecs)`。這句就是為這個 class 加入那些回傳 boolean (`@FeatureFlag`) 的 method。我們會用 `FunSpec.overriding` 來造這些 method。每個 method 入面都只有一句 return statement。Boolean 的值我們可以從 `@FeatureFlag` 的 `defaultValue` 找到。由於 Kotlin [string template](https://kotlinlang.org/docs/reference/basic-types.html#string-templates) 可以直接輸出 boolean 的 `true`/`false` 字串，所以不需要用到那些特別的 formatter。

{{< highlight kotlin >}}
val funSpecs = featureFlagElements.map {
    val executableElement = it as ExecutableElement
    val featureFlagAnnotation = it.getAnnotation(FeatureFlag::class.java)
    FunSpec.overriding(executableElement)
        .addStatement("return ${featureFlagAnnotation.defaultValue}")
        .build()
}
{{< /highlight >}}

基本上來到這裏就大致完成了我們的 annotation processor。不過我們做一些特別的東西：加上 companion object 並加上一個名為 `defaultValues` 的 `Map<String, Boolean>`，用來放那個 feature flag 的名稱和它們的值。

{{< highlight kotlin >}}
TypeSpec.companionObjectBuilder(null)
    .addProperty(
        PropertySpec.builder(
            "defaultValues",
            Map::class.asClassName()
                .parameterizedBy(String::class.asClassName(), Boolean::class.asClassName())
        )
            .initializer(defaultValuesMapCodeBlock)
            .addKdoc("Default value map")
            .build()
    )
    .build()
{{< /highlight >}}

`defaultValuesMapCodeBlock` 就是 `mapOf("key1" to true, "key2" to false)` 那一句。

`CodeBlock` 除了先前看到的 `CodeBlock.of` 外，𨗢有 builder 這種用法。[`%S` 是 string 的意思](https://square.github.io/kotlinpoet/#s-for-strings)，不用我們自己人手加 `"`，讓 KotlinPoet 做就行了，自己做容易做錯。另外，`to` 的前後都有一個點 (`·`)。這亦都是 [KotlinPoet 特別的字符](https://square.github.io/kotlinpoet/#spaces-wrap-by-default)。KotlinPoet 為了生成易讀的 code，它會自動斷行而不會一直把 code 寫成一行。但它有機會在不適當的地方斷行導致不能 compile。為避免它在 `to` 前後斷行，我們用 `·` 取代空白字元。

{{< highlight kotlin >}}
val defaultValuesMapCodeBlock =
    featureFlagElements.foldIndexed(CodeBlock.builder().add("mapOf(\n")) { index, builder, element ->
        val featureFlagAnnotation = element.getAnnotation(FeatureFlag::class.java)
        builder.add("%S·to·${featureFlagAnnotation.defaultValue}", featureFlagAnnotation.key)
        if (index < featureFlagElements.size - 1) {
            builder.add(",\n")
        } else {
            builder.add("\n)")
        }
        builder
    }.build()
{{< /highlight >}}

## Sample Module

現在可以來看看它的效果。完整的 code 可以在 [Gist](https://gist.github.com/ericksli/ea72dbb18884a4c0af6b758b54eccfc6) 找到。

{{< figure src="sample-run.png" title="測試用的 Main、interface class 和生成的 class" >}}

## 參考

* [Annotation Processing 101](http://hannesdorfmann.com/annotation-processing/annotationprocessing101)
* [Java 注解处理 (Annotation Processor)：（三）代码生成](https://blog.csdn.net/jjxojm/article/details/90349756)

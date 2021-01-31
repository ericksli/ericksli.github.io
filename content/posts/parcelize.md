---
title: Kotlin Parcelize
tags:
  - Kotlin
  - Android
date: 2018-02-11T23:28:05+08:00
---


[Kotlin Android extensions](https://kotlinlang.org/docs/tutorials/android-plugin.html) 入面有一個實驗功能：Parcelize。它是一個 annotation，只需要在 [data class](https://kotlinlang.org/docs/reference/data-classes.html) 加上 `@Parcelize` annotation 和 implement `Parcelable` interface 就能夠在 compile 時自動生成所需的 boilerplate。

{{< highlight kotlin >}}
@Parcelize
data class Product(val name: String, val price: Double) : Parcelable
{{< /highlight >}}

留意要在 build.gradle 加上：

{{< highlight gradle >}}
androidExtensions {
    experimental = true
}
{{< /highlight >}}

這個基本用法在不少網站都有介紹過，的確可以節省不少時間 copy and paste code，或者可以不需要再用 PaperParcel 之類的 library。不過如果要自訂個別 property 的 adapter 的話（例如那個 property 的 data type 不是自己的 class 又沒有 implement `Parcelable`），就可以用 `@WriteWith` 來註明 adapter：

{{< highlight kotlin >}}
@Parcelize
data class Product(
    val name: String,
    val price: Double,
    val expiryDate: @WriteWith<ExpiryDateParceler> ExpiryDate
) : Parcelable
{{< /highlight >}}

假設我們有個 `ExpiryDate` 的 field，在 type 前面加入 `@WriteWith` annotation。`ExpiryDateParceler` 就是我們特別為 `ExpiryDate` 寫的 adapter object class。

{{< highlight kotlin >}}
object ExpiryDateParceler : Parceler<ExpiryDate> {
    override fun create(parcel: Parcel): ExpiryDate = ExpiryDate.createByFullDate(parcel.readString())

    override fun ExpiryDate.write(parcel: Parcel, flags: Int) {
        parcel.writeString(this.fullDate)
    }
}
{{< /highlight >}}

`Parceler` 有兩個 method 要實作：一個是從 `ExpiryDate` serialize 變成 `Parcel`；另一個是由 `Parcel` deserialize 變成 `ExpiryDate`。這個例子用了 string 來 serialize，你可以用其他 type 來 serialize，最重要是之後可以被還原。留意必須使用 object class。

如果 `ExpiryDate` 是 nullable 的話，在 `ExpiryDateParceler` 的 `ExpiryDate` 改成 `ExpiryDate?` 即可。

---
title: "Jetpack Compose Navigation component sub-graph"
date: 2022-07-27T21:30:00+08:00
tags:
  - Android
  - Jetpack Compose
---

這次[遷移到 Compose]({{< ref "jetpack-compose-migration-1" >}}) 時特別花了時間試用 Compose 的 Navigation component，終於弄清 nested graph 的意義。其實 Compose 的 Navigation component 底層都是跟 XML 版的 Navigation component 一樣，只是底層多了以 route 形式的處理。以往的說明文件在介紹 [nested navigation graph](https://developer.android.com/guide/navigation/navigation-nested-graphs) 時沒有太具體說明 nested graph 背後的意義，看完之後可能覺得只是用來避免單一 XML 檔過長而拆成不同 sub-graph。但其實在 deep link 時是有特別意義。

<!-- more -->

接下來會用例子示範 nested graph 在 navigation 時出現的不同效果。首先放上在 `NavHost` 各頁會用到的 `Screen` composable function，它會顯示文字和按鈕，用來顯示當前頁面的 route 和按下按鈕時會轉到另一頁。

```kotlin
@Composable
fun Screen(
    name: String,
    onClick: (() -> Unit)? = null,
    secondaryOnClick: (() -> Unit)? = null,
) {
    Column {
        Text(text = name)
        if (onClick != null) {
            Button(onClick = onClick) {
                Text(text = "next screen")
            }
        }
        if (secondaryOnClick != null) {
            Button(onClick = secondaryOnClick) {
                Text(text = "secondary button")
            }
        }
    }
}
```

接下來就是 navigation graph。

```kotlin
val navController = rememberNavController()
NavHost(
    modifier = modifier,
    navController = navController,
    startDestination = "top",
) {
    composable(route = "top") {
        Screen(
            name = "top",
            onClick = { navController.navigate("partA") },
            secondaryOnClick = { navController.navigate("b3") },
        )
    }
    navigation(startDestination = "a1", route = "partA") {
        composable(route = "a1") {
            Screen(name = "a1") { navController.navigate("a2") }
        }
        composable(route = "a2") {
            Screen(name = "a2") { navController.navigate("a3") }
        }
        composable(route = "a3") {
            Screen(
                name = "a3",
                onClick = { navController.navigate("partB") },
                secondaryOnClick = {
                    val intents = navController.createDeepLink()
                        .addDestination("b3")
                        .createTaskStackBuilder()
                        .intents
                    navController.handleDeepLink(intents.first())
                },
            )
        }
    }
    navigation(startDestination = "b1", route = "partB") {
        composable(route = "b1") { Screen(name = "b1") }
        navigation(startDestination = "b2", route = "partB2") {
            composable(route = "b2") { Screen(name = "b2") }
            composable(route = "b3") { Screen(name = "b3") }
        }
    }
}
```

先看首頁 `top`，按 `top` 的「next screen」按鈕能進入 `a1` 頁，因為 `a1` 是 `partA` sub-graph 的 start destination。其實在這個情況下可以把 `onClick` lambda 寫成 `navController.navigate("a1")` 都是一樣效果。如果一直按「next screen」按鈕就會順序進入 `a1` ➔ `a2` ➔ `a3` ➔ `b1` 頁。按 back 掣就能由 `b2` 沿路回到 `a1`，最後結束 `Activity`（假設 `NavHost` 是放在 `Activity`）。

這個看起來沒甚麼特別。之後換成在 `top` 按「secondary button」鈕能的話就會直接進入 `b3` 頁，按 back 掣一次就會回到 `top`。

接着在 `top` 頁一直按「next screen」按鈕直到看到 `a3` 頁時按「secondary button」按鈕，你會發現它進了 `b3` 頁。這時你再按 back 掣就會看到它回到 `b2` ➔ `b1` ➔ `top`。`a3` 頁的 `secondaryOnClick` 用了 deep link 做轉頁，所以當前的 back stack 會被清空，然後換成適用於 `b3` 頁的 back stack。而這個 back stack 的生成方法是看 `b3` 頁所屬的 sub-graph (`partB2`) 的 start destination (`b2`)，所以 `b3` 的上一頁是 `b2` 頁。而 `b2` 頁的上一頁就是看它所屬的 sub-graph (`partB`) 的 start destination (`b1`)。按照同一原理 `b2` 頁的上一頁就是 `top` 頁。

如果我們把 `top` 的 `secondaryOnClick` 改為用 deep link 形式進入 `a3` 頁（即是改成下面的 code），你會看到它的 back stack 是 `a3` ➔ `a1` ➔ `top`。沒有 `a2` 是因為 `a3` 的 sub-graph `partA` 的 start destination 是 `a1`。換句話說 `a1`、`a2`、`a3` 都是平級。

```kotlin
composable(route = "top") {
    Screen(
        name = "top",
        onClick = { navController.navigate("partA") },
        secondaryOnClick = {
            val intents = navController.createDeepLink()
                .addDestination("a3")
                .createTaskStackBuilder()
                .intents
            navController.handleDeepLink(intents.first())
        },
    )
}
```

所以 sub-graph 的作用是在 deep linking 時控制 back stack，這種寫法是確保 Navigation component 能掌握該頁的上一頁，並且只有一頁能做上一頁（簡單來說是退路）。如果不是 deep linking 的話其實你可以隨便轉去整個 navigation graph 內的任何一頁，只要你知道它的 route 就可以了。另外，deep linking 不一定要配 URI（要在 *AndroidManifest.xml* 加上 `intent-filter` 那種），剛才例子的寫法是不用加 `intent-filter`。如果你要在某個很深層的頁面跳轉去另一個深層的頁面，用這個寫法會比每次特意控制 pop back stack 更加自然。當然，如果你是用 URI 那種 deep linking 的話，sub-graph 就是控制 deep link 開 app 後所造出來的 back stack。

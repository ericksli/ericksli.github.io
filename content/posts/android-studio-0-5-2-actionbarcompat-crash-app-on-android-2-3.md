---
title: Android Studio 0.5.2 + ActionBarCompat 在 Android 2.3 設備死 app
tags:
  - Android
date: 2014-03-23T17:19:29+08:00
---

最近開始轉用 Android Studio 寫 Android app。昨日升級到 0.5.2 時發現 gradle.xml 和另外兩個 .iml 檔案的路徑都被改為絕對路徑，而非先前的 `$PROJECT_DIR$` 和 `$MODULE_DIR$`。Google 過似乎還未有解決方法，現在惟有不 commit 這幾個檔案。

另一個問題是在 Android 2.3 的設備運行用了 ActionBarCompat 的 app 會「彈 app」。只要你的 activity 有 action overflow 的話，當按動設備的 Menu 掣時，activity 就會自行結束。但在 logcat 就不覺有 exception 的 stack trace。Google 過之後就發現原來新版 Android Studio 所附帶的 Gradle 用了新的 PNG cruncher 會令到 Android 2.3 死機。目前的解決方法是使用舊版 PNG cruncher。

<!--more-->

第一個方法是修改 project 目錄的 build.gradle

{{< highlight gradle >}}
// Top-level build file where you can add configuration options common to all sub-projects/modules.

buildscript {
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:0.9.0'
    }
}

allprojects {
    repositories {
        mavenCentral()
    }
}
{{< /highlight >}}

另一個是修改 project module 目錄的 build.gradle

{{< highlight gradle >}}
apply plugin: 'android'

android {
    compileSdkVersion 19
    buildToolsVersion "19.0.0"

    // Use old PNG cruncher because of crashes on GB
    android.aaptOptions.useAaptPngCruncher = true

    defaultConfig {
        minSdkVersion 9
        targetSdkVersion 19
        versionCode 1
        versionName "1.0"
    }
    buildTypes {
        release {
            runProguard false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.txt'
        }
    }
    signingConfigs {
        debug {
            storeFile file('debug.keystore')
        }
    }
}

dependencies {
    compile 'com.android.support:support-v4:19.0.1'
    compile 'com.android.support:gridlayout-v7:19.0.0'
    compile 'com.android.support:appcompat-v7:19.0.1'
    compile 'com.google.android.gms:play-services:4.2.42'
}
{{< /highlight >}}

改完之後按一下 Sync Project with Gradle Files，之後 build 出來的 app 就正常了。

## 參考

* [Android Issue Tracker - Issue 67376](https://code.google.com/p/android/issues/detail?id=67376)
* [Android Issue Tracker - Issue 67388](https://code.google.com/p/android/issues/detail?id=67388)

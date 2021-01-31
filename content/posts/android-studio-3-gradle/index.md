---
title: Android Studio 3 的 Gradle 更新
tags:
  - Android
  - Gradle
date: 2017-10-27T23:05:53+08:00
---


昨日使用 Android Studio 途中彈了 Android Studio 3 的更新通知，那時因為知道升級會有 breaking change，擔心升級要大改 Gradle 設定檔。所以延後了一天才更新。今日試了將現有的 project 更新，暫時未遇到問題，應該算是完成了。

{{< figure src="as3-update.png" title="Android Studio 3 的更新通知" >}}

<!-- more -->

Android Studio 升級完成後，開啟 project。Android Studio 會提示你升級 build tool 之類的東西，按照指示進行。Android Studio 會改動你的 Gradle wrapper，build tool 版本。但之後 Gradle project sync 時可能會有一大堆 error。這時可以試試以下的方法：

檢查 *app/build.gradle* 的 `buildToolsVersion` 是否太舊。在根目錄的 *build.gradle* 的 `ext` 內加入 `compileSdkVersion` 和 `buildToolsVersion`：

{{< highlight gradle >}}
ext {
    compileSdkVersion = 26
    buildToolsVersion = '26.0.2'
    // ...
}
{{< /highlight >}}

之後將 *app/build.gradle* 的 `compileSdkVersion` 和 `buildToolsVersion` 換成 `ext` 的 property：

{{< highlight gradle >}}
android {
    compileSdkVersion rootProject.ext.compileSdkVersion
    buildToolsVersion rootProject.ext.buildToolsVersion
    // ...
}
{{< /highlight >}}

然後在根目錄的 *build.gradle* 加入以下的內容。用意是將 subproject 的 Android 或 Android library project 的 `compileSdkVersion` 和 `buildToolsVersion` 強行設定成 `ext` 所訂的版本。如果你的 project 是 React Native 的話，應該會加入不少 subproject，而每個 subproject 都會有自己的 `compileSdkVersion` 和 `buildToolsVersion`，如果版本不一致的話是不能 build 的。你可以手動逐個更新 *build.gradle*，但之後用 NPM update 過那些 React Native component 之後你又要再手動改一次，非常麻煩。所以都是用下面的方法比較好：

{{< highlight gradle >}}
subprojects { subproject ->
    afterEvaluate {
        if ((subproject.plugins.hasPlugin('android') || subproject.plugins.hasPlugin('android-library'))) {
            android {
                compileSdkVersion rootProject.ext.compileSdkVersion
                buildToolsVersion rootProject.ext.buildToolsVersion
            }
        }
    }
}
{{< /highlight >}}

之後試試 build project，可能已經成功。如果不能的話，就要繼續改其他地方。例如我的 project 有自訂 `buildTypes`，現在需要補回 `matchingFallbacks` 確保其他 Android 或 Android library subproject 的 `buildTypes` 能配對到你的 `buildTypes`。

{{< highlight gradle >}}
android {
    // ...
    buildTypes {
        debug {
            debuggable true
            minifyEnabled false
            shrinkResources false
            applicationIdSuffix devAppIdSuffix
            versionNameSuffix devVersionNameSuffix
            signingConfig signingConfigs.debug
            manifestPlaceholders = [
                    // ...
            ]
            ext.alwaysUpdateBuildId = false
        }
        upload {
            minifyEnabled true
            shrinkResources true
            signingConfig signingConfigs.upload
            manifestPlaceholders = [
                    // ...
            ]
            matchingFallbacks = ['release']
        }
        // ...
    }
    // ...
}
{{< /highlight >}}

每個 `productFlavors` 都要加 `dimension`，之前無做就要補回。

{{< highlight gradle >}}
android {
    // ...
    flavorDimensions "api"
    productFlavors {
        development {
            dimension "api"
            buildConfigField "String", "ENVIRONMENT", "\"development\""
            buildConfigField "String", "API_ENDPOINT", "\"foo.com\""
        }
        production {
            dimension "api"
            buildConfigField "String", "ENVIRONMENT", "\"production\""
            buildConfigField "String", "API_ENDPOINT", "\"bar.com\""
        }
    }
    // ...
}
{{< /highlight >}}

加入 Google Maven repository。現在 Google 會將以下最新版本的 library 經這個 repository 發布。 

- Android Support Library
- Architecture Components Library
- Constraint Layout Library
- Android Test Support Library
- Databinding Library
- Android Instant App Library
- Google Play services
- Firebase

如果之前有加過 `maven { url 'https://maven.google.com' }` 的話，要將它刪走。

{{< highlight gradle >}}
buildscript {
    repositories {
        google()
        jcenter()
        // ...
    }
}

allprojects {
    repositories {
        google()
        jcenter()
        // ...
    }
}
{{< /highlight >}}

我的 project 到了現在已經能夠 build 到，如果不能的話要再查看其他文檔。

之後可以再將 dependency 設定更新到 Gradle 3.4 的新設定方式。`compile` 已經在 Gradle 3.4 開始 deprecated，要改用 `implementation` 或 `api`。`provided` 要轉用 `compileOnly`。跟據文檔的講法，應該要盡量使用 `implementation`。因為它會限制這個 dependency 的使用範圍，所以可以加快因改動 module 後再 build 的速度。如果出現 build error 的話，可以試試將 `implementation` 換成 `api`，`api` 的功能其實和之前的 `compile` 一樣。`debugCompile` 換成 `debugImplementation`；`testCompile` 換成 `testImplementation`……以下是一些轉用 `implementation` 例子：

{{< highlight gradle >}}
dependencies {
    implementation fileTree(dir: "libs", include: ["*.jar"])
    
    implementation "org.jetbrains.kotlin:kotlin-stdlib-jre7:$kotlinVersion"
    
    implementation project(':react-native-camera')
    
    implementation "com.android.support:appcompat-v7:$androidSupportLibVersion"
    implementation "com.google.dagger:dagger:$daggerVersion"
    kapt "com.google.dagger:dagger-compiler:$daggerVersion"
    
    debugImplementation "com.squareup.leakcanary:leakcanary-android-no-op:$leakCanaryVersion"
    debugLeakCanaryImplementation "com.squareup.leakcanary:leakcanary-android:$leakCanaryVersion"
    releaseImplementation "com.squareup.leakcanary:leakcanary-android-no-op:$leakCanaryVersion"

    androidTestImplementation('com.android.support.test.espresso:espresso-core:2.2.2', {
        exclude group: 'com.android.support', module: 'support-annotations'
        exclude group: 'com.google.code.findbugs', module: 'jsr305'
    })
    testImplementation 'junit:junit:4.12'
    testImplementation "org.robolectric:robolectric:3.4.2"
}
{{< /highlight >}}

註：`kapt` 是 Kotlin annotation processing tool，如果是 Java project 應該用 `annotationProcessor`。

## 參考

- [Migrate to Android Plugin for Gradle 3.0.0](https://developer.android.com/studio/build/gradle-plugin-3-0-0-migration.html)
- [Boost compilation time of your React Native or Android app with the latest build-tools](https://medium.com/bam-tech/boost-compilation-time-of-your-react-native-or-android-app-with-the-latest-build-tools-a3d5d398ed33)
- [What's the difference between implementation and compile in gradle](https://stackoverflow.com/questions/44493378/whats-the-difference-between-implementation-and-compile-in-gradle/44493379#44493379)
- [Implementation vs API dependency](https://jeroenmols.com/blog/2017/06/14/androidstudio3/)

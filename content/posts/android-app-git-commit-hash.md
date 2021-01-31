---
title: 在 Android app 內顯示 Git commit hash
tags:
  - Android
  - Git
date: 2016-08-29T22:13:46+08:00
---


有些 Android app 除了顯示版本號碼之外，還會有該版本的 Git commit hash。如果有好幾部測試機做測試的話，可能會在開發時先後在不同的機安裝過不同版本的 app。但就沒有每次都更新版本號碼。加了 commit hash 就容易分辨 app 的實際版本。其實要顯示 commit hash 的做法不太難，不需要每次人手更改的。只需要改一下 *app* 的 *build.gradle* 就可以了。

<!--more-->

{{< highlight groovy >}}
import java.util.regex.Pattern

apply plugin: 'com.android.application'

/**
 * Extract Git commit hash of HEAD with number of changed files
 */
def getGitHashVersionName = {
    try {
        def hashOutput = new ByteArrayOutputStream()
        def changeOutput = new ByteArrayOutputStream()
        def gitVersionName
        exec {
            commandLine 'git', 'rev-list', '--max-count=1', 'HEAD'
            standardOutput = hashOutput
        }
        exec {
            commandLine 'git', 'diff-index', '--shortstat', 'HEAD'
            standardOutput = changeOutput
        }
        gitVersionName = hashOutput.toString().trim().substring(0, 7);
        if (!changeOutput.toString().trim().empty) {
            def pattern = Pattern.compile("\\d+");
            def matcher = pattern.matcher(changeOutput.toString().trim())
            if (matcher.find()) {
                gitVersionName += "-" + matcher.group()
            }
        }
        return gitVersionName
    } catch (ignored) {
        return "UNKNOWN";
    }
}

android {
    // Skip the other items...

    buildTypes {
        debug {
            debuggable true
            minifyEnabled false
            signingConfig signingConfigs.debug
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
            applicationIdSuffix debug_application_id_suffex
            buildConfigField "String", "GIT_VERSION_NAME", "\"${getGitHashVersionName()}\""
        }
        release {
            minifyEnabled true
            signingConfig signingConfigs.release
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
            buildConfigField "String", "GIT_VERSION_NAME", "\"${getGitHashVersionName()}\""
        }
    }
}
{{< /highlight >}}

上面的 `getGitHashVersionName` function 就是用 `git getGitHashVersionName` 和 `git diff-index` 取得 commit hash 和未 commit 的檔案數目。如果有未 commit 的檔案的話，commit hash 後面就會補上檔案數目，方便分辨。

在 `buildTypes` 內的 `buildConfigField` 就是為 Gradle 每次 build 都會自動產生的 `BuildConfig` class 加入一個 `public static final` 的 field。只需要在 app 的 source code 直接用 `BuildConfig.GIT_VERSION_NAME` 就能拿到 Git commit hash。

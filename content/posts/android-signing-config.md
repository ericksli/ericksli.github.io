---
title: Android 隱藏 signing config
tags:
- Android
date: 2017-04-25T22:53:27+08:00
---


其實[官方網站](https://developer.android.com/studio/publish/app-signing.html#secure-shared-keystore)有介紹過做法，不過就令到 build 那時一定要有 *keystore.properties*，否則就不能 build。部分 CI 可能會針對 Android 會提供專門的方式來設定 release keystore 和密碼。而在 VCS checkout source code 後在 file system 補上 *keystore.properties* 和 keystore 未必可以在 CI 環境上做到。所以我將那個教學稍作改動，令到當 *keystore.properties* 不存在時就不提供 signing config，使它能 build 未加簽的 apk，然後才讓 CI 加簽 apk。

{{< highlight groovy >}}
apply plugin: 'com.android.application'

// Release signing properties
def keystorePropertiesFile = rootProject.file("keystore.properties");
def keystoreProperties = new Properties()
if (keystorePropertiesFile.exists()) {
    keystoreProperties.load(new FileInputStream(keystorePropertiesFile))
}

android {
    compileSdkVersion 25
    buildToolsVersion '25.0.2'

    defaultConfig {
        applicationId "com.example.helloworld"
        minSdkVersion 16
        targetSdkVersion 25
        versionCode 1
        versionName "1.0"
        proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
    }

    signingConfigs {
        debug {
            storeFile file("../app/debug.keystore")
            storePassword "android"
            keyAlias "androiddebugkey"
            keyPassword "android"
        }
        if (keystorePropertiesFile.exists()) {
            release {
                storeFile file(keystoreProperties['storeFile'])
                storePassword keystoreProperties['storePassword']
                keyAlias keystoreProperties['keyAlias']
                keyPassword keystoreProperties['keyPassword']
            }
        }
    }

    buildTypes {
        debug {
            debuggable true
            minifyEnabled false
            signingConfig signingConfigs.debug
        }
        release {
            minifyEnabled true
            shrinkResources true
            if (keystorePropertiesFile.exists()) {
                signingConfig signingConfigs.release
            }
        }
    }
}

dependencies {
    // ...
}
{{< /highlight >}}

{{< highlight properties >}}
storeFile = ../app/release.keystore
storePassword = mypassword
keyAlias = myalias
keyPassword = mypassword
{{< /highlight >}}

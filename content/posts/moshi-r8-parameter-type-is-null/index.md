---
title: Moshi Kotlin Codegen + R8 出現 parameter type is null
tags:
  - Android
date: 2020-03-15T11:10:42+08:00
---


[Moshi](https://github.com/square/moshi) 是一個 JSON serialization/deserialization 的 library。和 Gson 不同的是它提供了 Kotlin Codegen，它可以生成 serialization/deserialization 的 adapter class，所以可以避免使用 reflection，而且 adapter 還會參照 Kotlin 的 non-null 和 default value。不過最近發現 production app 會出現 parameter type is null 的錯誤訊息。

<!-- more -->

{{< figure src="before-fix.png" title="parameter type is null" >}}

```text
java.lang.NoSuchMethodException: parameter type is null
    at java.lang.Class.getConstructor0(Class.java:2322)
    at java.lang.Class.getDeclaredConstructor(Class.java:2166)
    at net.swiftzer.metroride.remote.mtrupdate.status.LineStatusResponse_LineStatusJsonAdapter.a(LineStatusResponse_LineStatusJsonAdapter.kt:70)
    at net.swiftzer.metroride.remote.mtrupdate.status.LineStatusResponse_LineStatusJsonAdapter.a(LineStatusResponse_LineStatusJsonAdapter.kt:19)
    at f.d.a.t.a.a(NullSafeJsonAdapter.java:40)
    at f.d.a.d.a(CollectionJsonAdapter.java:76)
    at f.d.a.d$b.a(CollectionJsonAdapter.java:53)
    at f.d.a.t.a.a(NullSafeJsonAdapter.java:40)
    at net.swiftzer.metroride.remote.mtrupdate.status.LineStatusResponseJsonAdapter.a(LineStatusResponseJsonAdapter.kt:47)
    at net.swiftzer.metroride.remote.mtrupdate.status.LineStatusResponseJsonAdapter.a(LineStatusResponseJsonAdapter.kt:21)
    at f.d.a.t.a.a(NullSafeJsonAdapter.java:40)
    at q.z.a.c.a(MoshiResponseBodyConverter.java:45)
    at q.z.a.c.a(MoshiResponseBodyConverter.java:27)
    at q.m.a(OkHttpCall.java:225)
    at q.m.h(OkHttpCall.java:188)
    at q.y.a.c.b(CallExecuteObservable.java:45)
    at i.a.e.a(Observable.java:12268)
    at q.y.a.a.b(BodyObservable.java:34)
    at i.a.e.a(Observable.java:12268)
    at i.a.p.e.b.d.b(ObservableSingleSingle.java:35)
    at i.a.i.a(Single.java:3603)
    at i.a.p.e.c.b.b(SingleMap.java:34)
    at i.a.i.a(Single.java:3603)
    at i.a.p.e.c.e.b(SingleZipArray.java:63)
    at i.a.i.a(Single.java:3603)
    at i.a.p.e.c.b.b(SingleMap.java:34)
    at i.a.i.a(Single.java:3603)
    at i.a.p.e.c.d$a.run(SingleSubscribeOn.java:89)
    at i.a.h$a.run(Scheduler.java:578)
    at i.a.p.f.h.run(ScheduledRunnable.java:66)
    at i.a.p.f.h.call(ScheduledRunnable.java:57)
    at java.util.concurrent.FutureTask.run(FutureTask.java:266)
    at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:301)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
    at java.lang.Thread.run(Thread.java:764)
```

Google 過才發現 R8 的問題。如果將 R8 改用 2.0.39 版就會變回正常。

```groovy
buildscript {
    repositories {
        maven {
            url 'https://storage.googleapis.com/r8-releases/raw'
        }
    }

    dependencies {
        classpath 'com.android.tools:r8:2.0.39.         // Must be before the Gradle Plugin for Android.
        classpath 'com.android.tools.build:gradle:X.Y.Z' // Your current AGP version.
    }
}
```

效果：

{{< figure src="after-fix.png" title="改了 R8 版本後 Moshi 能正常 deserialize JSON" >}}

## 參考

- [Moshi 1.9.2: Crash because Reflection in Codegen not handled in Proguard](https://github.com/square/moshi/issues/1049)
- [4.0.0-alpha08 - java.lang.RuntimeException: Cannot create an instance of class ViewModel](https://issuetracker.google.com/u/1/issues/147972078#comment9)

## 註

現在（2020 年 4 月 26 日）的 R8 版本應該沒有這個問題。

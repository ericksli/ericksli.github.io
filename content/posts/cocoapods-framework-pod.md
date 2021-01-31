---
title: 自己造一個 CocoaPods Framework Pod
tags:
  - iOS
  - tvOS
  - CocoaPods
date: 2016-09-28T10:53:11+08:00
---

最近工作需要將一個 tvOS app 的某些 class 抽出來變成能被 CocoaPods 安裝的私家 close source framework 讓其他人用。我都是第一次做 framework，在網上的教學主要都是教純 Xcode 的做法和使用 CocoaPods 下載 project dependency，詳細提及如何造一個 CocoaPods Pod 就比較少人寫。所以就寫出來跟大家分享。

<!--more-->

[CocoaPods](https://cocoapods.org/) 是管理 Cocoa project 的 dependency manager。其他同類的還有 [Carthage](https://github.com/Carthage/Carthage)，但目前 CocoaPods 還是最多人用。假如我們想做一個叫 *MyOwnPod* 的私人 CocoaPods close source framework，要準備三個 Git repository：

1. 用來存放 framework 源碼 (myownpod)
   只讓內部組員存取
2. 用來存放已 compile 的 CocoaPods Pod (myownpod-release)
   讓用家存取
3. 作為 Specs Repository (specs-repo)
   讓用家存取，非必須

我就將步驟以這三個 repository 分成三部分寫。

## Framework 源碼  (myownpod)

要造一個 CocoaPods 的 pod（即是 dependency）並不需要開一個 Xcode framework project，只需要準備好 source code 和 podspec 檔案就可以了。因為 podspec 內會定義版本、source code 位置、public header file、要調用那些 framework 等等本來是在 Xcode framework project 定義的設定。如果你的 pod 需要依賴其他 pod 的話，也是定義在 podspec 內而不是用 podfile 來定義。

一個 framework 通常都會有一個示範用的 Xcode project 來讓其他人知道 framework 的用法，而且在開發 framework 時還可以借它來測試一下。不過是 close source 的話這個示範 Xcode project 應該要另外做多一個放到其他地方。因為這個 repository 是不會讓其他人看到的。

整個 repository 的檔案結構大概概會是這樣：

```text
.
├── Example
│   ├── MyOwnPodExample
│   │   ├── AppDelegate.h
│   │   ├── AppDelegate.m
│   │   ├── Assets.xcassets
│   │   ├── Base.lproj
│   │   ├── Info.plist
│   │   ├── ViewController.h
│   │   ├── ViewController.m
│   │   └── main.m
│   └── MyOwnPodExample.xcodeproj
├── LICENSE
├── Podfile
├── Podfile.lock
├── Pods
├── README.md
├── MyOwnPod
│   ├── MyClassA.h
│   ├── MyClassA.m
│   ├── MyClassB.h
│   ├── MyClassB.m
│   ├── MyClassC.h
│   ├── MyClassC.m
│   ├── MyClassC.xib
├── MyOwnPod.podspec
└── MyOwnPod.xcworkspace
```

*Example* 目錄是用來放示範用的 Xcode project；而 *MyOwnPod* 目錄就是用來放 framework source code。如果按照 [CocoaPods 的建議](https://guides.cocoapods.org/making/using-pod-lib-create.html)，這個目錄應該是要叫做 *Pod*。

首先，將 framework source code 放在 *MyOwnPod* 目錄內，然後準備 podspec 檔。

- `s.version` 要用 [semantic versioning](http://semver.org/)。
- `s.platform` 用了 `:tvos` 是因為這個 framework 是給 tvOS 用。如果是給 iOS 用的話就要寫成 `:ios`。
- `s.public_header_files` 是用來標示那些 header file 可以被使用者看到，CocoaPods 會用它來生成 umbrella header。
- 這個 framework 需要用到 libxml2，所以多了 `s.library` 和 `s.xcconfig`，如果不需要用的話是不用加的。

{{< highlight ruby >}}
Pod::Spec.new do |s|
  s.name             = 'MyOwnPod'
  s.version          = '0.1.0'
  s.summary          = 'My own pod for demo.'

  s.description      = <<-DESC
This is just an example pod.
                       DESC

  s.homepage         = 'https://gitlab.com/ericksli/myownpod'
  # s.screenshots     = 'www.example.com/screenshots_1', 'www.example.com/screenshots_2'
  s.license          = { :type => 'proprietary', :file => 'LICENSE' }
  s.author           = { 'Eric Li' => 'eric@swiftzer.net' }
  s.source           = { :git => 'ssh://git@gitlab.com:ericksli/myownpod.git', :tag => s.version.to_s }
  s.social_media_url = 'https://twitter.com/ericksli'

  s.platform = :tvos, '9.0'

  s.source_files = 'MyOwnPod/**/*.{h,m}'
  
  s.resource_bundles = {
    'MyOwnPod' => ['MyOwnPod/**/*.{storyboard,xib}']
  }

  s.public_header_files = [
    'MyOwnPod/MyClassA.h',
    'MyOwnPod/MyClassB.h'
  ]
  s.frameworks = 'UIKit', 'CoreGraphics'
  s.library = 'xml2'
  s.xcconfig = { 'HEADER_SEARCH_PATHS' => '$(SDKROOT)/usr/include/libxml2' }
  s.dependency 'Masonry'
  s.tvos.deployment_target = '9.0'
  s.requires_arc = true
end
{{< /highlight >}}

然後，用 Xcode 以平時的方法在 *Example* 目錄建立一個新 Xcode project。之後將 project 關閉並建立 Podfile。Podfile 就是要將 framework pod 加到示範 project 內。

{{< highlight ruby >}}
platform :tvos, '9.0'
use_frameworks!

workspace 'MyOwnPod'

target 'MyOwnPodExample' do
  project 'Example/MyOwnPodExample.xcodeproj'
  pod 'MyOwnPod', :path => '.'
end
{{< /highlight >}}

準備好 Podfile 後就可以用 `pod install` 安裝了，執行完之後會多了一個 MyOwnPod.xcworkspace 檔。以後就要用這個檔案來開啟示範 project。要修改 framework 的檔案可以在 Xcode 的 project navigator 中展開 Pods > Development Pods > MyOwnPod。內裹的檔案都是 symlink framework 的檔案。如果要增加或者刪除 framework 的檔案，需要再執行多次 `pod install` 或 `pod update`。

## CocoaPods Pod Repository (myownpod-release)

第二個要準備的 Git repository 就是用來放已 compile 的 framework。這個我們需要用 [CocoaPods Packager](https://github.com/CocoaPods/cocoapods-packager) 來替我們將 podspec 檔所定義的 pod build 成 framework。

```bash
pod package MyOwnPod.podspec --dynamic --force --verbose
```

執行完後，應該會生成到 framework 的目錄。我們可以用 `lipo` 檢查一下它支援的 architecture：

```bash
lipo -info MyOwnPod.framework/MyOwnPod
```

以 tvOS 為例，framework 應該要有齊 `x86_64` and `arm64`，這樣就能在 simulator 和實機上使用。但 CocoaPods Packager 目前只支援 iOS 和 macOS，如果要順利 build 到 tvOS framework 就要[修改它的 source code](https://github.com/CocoaPods/cocoapods-packager/pull/161)。

之後就可以準備 Git repository 的內容。將生成出來的內容抄到 Git repository 內，結構會是這樣：

```text
.
├── MyOwnPod.podspec
└── tvos
    └── MyOwnPod.framework
```

這個 podspec 就是剛才 CocoaPods Packager 生成出來的，它會直接指向用 build 出來的 framework 而不是用 source code。Commit 後就要為這個 commit 加上版本號 tag。如果你的 `s.version` 是 `0.1.0` 的話，你就要 tag 它做 `0.1.0`。日後再出新版的話就將 repository 的檔案重新覆蓋，commit 完再為它加上新的 tag 然後 push 就可以了，只需要用 master branch，不用特別開其他 branch。CocoaPods 會用 tag 來找尋適當的版本。

Push 完之後如果要用這個 framework 的話 Podfile 會直接使用 Git repository URL 和 tag 名：

{{< highlight ruby >}}
platform :tvos, '9.0'
use_frameworks!

target 'MyOwnTVApp' do
    pod 'MyOwnPod', :git => 'git@gitlab.com:ericksli/myownpod-release.git', :tag => '0.1.0'
end
{{< /highlight >}}

## 私家 Specs Repository (specs-repo)

最後一個 Git repository 就是要做一個私家 specs repository，這部分不是必需的。Spec repository 就是用來集中存放不同 pod 的 podspec，如果你造了不少私家 pod 的話，可以考慮做一個 specs repository。這個 Git repository 和剛才的 pod repository 一樣，都是只用 master branch。但就不用加 tag。下面就是檔案結構：

```text
.
└── Specs
    ├── MyOwnPod
    │   ├── 0.1.0
    │   │   └── MyOwnPod.podspec
    │   └── 0.2.3
    │       └── MyOwnPod.podspec
    └── AnotherPod
        └── 5.6.7
            └── AnotherPod.podspec
```

這個 repository 所需要用到的 podspec 可以抄之前用 CocoaPods Packager 產生的 podspec，一個版本就有一個對應的目錄和 podspec。但要將 `s.source` 改成 CocoaPods pod repository 的 URL。

{{< highlight ruby >}}
Pod::Spec.new do |s|
  s.name = 'MyOwnPod'
  s.version = '0.1.0'
  s.summary = 'My own pod for demo.'
  s.license = {"type"=>"proprietary", "file"=>"LICENSE"}
  s.authors = {"Eric Li"=>"eric@swiftzer.net"}
  s.homepage = 'https://gitlab.com/ericksli/myownpod'
  s.description = 'This is just an example pod.'
  s.social_media_url = 'https://twitter.com/ericksli'
  s.frameworks = ["UIKit", "CoreGraphics"]
  s.libraries = 'xml2'
  s.requires_arc = true
  s.xcconfig = {"HEADER_SEARCH_PATHS"=>"$(SDKROOT)/usr/include/libxml2"}
  s.source = { :git => 'ssh://git@gitlab.com:ericksli/myownpod-release.git', :tag => s.version.to_s }

  s.tvos.deployment_target    = '9.0'
  s.tvos.vendored_framework   = 'tvos/MyOwnPod.framework'
end
{{< /highlight >}}

Push 完之後就可以使用這個 specs repository。在使用前，要先加入這個 repository（名稱沒有規定）：

```bash
pod repo add eric ssh://git@gitlab.com:ericksli/specs-repo.git
```

加入後，可以用 `pod repo` 檢查是否有官方（名稱是 master）和自己的 spec repository：

```text
eric
- Type: git (master)
- URL:  ssh://git@gitlab.com:ericksli/specs-repo.git
- Path: /Users/eric/.cocoapods/repos/eric

master
- Type: git (master)
- URL:  https://github.com/CocoaPods/Specs.git
- Path: /Users/eric/.cocoapods/repos/master

2 repos
```

要在 project 使用自己的 pod，Podfile 的寫法基本上和平時差不多。但要用 `source` 加入官方和自己的 spec repository 才可使用自己的 pod。

{{< highlight ruby >}}
source 'https://github.com/CocoaPods/Specs.git'
source 'ssh://git@gitlab.com:ericksli/specs-repo.git'

platform :tvos, '9.0'
use_frameworks!

target 'MyOwnTVApp' do
    pod 'OtherThirdPartyPod'
    pod 'MyOwnPod', '~> 0.1.0'
end
{{< /highlight >}}

執行一次 `pod install` 就會下載剛才自製的 pod。

## 補充

- 如果是做 open source pod 的話其實只需要做第一部分，然後直接交上去官方 repository 就完成。
- 其實 CocoaPods 有 [`pod lib create`](https://guides.cocoapods.org/making/using-pod-lib-create.html) 指令來幫你起步，可以直接用它。
- CocoaPods 建議用 Git repository 用 HTTPS 而非 SSH 連接，但私人使用的話其實也沒有甚麼問題。
- CocoaPods Packager 在 build 的時候應該會同時 build 不同的 architecture（即是 universal framework），如果是 open source pod 的話就不用擔心 architecture 問題，因為用家可以自己控制用甚麼 architecture。

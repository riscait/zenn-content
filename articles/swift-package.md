---
title: "Swift Packageで共通ファイルを使用する"
emoji: "👏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---
Xcode File New Swift Package...Xcode File New Swift Package...

##
[iOS 14 Widgets + SwiftUI + Firebase?](https://stackoverflow.com/a/63748193/12017315)

## XcodeGen でパッケージを使う
[XcodeGen - ProjectSpec - Local-package](https://github.com/yonaskolb/XcodeGen/blob/master/Docs/ProjectSpec.md#local-package)

```yaml:project.yml
packages:
  MyPackage:
    path: ./MyPackage

targets:
  SampleApp:
    dependencies:
      - package: MyPackage
  SampleAppWidget:
    dependencies:
      - package: MyPackage
```
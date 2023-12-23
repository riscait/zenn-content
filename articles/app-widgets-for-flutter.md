---
title: "FlutterでiOS 14のWidgetsを実装してみた"
emoji: "🦢"
type: "tech"
topics: ["flutter"]
published: false
publication_name: "altiveinc"
---
こんにちは、Flutterでのアプリ開発をメインとしている「[Altive株式会社](https://altive.co.jp)」の村松龍之介（[@riscait](https://x.com/riscait)）です！

日中は企業でiOSアプリのネイティブ開発を、個人ではFlutter x Firebaseで iOS & Androidアプリを作っています。

@[tweet](https://twitter.com/riscait/status/1308171643089379329)


## Widgetについて
Application ExtensionごとにiOS Deployment Targetが設定でき、アプリ自体のDeployment Target

Product Name: Extension名

✅ Include Configration Intent
> ウィジェットがユーザー構成可能なプロパティを提供する場合は、「構成意図を含める」チェックボックスをオンにします。

## 作成されたファイル
```
ReStockWidget
  ReStockWidget.swift
  ReStockWidget.intentdefinition
  Assets.xcassets
  Info.plist
```

## BitcodeをNOに

```
'/Users/riscait/Library/Developer/Xcode/DerivedData/Runner-dtvhbpvdjpkglmbuzeqfnnvslhfq/Build/Products/Debug-iphoneos/AppAuth/AppAuth.framework/AppAuth' does not contain bitcode. You must rebuild it with bitcode enabled (Xcode setting ENABLE_BITCODE), obtain an updated library from the vendor, or disable bitcode for this target. file '/Users/riscait/Library/Developer/Xcode/DerivedData/Runner-dtvhbpvdjpkglmbuzeqfnnvslhfq/Build/Products/Debug-iphoneos/AppAuth/AppAuth.framework/AppAuth' for architecture arm64
```

## AppIcon名がセットされていた
```
/Users/riscait/development/Git/Riscait/rolling_stock/ios/ReStockWidget/Assets.xcassets:1:1: None of the input catalogs contained a matching stickers icon set or app icon set named  "AppIcon-development".
```
Asset Catalog App Icon Set Name
ResStockWidgetを削除してResolvedを空に。

https://makotton.com/2014/10/03/559

## Configuration
```
configuration didn't match to Development, Staging or Production
Debug
Command PhaseScriptExecution failed with a nonzero exit code
```

Edit Scheme
Run
Debug-Developmentを選択

## 宣伝

Altive株式会社では、Flutterアプリの開発・運営を承っております。
お気軽にお問い合わせください🫡 
https://altive.co.jp/contact

---

Riverpod の実践入門本を公開中です📘
https://zenn.dev/riscait/books/flutter-riverpod-practical-introduction

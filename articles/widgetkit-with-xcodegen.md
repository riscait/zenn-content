---
title: "iOS 14のWidgetをXcodeGenを使ってPreviewさせるまでの設定"
emoji: "🔨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [xcode, xcodegen, ios, widgetkit]
published: true
publication_name: "altiveinc"
---

2020年10月25日：Intents Extension を追加した場合のTarget, Schemeを追記しました！

---

こんにちは、Flutterでのアプリ開発をメインとしている「[Altive株式会社](https://altive.co.jp)」の村松龍之介（[@riscait](https://x.com/riscait)）です！です。

iOS 14 で追加されたWidget(WidgetKit)を実装してみよう！
と思い立ち、プロジェクトを作り始めましたが、XcodeGenとの併用に詰まりました。

数時間かけて `project.yml` を書き換えて、WidgetのPreviewまで漕ぎ着けたので備忘録として残します。

Widgetを実装していくにあたり、もし今後、追加設定が必要だった場合は書き足していきたいと思います。

## まずはXcodeでWidget Extensionの追加

Xcode -> File -> New -> Target から、
Application Extensionの「Widget Extension」を選択して作成。

ここでは仮に `SampleWidget` という名前で Widget Extensionを作成したとします。

すると以下のファイルが新規作成されます。
ちなみに追加後に表示されるプロンプトで `Activate` を選択すると、自動で `Scheme` も追加されます。

```
SampleWidget/
  Assets.xcassets
  SampleWidget.intentdefinition
  SampleWidget.swift
  info.plit
```

## project.yml に追記
このWidget Extensionを新規追加した状態をXcodeGenで再現させるために、設定情報を見比べて `project.yml` ファイルに追記しました。

プロジェクトによっては、すでに追加済みだったり、追加不要なものもあるかもしれませんが、できるだけWidget追加時の構成に合わせるようにしました。

最後に `project.yml` ファイルの全体像を載せてあるので、そちらを見た方が分かりやすいかもしれません。

ここでは、プロジェクト名を `SampleApp` として解説します。

### settingsを追加
`VALIDATE_PRODUCT` が true となっていたので追記します。

```yml:project.yml
name: SampleApp # プロジェクト名

options:
  deploymentTarget:
    iOS: 14.0 # 対象下限iOSバージョン

settings: # Project の Build Setting
  configs:
    Release:
      VALIDATE_PRODUCT: true # 生成されたProductに対して検証テストを行うかどうか
```

### SampleAppのTargetに追記

```yml:project.yml
targets:
  SampleApp: # iOSアプリ本体のターゲット
    type: application
    platform: iOS
    sources:
      - SampleApp
      # Widgetのintentdefinitionファイルパスを追加
      - path: SampleApp/SampleWidget.intentdefinition
    settings:
      base:
        PRODUCT_BUNDLE_IDENTIFIER: com.example.SampleApp
        PRODUCT_NAME: $(TARGET_NAME)
        # Preview Contentのパスを追加
        DEVELOPMENT_ASSET_PATHS: "\"SampleApp/Preview Content\""
        # 追加
        ALWAYS_EMBED_SWIFT_STANDARD_LIBRARIES: true
        # プレビューを有効化
        ENABLE_PREVIEWS: true
        # アクセントカラー名を追加
        ASSETCATALOG_COMPILER_GLOBAL_ACCENT_COLOR_NAME: AccentColor
    dependencies:
      # Widget Extensionのターゲットを追加
      - target: SampleWidget
```

Widget Extensionのターゲットを追記
Bundle IDや、`dependencies` に `SwiftUI` , `WidgetKit` SDKを追加しましょう。

```yml:project.yml
  SampleWidget: # Widget Extension
    type: app-extension
    platform: iOS
    settings:
      base:
        ## Bundle IDを忘れず追加
        PRODUCT_BUNDLE_IDENTIFIER: com.example.SampleApp.SampleWidget
        INFOPLIST_FILE: SampleWidget/Info.plist
        DEVELOPMENT_TEAM: XXXXXXXXXX
        SKIP_INSTALL: true
        PRODUCT_NAME: $(TARGET_NAME)
        ASSETCATALOG_COMPILER_GLOBAL_ACCENT_COLOR_NAME: AccentColor
        ASSETCATALOG_COMPILER_WIDGET_BACKGROUND_COLOR_NAME: WidgetBackground
    sources:
      - SampleWidget
    dependencies:
      - sdk: SwiftUI.framework
      - sdk: WidgetKit.framework
```

Widget Extension用のSchemeを追加
`environmentVariables` などを忘れずに追加します。

```yml:project.yml
schemes:
  SampleWidgetExtension: # Scheme名は任意のものを
    build:
      targets:
        SampleWidget: all
        SampleApp: all
    run:
      config: Debug
      askForAppToLaunch: true
      debugEnabled: false
      environmentVariables:
        - variable: _XCWidgetKind
          value:
          isEnabled: false
        - variable: _XCWidgetDefaultView
          value: timeline
          isEnabled: false
        - variable: _XCWidgetFamily
          value: medium
          isEnabled: false
    test:
      config: Debug
    profile:
      config: Release
      askForAppToLaunch: true
    analyze:
      config: Debug
    archive:
      config: Release
```

## Intents Extensionを追加した場合
![](https://storage.googleapis.com/zenn-user-upload/k9940df92mjxutwiv9n2t59lwnjx)

Widget を長押しした時に「Widgetを編集」というメニューを表示し、その中で動的なリストでユーザーに選択肢を与えたい場合は、Intents Extensionが必要になります。

### まずは、XcodeからIntents Extensionを追加する

Xcode -> File -> New -> Target から、
Application Extensionの「Intents Extension」を選択して作成します。

![](https://storage.googleapis.com/zenn-user-upload/dg2rd63364wz6sm64dgzvyxca2rn)

ここでは仮に `SampleIntent` という名前で Widget Extensionを作成したとします。

![](https://storage.googleapis.com/zenn-user-upload/0bpblzbe027m8n53r7hbul1b138m)

すると以下のファイルが新規作成されます。
ちなみに追加後に表示されるプロンプトで `Activate` を選択すると、自動で `Scheme` も追加されます。

![](https://storage.googleapis.com/zenn-user-upload/nbsyej2bwmo7lwnarm39bnvzxdeo)

以下のファイルが生成されました。

```
SampleIntent/
  IntentHandler.swift
  info.plit
```

## project.yml に追記
`.intentdefinition` でTypeを追加したり、parameterを設定したりしなくてはいけませんが、
この記事では XcodeGen に焦点を当てているためその部分の実装は割愛します🙏

```yml:project.yml
  SampleIntent: # Intents Extension
    type: app-extension
    platform: iOS
    settings:
      base:
        ## Bundle IDを忘れず追加
        PRODUCT_BUNDLE_IDENTIFIER: com.example.SampleApp.SampleIntent
        INFOPLIST_FILE: SampleIntent/Info.plist
        SKIP_INSTALL: true
        PRODUCT_NAME: $(TARGET_NAME)
    sources:
      - SampleIntent
      - SampleWidget/SampleWidget.intentdefinition
```

Intents Extension用のSchemeを追加

```yml:project.yml
schemes:
  SampleIntent: # Scheme名は任意のものを
    build:
      targets:
        SampleIntent: all
        SampleApp: all
    run:
      config: Debug
      askForAppToLaunch: true
      debugEnabled: false
    test:
      config: Debug
    profile:
      config: Release
      askForAppToLaunch: true
    analyze:
      config: Debug
    archive:
      config: Release
```

## project.yml の全体像
最後に僕のアプリプロジェクトで実際にWidgetのPreviewが成功した `project.yml` ファイル全体を載せておきます。

アプリで実際に使用している `project.yml` ファイルのリンクです。
https://github.com/Riscait/Graphidget/blob/develop/project.yml

## Schemeのアイコンが違うのが心残り
XcodeでWidget ExtensionとSchemeを追加したばかりのSchemeアイコン
![](https://storage.googleapis.com/zenn-user-upload/9aw2y9c5xc30crvz22vbb8x832q8)

XcodeGenでプロジェクト生成した場合のSchemeアイコン
![](https://storage.googleapis.com/zenn-user-upload/4ofxvfr0ix1n62btupp4vnx3ok7u)

(E) になって欲しいのですが、アプリアイコンになってしまいます。

なぜこうなるのか…
![](https://storage.googleapis.com/zenn-user-upload/hzhu0yllds4o3kn07fdm6j0vytqn)
見比べてみると、 `Build` のTargetsの並び順が怪しいです。XocdeGenで指定した順じゃなくて、名前順になっているからかも？

![](https://storage.googleapis.com/zenn-user-upload/5kkuftf210t5xt8ckj7expyf4gnm)

このままでもPreviewできるので問題なさそうですが、気になっちゃいますよね👀

---

最後までご閲覧ありがとうございます🙏

もっと良い書き方があるよ！や書いてみてこっちの方が良かった！という構成があれば教えていただけると嬉しいです。

Xでは、主にアプリ開発について呟いております。
フォローしていただければ嬉しいです☺️ → 村松龍之介（[@riscait](https://x.com/riscait)）

## 宣伝

Altive株式会社では、Flutterアプリの開発・運営を承っております。
お気軽にお問い合わせください🫡 
https://altive.co.jp/contact

---

Riverpod の実践入門本を公開中です📘
https://zenn.dev/riscait/books/flutter-riverpod-practical-introduction

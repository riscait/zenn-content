---
title: "【Flutter 3.7以上】Dart-define-from-fileを使って開発環境と本番環境を分ける"
emoji: "🔨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [flutter, flavor, dart]
published: true
publication_name: "altiveinc"
---

| 開発環境 | ステージング環境 | 本番環境 |
| ---- | ---- | ---- |
| ![](/images/separating-environments-in-flutter/flavor-dev-ios-screenshot.png =200x) | ![](/images/separating-environments-in-flutter/flavor-stg-ios-screenshot.png =200x) | ![](/images/separating-environments-in-flutter/flavor-prod-ios-screenshot.png =200x) |

# はじめに

こんにちは、Flutterアプリ開発者の村松龍之介（[@riscait](https://twitter.com/riscait)）です😸

Flutterにおいて、その環境分けの方法はいくつかありますが、今回は `Dart-defines-from-file` のみを使用して実現する方法をご紹介します。

従来は `dart-define` を使用して環境分けをしていましたが、Flutter 3.7 で `dart-define-from-file` が使用できるようになり、従来の方法に比べてだいぶ簡単に環境が分けられるようになりました！

特に改善されたのは、iOSでスクリプトを書いたり `xcconfig` ファイルを作成する必要がなくなったところだと思います👍

:::message
`dart-define-from-file` ではなく、 `dart-define` で環境分けをすでに実行済みで、 `dart-define-from-file` を使用した方法に変更したい場合は、以下のプルリクエストのファイル差分を参考にしてください👌
https://github.com/altive/flutter_app_template/pull/164/files

また、`dart-define` を使用した旧版もアーカイブとして別記事に残しました。必要であれば参照してください。
https://zenn.dev/altiveinc/articles/separating-environments-in-flutter-old-edition
:::

:::message
個人開発では無くてもなんとかなるかもしれませんが、複数人が関わったり規模の小さくないアプリを作成する場合、**開発・ステージング（テスト）・本番のように複数環境が必要**になってきます。
:::

# `dart-define-from-file` のメリット

Flavor用のパッケージを使う場合等と比べて、**ビルドコマンドがシンプルになる**ことや、**自動生成ファイルが少なく、取り回しがしやすい**ことが大きな利点です。

- パッケージの導入が不要（アイコン生成には使用します）
- `--dart-define-from-file` のみでiOS, Androidネイティブへも「環境分け」を伝播できる
- `main.dart` を環境ごとに分ける必要がない
- iOSの `scheme` や `Configuration` を作成する必要がない

# 前提
- この記事では以下の3環境に分けていきます
（あくまで一例なので、数や名前は適宜変更してください）
  - `dev`: ローカルの開発環境
  - `stg`: 疑似的なDB等を用意し、テストを行うステージング環境
  - `prod`: ストアで実際に公開するアプリで使用する本番環境
- Flutter 3.7 以上を想定
  - 当記事の手法はFlutter 3.7 以上で利用可能です。
- 対応OS
  - ✅ Android
  - ✅ iOS
  - ⬜️ Web（ToDo）

## サンプルコード（リポジトリ）
- 環境分けを適用したアプリプロジェクトのサンプルはこちら↓

https://github.com/altive/flutter_app_template/tree/main/packages/flutter_app

## 当記事で例として使用する環境分け情報
- アプリ名
  - dev: `FAT dev`
  - stg: `FAT stg`
  - prod: `FAT`
- アプリID (Bundle ID, Package name)
  - dev: `jp.co.altive.fat.dev`
  - stg: `jp.co.altive.fat.stg`
  - prod: `jp.co.altive.fat`

dev, stg環境の場合はアプリ名とアプリIDに、それぞれ環境名を足したもので表現したいと思います。
もちろん、「アプリ名の方は（dev）のように括弧で表現する」といったことも自由なので、適宜ご調整ください。

## 必要な下準備
- まだの方は、環境分けを行いたいFlutterアプリを作成、Clone等してください。
- Firebaseを使用している場合は、 `flutterfire-cli` を使用したり、環境の数分のFirebaseプロジェクトを作成して、iOS用の `GoogleService-Info.plist` と `google-services.json` をダウンロードしておきましょう。

:::message
Firebaseを使用するための初期化処理などは割愛しますので、公式ドキュメントを参照ください↓
:::
https://firebase.flutter.dev/docs/overview

# 環境（Flavor）を定義する

## 環境別の定義をまとめたJSONファイルを作成する

まずは何はともあれ、環境名や、環境ごとのアプリ名など、必要な項目を好きな名前で定義しましょう👌

ファイルの作成場所とファイル名は自由です。
例として `dart_defines` ディレクトリを作成して `dev.json` という名前で作成するとします。

```json:dart_defines/dev.json
{
    "flavor": "dev",
    "appName": "FAT dev",
    "appIdSuffix": ".dev",
}
```

同じように環境の種類分JSONファイルを作成しましょう。

```json:dart_defines/stg.json
{
    "flavor": "stg",
    "appName": "FAT stg",
    "appIdSuffix": ".stg",
}
```

```json:dart_defines/prod.json
{
    "flavor": "prod",
    "appName": "FAT",
    "appIdSuffix": "",
}
```

## アプリビルド時にコマンドで環境を指定する
アプリ起動(run)やビルド(build)時に環境を分けるために、 `--dart-define-from-file` というオプションを指定します。
下記の例では、 `dev` （開発環境）を指定しています。

```shell
# アプリ起動
flutter run --dart-define-from-file=dart_defines/dev.json
# アプリビルド
flutter build ios --dart-define-from-file=dart_defines/dev.json
```

VS Code や Android Studio を使用している場合は、ボタンやショートカットからアプリ起動することも多いと思います。
その場合は `--dart-define-from-file=dart_defines/dev.json` の設定を追加してください。

:::message
`--flavor dev` の指定は不要です。
:::

### VS Code の設定例
VS Code では、 `launch.json` で起動コマンドを編集できます。
すでにファイルがある場合は、既存ファイルを編集し、ない場合は `.vscode` ディレクトリを作成し、 `launch.json` を追加しましょう。

以下のように `args` にて `--dart-define-from-file` を指定可能です。
dev, stg, prod 3環境分用意した例です。

:::details launch.json
```dart
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug dev",
            "request": "launch",
            "type": "dart",
            "flutterMode": "debug",
            "args": [
                "--dart-define-from-file=dart_defines/dev.json"
            ]
        },
        {
            "name": "Debug stg",
            "request": "launch",
            "type": "dart",
            "flutterMode": "debug",
            "args": [
                "--dart-define-from-file=dart_defines/stg.json"
            ]
        },
        {
            "name": "Debug prod",
            "request": "launch",
            "type": "dart",
            "flutterMode": "debug",
            "args": [
                "--dart-define-from-file=dart_defines/prod.json"
            ]
        }
    ]
}
```
:::

### Android Studio の設定例
Daigoさんが書いてくださいました🙌
Android Studioでの設定方法は↓を確認してください👍
https://zenn.dev/mamushi/scraps/13c0232c88227e

## FlutterアプリでFlavorを取得して使いたい場合にすること
まず、Flutterアプリ側でFlavorを取得したい場合の設定を解説します。
例えば、環境ごとに何らかの分岐を行ったりしたい時に使用するかと思います。

また、起動/ビルドコマンドで指定した `dart-define-from-file` がきちんと反映されているかも確認することができるので試しておきましょう。
JSONファイル内で定義したプロパティは `fromEnvironment` メソッドで個別に取得することが可能です👍

- 文字列： `String.fromEnvironment(name)`
- 数値： `int.fromEnvironment(name)`
- 真偽値： `bool.fromEnvironment(name)`

:::message
アプリ名やアプリID、Firebaseの環境だけ変わればよい場合は、読み飛ばしてAndroidの設定に移っても大丈夫です。
:::

```dart
// `--dart-define-from-file=dart_defines/dev.json` に定義した `flavor` プロパティを取得したい場合
const flavorName = String.fromEnvironment('flavor');
print(flavor) // 'dev'
```

:::message
存在しないプロパティを `fromEnvironment` で取得しようとした場合は、`""` や `0` , `false` が入ってきます。
任意で別のデフォルト値を指定することもできますが、 `defaultValue` には頼らずに `dart-define-from-file` を指定し忘れないようにする方がオススメです。
:::

## アプリIconを環境別に分ける場合に必要なこと
※ Icon画像を環境別に分けない場合は読み飛ばしてください。

![](/images/separating-environments-in-flutter/home-flavor-icons.png =350x)

### 環境別のアイコン画像をフォルダに配置します
※　画像は適宜用意してください
`assets/launcher_icon/icon-dev.png`
`assets/launcher_icon/icon-stg.png`
`assets/launcher_icon/icon-prod.png`

### flutter_launcher_iconsパッケージをインストール

```shell
flutter pub add flutter_launcher_icons --dev
```
もしくは pubspec.yaml に追記して `pub get` します。
```yaml
dev_dependencies:
  flutter_launcher_icons: ^0.11.0 # インストール時点での最新版を推奨
```

### Flavorごとの設定ファイルを作成
- `flutter_launcher_icons-dev.yaml`
- `flutter_launcher_icons-stg.yaml`
- `flutter_launcher_icons-prod.yaml`

```yaml
# flutter_launcher_icons-dev.yaml

flutter_icons:
  android: true
  ios: true
  image_path: "assets/launcher_icon/icon-dev.png"
```

### pubspec.yaml にも設定を追記
```yaml
# pubspec.yaml

flutter_icons:
  android: true
  ios: true
```

### アイコン画像の書き出し実行
```shell
flutter pub run flutter_launcher_icons:main
```
`✓ Successfully generated launcher icons for flavors` と表示されれば成功です。

iOSでは `ios/Runner/Assets.xcassets/AppIcon-{Flavor名}.appiconset/`
Androidでは `android/app/src/{Flavor名}/mipmap**/launcder.png`

に出力されているはずです👍

## Androidアプリに環境を反映させるために必要なこと
定義した `Dart-define` を反映させるために `build.gradle` と `AndroidManifest.xml` ファイルを編集していきましょう。

### `defaultConfig` でアプリIDとアプリ名を設定

```diff groovy
 // android/app/build.gradle

 defaultConfig {
     minSdkVersion flutter.minSdkVersion
     targetSdkVersion flutter.targetSdkVersion
     versionCode flutterVersionCode.toInteger()
     versionName flutterVersionName
+     applicationIdSuffix appIdSuffix
+     resValue "string", "app_name", appName
 }
```

- `applicationIdSuffix` に `{環境}.json` で定義した `appIdSuffix` を指定
- アプリ名として使う変数 `string/app_name` のに `appName` を指定

### defaultConfigで設定したアプリ名を使用するようにする
`android/app/src/main/AndroidManifest.xml` を編集して、 `build.gradle` で設定したアプリ名を使用するようにします。

```diff xml
 <!-- AndroidManifest.xml -->
- android:label="flutter_app_template"
+ android:label="@string/app_name"
```
これで環境によってアプリ名が変わるようになりました。

### アイコンを切り替えるためのタスクを追加する
`flutter_launcher_icons` パッケージを使って環境ごとのアイコンを作成しておきます。
`src/{環境名}/res` ディレクトリに複数の `mipmap-xxx` ディレクトリがあり、 `ic_launcher.png` が生成されています。

環境により、これらのアイコンを切り替えたいため、 `build.gradle` に以下を追記します。

```diff groovy
// android/app/build.gradle

task copySources(type: Copy) {
    from "src/$flavor/res"
    into 'src/main/res'
}
tasks.whenTaskAdded { task ->
    task.dependsOn copySources
}
```

やっていることは、ビルド時に{flavor}ディレクトリの `res` を `src/main/res` にコピーしています。

### `.gitignore` ファイルに追加
`android/app/src/main/res/mipmap*/` は、ビルド時に生成されてFLAVORにより内容が変わるので、Git管理対象外にします。
```diff gitignore
+ **/android/app/src/main/res/mipmap*/
```

### Firebase対応 (Android)
:::message
Firebaseを使用していない場合や、環境ごとにFirebaseプロジェクトを分けない場合は読み飛ばしてください。
:::

#### `android/app/src` に各環境（Flavor）と同名のディレクトリ（フォルダ）を作成。
- `android/app/src/dev/` , `android/app/src/stg/` , `android/app/src/prod/`

各Firebaseプロジェクトに作ったAndroidアプリ用の `google-services.json` をそれぞれのディレクトリに配置します。

#### `android/app/build.gradle` に追記
環境別ディレクトリに配置した `google-services.json` を `android/app` にコピーするタスクを定義しています。
```groovy
// android/app/build.gradle

task selectGoogleServicesJson(type: Copy) {
    from "src/$flavor/google-services.json"
    into './'
}

tasks.whenTaskAdded { task ->
    if (task.name == 'processDebugGoogleServices' || task.name == 'processReleaseGoogleServices') {
        task.dependsOn selectGoogleServicesJson
    }
}
```

:::message
アプリアイコンを切り替える処理を書いた場合は `tasks.whenTaskAdded` がすでに存在しているので、同じ `tasks.whenTaskAdded` を使用してください。
:::

#### `.gitignore` ファイルに追加
`android/app/google-services.json` は、ビルド時に生成されてFLAVORにより内容が変わるので、Git管理対象外にしましょう。
```diff gitignore
+ **/android/app/google-services.json
```

## iOSアプリに環境を反映させるために必要なこと

### アプリ表示名を環境によって変える
`ios/Runner/Info.plist` を編集します。

アプリ名に使われる `CFBundleDisplayName` と `CFBundleName` にアプリ名を指定します。

```diff plist
 <key>CFBundleName</key>
- <string>flutter_app_template</string>
+ <string>$(appName)</string>
+ <key>CFBundleDisplayName</key>
+ <string>$(appName)</string>
```

:::message
ここでは同じ `$(appName)` を設定していますが、もちろん、２つの名前別々なものを指定しても良いです。

`CFBundleDisplayName` がiOS端末のホームアイコンに表示されるアプリ名となります。
`CFBundleName` はIPAを作成した時の名前になります→ `${CFBundleName}.ipa` 。
※ この時日本語部分は省略されます。例： `sampleアプリ` → `sample.ipa`
:::

環境ごとに以下のようなアプリ名が表示されるようになりました👍
- dev: `FAT dev`
- stg: `FAT stg`
- prod: `FAT`

のようになります。

### Bundle IDを環境によって変える
`ios/Runner.xcodeproj/project.pbxproj` を `PRODUCT_BUNDLE_IDENTIFIER` で検索するか、
Xcode > Runner > TARGETS Runner > Build Settings の `Product Bundle Identifier` を表示して、
`$(appIdSuffix)` を末尾に追加します。

![](/images/separating-environments-in-flutter/add-bundle-id-suffix-from-build-settings.png)
*画像で使っている　`FLAVOR_SUFFIX` は古い名前です。 `appIdSuffix` に変更しました*

忘れずに Debug, Profile, Release すべてに接尾辞を追加して共通の値になるように注意しましょう。

```diff pbxproj
# ios/runner.xcodeproj/project.pbxproj
- PRODUCT_BUNDLE_IDENTIFIER = "jp.co.altive.fat";
+ PRODUCT_BUNDLE_IDENTIFIER = "jp.co.altive.fat$(appIdSuffix)";
```

これで、環境ごとにアプリのBundle IDが変わるようになりました👏

### Appアイコンを環境によって変える
`ios/Runner.xcodeproj/project.pbxproj` を `ASSETCATALOG_COMPILER_APPICON_NAME` で検索するか、
Xcode > Runner > TARGETS Runner > Build Settings の `Primary App Icon Set Name` を表示して、
`AppIcon` と指定されている値を `AppIcon-$(flavor)` に変更します。

![](/images/separating-environments-in-flutter/xcode-primary-app-icon-set-name.png)

忘れずに Debug, Profile, Release すべて共通の値になるようにしましょう。

```diff pbxproj
# ios/runner.xcodeproj/project.pbxproj
- ASSETCATALOG_COMPILER_APPICON_NAME = "AppIcon";
+ ASSETCATALOG_COMPILER_APPICON_NAME = "AppIcon-$(flavor)";
```

これで、環境ごとにアプリのアイコンが変わるようになりました👏

### Firebase対応 (iOS)
Firebaseを使用していてかつ環境ごとにプロジェクトを分ける場合は、`GoogleService-Info.plist` を環境ごとに使い分ける必要があります。

:::message
Firebaseを使用していない場合や、環境ごとにFirebaseプロジェクトを分けない場合は読み飛ばしてください。
:::

#### `ios` ディレクトリに各環境（Flavor）と同名のディレクトリ（フォルダ）を作成。
- `ios/dev/` , `ios/stg/` , `ios/prod/`

各Firebaseプロジェクトに作ったiOSアプリ用の `GoogleService-Info.plist` をそれぞれのディレクトリに配置します。

#### Select GoogleService-Info.plist
ビルド時に、環境に対応した `GoogleService-Info.plist` を使用できるようにするスクリプトを追加します。
1. `Build phases` -> `New run script` を選択して新しい `Run Script` を追加
1. 名前をわかりやすいように `Select GoogleService-Info.plist` に変更
1. スクリプトを記述
1. `Output Files` に `$(SRCROOT)/GoogleService-Info.plist` を追加
1. 既存Scriptである `Copy Bundle Resources` より上に移動

```shell
\cp -f ${SRCROOT}/${flavor}/GoogleService-Info.plist ${SRCROOT}/GoogleService-Info.plist
```

![](/images/separating-environments-in-flutter/select-googleservice-info-plist-to-build-phases.png)

#### GoogleService-Info.plistへの参照を追加する
（[@akaboshinit](https://zenn.dev/link/comments/82fafd879f7b7b) さん、追加情報ありがとうございます！） <!-- cspell:disable-line -->

実際には、スクリプトによってコピーされた `GoogleService-Info.plist` が使用されることになるのですが、このままでは `${SRCROOT}/GoogleService-Info.plist` ファイルへの参照がなくXcodeで認識できません。
Xcode上でRunnerディレクトリへ `GoogleService-Info.plist` ファイル（どの環境のものでも良い）をドラッグ＆ドロップで追加してファイルと参照を追加しましょう。

![](/images/separating-environments-in-flutter/xcode-google-service-info-plist-adding.png)
*Downloadフォルダ等から、Runnerフォルダ内へドラッグ＆ドロップ (`Copy items if needed` にチェック）*

![](/images/separating-environments-in-flutter/xcode-google-service-info-plist-added.png)
*Runnerフォルダ内に追加されました。*

このファイルはビルド時に任意の環境用の `GoogleService-Info.plist` に置き換わります🙆‍♂️

#### `.gitignore` ファイルに追加
`ios/GoogleService-Info.plist` は、ビルド時に生成されてFLAVORにより内容が変わるので、Git管理対象外にします。
```diff gitignore
+ **/ios/GoogleService-Info.plist
```

# さいごに

## Flutterアプリを起動して、Flavorがきちんと伝わっているか確かめる
`--dart-define-from-file` がきちんとネイティブに伝わり、アプリ名やBundle IDがFlavorごとに変更されていることを手軽に確かめるためには、 `package_info_plus` パッケージを使用します。

https://pub.dev/packages/package_info_plus

`PackageInfo` の下記メソッドを使用して確認できます。

- `.appName`
  - iOS: アプリ名 (`CFBundleDisplayName`)
  - Android: `android:label="@string/app_name"`
- `.packageName`
  - iOS: Bundle ID (`CFBundleIdentifier`)
  - Android: `applicationId`

## 宣伝
Riverpod の実践入門本を書きました👍
https://zenn.dev/riscait/books/flutter-riverpod-practical-introduction

## 参考記事
https://itnext.io/flutter-1-17-no-more-flavors-no-more-ios-schemas-command-argument-that-solves-everything-8b145ed4285d
https://medium.com/flutter-jp/flavor-b952f2d05b5d
https://qiita.com/hiromasa-fun/items/c79c99535f6f1db2a6a9
https://zenn.dev/kingu/articles/9192b91062841b8e0bba
https://medium.com/flutter-jp/icon-935d637d2da0
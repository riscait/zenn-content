---
title: "【Flutter 3.19対応】Dart-define-from-fileを使って開発環境と本番環境を分ける"
emoji: "🔨"
type: "tech"
topics: [flutter, flavor, dart, firebase]
published: true
publication_name: "altiveinc"
# cSpell:ignore kurogoma4d
---
:::message
Flutter 3.16 までは、 `dart-define-from-file` で定義した環境変数が、iOSやAndroidで使用できていました。
ところが、この動作は意図せず `dart-define` の動作とも一致しないため、 Flutter 3.19 (stable) 以降で無効になりました。

Flutter 3.19で動作確認しつつ、必要な対応を弊社のテンプレートリポジトリに追加しました。
追加対応を行ったプルリクエスト（コミット）はこちらです。
https://github.com/altive/flutter_app_template/pull/321/commits

あわせて、こちらの記事も、3.19以降に必要な対応を含めた内容に更新しました👌
:::

| 開発環境                                                                            | ステージング環境                                                                    | 本番環境                                                                             |
| ----------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| ![](/images/separating-environments-in-flutter/flavor-dev-ios-screenshot.png =200x) | ![](/images/separating-environments-in-flutter/flavor-stg-ios-screenshot.png =200x) | ![](/images/separating-environments-in-flutter/flavor-prod-ios-screenshot.png =200x) |

# はじめに

こんにちは、Flutterでのアプリ開発をメインとしている「[Altive株式会社](https://altive.co.jp)」の村松龍之介（[@riscait](https://x.com/riscait)）です！

![](/images/ProfileBanner_Muramatsu.jpg)

Flutterにおいて、その環境分けの方法はいくつかありますが、今回は `Dart-defines-from-file` を使用して実現する方法をご紹介します。

:::message
`dart-define` を使用した旧版もアーカイブとして別記事に残しました。必要であれば参照してください。
https://zenn.dev/altiveinc/articles/separating-environments-in-flutter-old-edition
:::

# `dart-define-from-file` のメリット

Flavor用のパッケージを使う場合等と比べて、**ビルドコマンドがシンプルになる**ことや、**自動生成ファイルが少なく、取り回しがしやすい**ことが大きな利点です。

- `main.dart` を環境ごとに分ける必要がない
- iOSの `scheme` や `Configuration` を作成する必要がない
- パッケージの導入が不要（アイコン生成には使用します）

# 前提
- この記事では以下の3環境に分けていきます
（あくまで一例なので、数や名前は適宜変更してください）
  - `dev`: ローカルの開発環境
  - `stg`: 本番環境に似せた環境を用意し、検証を行う環境
  - `prod`: 実際に公開するアプリで使用する本番環境
- Flutter 3.7 以上を想定
  - 当記事の手法はFlutter 3.7 以上で利用可能です。
- 対応OS
  - ✅ Android
  - ✅ iOS
  - ✅ macOS
  - ✅ Web

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

## Firebaseを使用する場合
- Firebaseを使用している場合は、 `flutterfire-cli` を使用したり、FirebaseのWebコンソールから環境数分のプロジェクトとアプリを作成して、iOS用の `GoogleService-Info.plist` と `google-services.json` をダウンロードしておきましょう。

:::message
Firebaseを使用するための初期化処理などは割愛しますので、公式ドキュメントを参照ください↓
:::
https://firebase.google.com/docs/flutter/setup?hl=ja

# dart-define-from-fileを使う

それでは、 `dart-define-from-file` を使った環境分けを行っていきましょう。

## 環境の数だけ定義をまとめたファイルを作成する（JSON or ENV）

まずは環境名や、環境ごとのアプリ名など、必要な項目を定義するファイルを作成しましょう👌

ファイルは、 `.json` か `.env` が使えます。
ファイルの配置場所とファイル名は自由です。

今回は例として `dart_defines` ディレクトリを作成して `dev.env` という名前で作成するとします。

```env:dart_defines/dev.env
flavor="dev"
appName="FAT dev"
appId="jp.co.altive.fat.dev"
googleReversedClientId="com.googleusercontent.apps.0123456789-xxxxxxxxxxxxxxxx"
```

同じように環境数分のファイルを作成しましょう。

```env:dart_defines/stg.env
flavor="stg"
appName="FAT stg"
appId="jp.co.altive.fat.stg"
googleReversedClientId="com.googleusercontent.apps.0123456789-xxxxxxxxxxxxxxxx"
```

```env:dart_defines/prod.env
flavor="prod"
appName="FAT"
appId="jp.co.altive.fat"
googleReversedClientId="com.googleusercontent.apps.0123456789-xxxxxxxxxxxxxxxx"
```

:::message
例の `googleReversedClientId` は、iOSのinfo.plistに記載する `CFBundleURLSchemes` に使用します。
もちろん、不要なら設定する必要はありません。
:::

## アプリビルド時にコマンドで環境を指定する
アプリ起動(run)やビルド(build)時に環境を分けるために、 `--dart-define-from-file` というオプションを指定します。
下記の例では、 `dev` （開発環境）を指定しています。

```shell
# アプリ起動コマンドの例
flutter run --dart-define-from-file=dart_defines/dev.env
# アプリビルドコマンドの例
flutter build ios --dart-define-from-file=dart_defines/dev.env
```

:::message
ややこしいですが、この記事では `flavor` 機能は使用していないため、 `--flavor dev` の指定は不要です。
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
                "--dart-define-from-file=dart_defines/dev.env"
            ]
        },
        {
            "name": "Debug stg",
            "request": "launch",
            "type": "dart",
            "flutterMode": "debug",
            "args": [
                "--dart-define-from-file=dart_defines/stg.env"
            ]
        },
        {
            "name": "Debug prod",
            "request": "launch",
            "type": "dart",
            "flutterMode": "debug",
            "args": [
                "--dart-define-from-file=dart_defines/prod.env"
            ]
        }
    ]
}
```
:::

### Android Studio の設定例

`Add Configuration` または `Edit Configurations` からFlutterの起動構成を設定しましょう。

詳しくは、Daigoさんが書いてくださった記事を参考にしてください👍
https://zenn.dev/mamushi/scraps/13c0232c88227e

## FlutterアプリでDart defineで設定した情報を取得する

起動/ビルドコマンドで指定した `dart-define-from-file` がきちんと反映されているかも確認することができるので試しておきましょう。

env（またはjson）ファイル内で定義したプロパティは `fromEnvironment` メソッドで個別に取得することが可能です👍

- 文字列： `String.fromEnvironment(name)`
- 数値： `int.fromEnvironment(name)`
- 真偽値： `bool.fromEnvironment(name)`

```dart
// `--dart-define-from-file=dart_defines/dev.env` に定義した `flavor` プロパティを取得したい場合
const flavor = String.fromEnvironment('flavor');
print(flavor) // 'dev'
```

:::message
存在しないプロパティを `fromEnvironment` で取得しようとした場合は、`""` や `0` , `false` が入ってきます。
任意で別のデフォルト値を指定することもできますが、 `defaultValue` には頼らずに `dart-define-from-file` を指定し忘れないようにする方がオススメです。
:::

## アプリIconを環境別に分ける
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

```yaml:pubspec.yaml
dev_dependencies:
  flutter_launcher_icons: ^0.13.1 # インストール時点での最新版を推奨
```

### Flavorごとの設定ファイルを作成
- `flutter_launcher_icons-dev.yaml`
- `flutter_launcher_icons-stg.yaml`
- `flutter_launcher_icons-prod.yaml`

```yaml:flutter_launcher_icons-dev.yaml
flutter_launcher_icons:
  android: true
  ios: true
  image_path: "assets/launcher_icon/icon-dev.png"
```

### アイコン画像の書き出し実行
```shell
dart run flutter_launcher_icons
```
`✓ Successfully generated launcher icons for flavors` と表示されれば成功です。

iOSでは `ios/Runner/Assets.xcassets/AppIcon-{flavor}.appiconset/`
Androidでは `android/app/src/{flavor}/mipmap**/ic_launcher.png`

に出力されているはずです👍

## Androidアプリに必要な対応
定義した `Dart-define` を反映させるために `build.gradle` と `AndroidManifest.xml` ファイルを編集していきましょう。

### build.gradle を編集して dart-define を受け取る
Dart-define をデコードして受け取ります。

```groovy:android/app/build.gradle
// dart-define を入れる変数を宣言しています。
def dartDefines = [:];
if (project.hasProperty('dart-defines')) {
    // カンマ区切りかつBase64でエンコードされている dart-defines をデコードして変数に格納します。
    dartDefines = dartDefines + project.property('dart-defines')
        .split(',')
        .collectEntries { entry ->
            def pair = new String(entry.decodeBase64(), 'UTF-8').split('=')
            pair.length == 2 ? [(pair.first()): pair.last()] : [:]
        }
}
```

:::message
valueが空のDart-defineがあった場合、valueがkeyの文字列になってしまう問題に対応しました。@k9i さん[コメント](https://zenn.dev/link/comments/7c3e53a32b687b)でのご報告ありがとうございました！
:::

### `defaultConfig` でアプリIDとアプリ名を設定

```diff groovy:android/app/build.gradle
 defaultConfig {
     minSdkVersion flutter.minSdkVersion
     targetSdkVersion flutter.targetSdkVersion
     versionCode flutterVersionCode.toInteger()
     versionName flutterVersionName
+     applicationId "${dartDefines.appId}"
+     resValue "string", "app_name", "${dartDefines.appName}"
 }
```

- `applicationId` でアプリ名を指定
- `resValue` を使用してアプリ名を指定

### defaultConfigで設定したアプリ名を使用するようにする
`android/app/src/main/AndroidManifest.xml` を編集して、 `build.gradle` で設定したアプリ名を使用するようにします。

```diff xml:AndroidManifest.xml
- android:label="flutter_app_template"
+ android:label="@string/app_name"
```
これで環境によってアプリ名が変わるようになりました。

### アイコンを切り替えるためのタスクを追加する

`flutter_launcher_icons` パッケージを使って環境ごとのアイコンを作成しておきます。
`src/{flavor}/res` ディレクトリに複数の `mipmap-xxx` ディレクトリがあり、 `ic_launcher.png` が生成されています。

環境により、これらのアイコンを切り替えたいため、 `build.gradle` に以下を追記します。

```diff groovy
// android/app/build.gradle

task copySources(type: Copy) {
    from "src/${dartDefines.flavor}/res"
    into 'src/main/res'
}
```

ビルド時に環境ごとのディレクトリ内の `res` を `src/main/res` にコピーしています。

:::details 以前のandroid.sourceSets.mainで複数のフォルダをres.srcDirsに指定する方法
※ `src/main/res` の中身を削除しても、 `flutter create` を実行すると、再度 `src/main/res` 内にアイコンファイル等が生成されてしまい、起動時にファイルの重複エラーが発生してしまうため、こちらの方法は使わないことにしました。

---

```diff groovy:android/app/build.gradle
android {
    compileSdkVersion flutter.compileSdkVersion

    sourceSets {
        main {
            java.srcDirs += 'src/main/kotlin'
            res.srcDirs = ['src/main/res', "src/${dartDefines.flavor}/res"]
        }
    }
...
```

`res.srcDirs` を指定し、 `src/main/res` と `src/{flavor}/res` をマージしています。
この時、この2つのディレクトリに同じファイルが存在するとエラーになるので、2箇所に同じファイルが存在しないようにしましょう。

@kurogoma4d さん、[コメント](https://zenn.dev/link/comments/e0ae923d2517ce)ありがとうございました！
:::

### Firebase対応 (Android)
:::message
Firebaseを使用していない場合や、環境ごとにFirebaseプロジェクトを分けない場合は読み飛ばしてください。
:::

#### `android/app/src` に各環境（Flavor）と同名のディレクトリ（フォルダ）を作成。
- `android/app/src/dev/` , `android/app/src/stg/` , `android/app/src/prod/`

各Firebaseプロジェクトに作ったAndroidアプリ用の `google-services.json` をそれぞれのディレクトリに配置します。

#### `android/app/build.gradle` に追記
環境別ディレクトリに配置した `google-services.json` を `android/app` にコピーするタスクを定義しています。
```groovy:android/app/build.gradle
task selectGoogleServicesJson(type: Copy) {
    from "src/${dartDefines.flavor}/google-services.json"
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

#### `.gitignore` ファイルに `google-services.json` を追加
`android/app/google-services.json` は、ビルド時に生成されて環境ごとに上書きされるので、Git管理対象外にしましょう。
```diff gitignore:.gitignore
+ **/android/app/google-services.json
```

## iOSアプリに必要な対応

### Dart define を受け取る Pre Actionを追加
ビルド時に指定した `--dart-define` をiOSで受け取るために、ビルド直前に実行されるスクリプトを追加する必要があります。

もちろん、スクリプトをXcode上から直接書き込んでも良いですが、
- `Runner.xcscheme` に改行がない状態で書き込まれるので差分が見にくい
- コメント等で日本語を書くとエンコードされて読めない

というデメリットがあり、新規ファイルを作成して使うことで、好きなエディタ（VS Codeなど）のハイライト機能等を利用しながら編集できる利点もあります。

### スクリプトファイル保存
`ios/scripts/extract_dart_defines.sh` というパスとファイル名で以下の内容の `sh` ファイルを保存します。

```shell:ios/scripts/extract_dart_defines.sh
#!/bin/sh

# Dart defineを書き出すファイルパスを指定します。
# ここでは `Dart-Defines.xcconfig` というファイル名で作成することにします。
OUTPUT_FILE="${SRCROOT}/Flutter/Dart-Defines.xcconfig"
# Dart defineの中身を変更した時に古いプロパティが残らないように、初めにファイルを空にしています。
: > $OUTPUT_FILE

# この関数でDart defineをデコードします。
function decode_url() { echo "${*}" | base64 --decode; }

IFS=',' read -r -a define_items <<<"$DART_DEFINES"

for index in "${!define_items[@]}"
do
    item=$(decode_url "${define_items[$index]}")
    # Dartの定義にはFlutter側で自動定義された項目も含まれます。
    # しかし、それらの定義を書き出してしまうとエラーによりビルドができなくなるので、
    # flutterやFLUTTERで始まる項目は出力しないようにしています。
    lowercase_item=$(echo "$item" | tr '[:upper:]' '[:lower:]')
    if [[ $lowercase_item != flutter* ]]; then
        echo "$item" >> "$OUTPUT_FILE"
    fi
done
```

:::message
上記の例では、flutter標準のDart defineを含めないように除外しています。
もしくは、`flavor_name=dev`, `flavor_appId=jp.co.altive.fat.dev` のように、
プレフィクスをつけて定義し、それらだけを抽出しても良いかもしれません。
:::

### XcodeのBuild Pre-actions に作成したスクリプトを登録する
1. Xcodeで、Product > Scheme > Edit Scheme (⌘ ⇧ <)を開きます
1. XcodeのScheme (Runner) をクリックして `Edit scheme` -> Build を展開して `Pre-actions` を選択します
1. 「＋」ボンタンを押して「New Run Script Action」を選択します。
1. 「Provide build settings from」は `Runner` を選択します。
1. 先ほど保存したファイルのパスである `${SRCROOT}/scripts/extract_dart_defines.sh` を書き込みます。

![](/images/separating-environments-in-flutter/build-pre-action-for-dart-defines.png)

### スクリプトファイルに実行権限を与える
そのままビルドしてもスクリプトファイルが実行されません。
以下のコマンドで実行限限を与えておきましょう。
```shell
chmod 755 ios/scripts/extract_dart_defines.sh
```
https://qiita.com/shisama/items/5f4c4fa768642aad9e06

### 各種xcconfigファイルで `Dart-Defines.xcconfig` をインポート
前項のスクリプトで生成される `Dart-Defines.xcconfig` がDebug, Releaseビルド両方で使われるように、既存の `*.xcconfig` ファイルでインクルードします。

```diff:ios/Flutter/Debug.xcconfig
 #include? "Pods/Target Support Files/Pods-Runner/Pods-Runner.debug.xcconfig"
 #include "Generated.xcconfig"
+ #include "Dart-Defines.xcconfig"
```

```diff:ios/Flutter/release.xcconfig
 #include? "Pods/Target Support Files/Pods-Runner/Pods-Runner.release.xcconfig"
 #include "Generated.xcconfig"
+ #include "Dart-Defines.xcconfig"
```

### `.gitignore` ファイルに `Dart-Defines.xcconfig` を追加
`Dart-Defines.xcconfig` は、ビルド時に生成され、環境により内容が上書きされるので、Git管理対象外にします。
```diff:.gitignore
+ **/ios/Flutter/Dart-Defines.xcconfig
```

:::message
iOSでは `Generated.xcconfig`, macOSでは `Flutter-Generated.xcconfig` が生成され、これらのファイルに書き込めば手間が減りますが、今回は使用しませんでした。
https://x.com/riscait/status/1751541002417086467?s=20
:::

### アプリ表示名を環境によって変える
`ios/Runner/Info.plist` を編集します。

アプリ名に使われる `CFBundleDisplayName` と `CFBundleName` にアプリ名を指定します。

```diff plist:ios/Runner/Info.plist
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

### Bundle IDを環境によって変える
`ios/Runner.xcodeproj/project.pbxproj` を `PRODUCT_BUNDLE_IDENTIFIER` で検索するか、
Xcode > Runner > TARGETS Runner > Build Settings の `Product Bundle Identifier` を表示して、
`$(appId)` に変更します。

![](/images/separating-environments-in-flutter/xcode-bundle-id-in-build-settings.png)

Debug, Profile, Release すべてのBundle IDに $(appId) が設定されていることを確認してください。

```diff pbxproj:ios/runner.xcodeproj/project.pbxproj
- PRODUCT_BUNDLE_IDENTIFIER = "jp.co.altive.fat";
+ PRODUCT_BUNDLE_IDENTIFIER = "$(appId)";
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
1. `Output Files` に `$(SRCROOT)/Runner/GoogleService-Info.plist` を追加
1. 既存Scriptである `Copy Bundle Resources` より上に移動

```shell:project.pbxproj
\cp -f ${SRCROOT}/${flavor}/GoogleService-Info.plist ${SRCROOT}/Runner/GoogleService-Info.plist
```

![](/images/separating-environments-in-flutter/select-googleservice-info-plist-to-build-phases.png)

#### `.gitignore` ファイルに `GoogleService-Info.plist` 追加
`ios/Runner/GoogleService-Info.plist` は、ビルド時に生成されて環境により内容が変わるので、Git管理対象外にします。
```diff gitignore:.gitignore
+ **/ios/Runner/GoogleService-Info.plist
```

## macOS対応
macOSの対応は、ほとんどiOSと同じです。
`ios` ディレクトリを `macos` ディレクトリに読み替えて同じように実行してください。

一部、異なる点を以下の通りです。

- iOSの `Debug.xcconfig` は、macOSでは `Flutter-Debug.xcconfig` という名前です
- iOSの `Release.xcconfig` は、macOSでは `Flutter-Release.xcconfig` という名前です

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

# さいごに
最後まで読んでいただきありがとうございました😊

## 宣伝

Altive株式会社では、Flutterアプリの開発・運営を承っております。
お気軽にお問い合わせください🫡 
https://altive.co.jp/contact

---

Riverpod の実践入門本を公開中です📘
https://zenn.dev/riscait/books/flutter-riverpod-practical-introduction

## 参考記事
https://itnext.io/flutter-1-17-no-more-flavors-no-more-ios-schemas-command-argument-that-solves-everything-8b145ed4285d
https://medium.com/flutter-jp/flavor-b952f2d05b5d
https://qiita.com/hiromasa-fun/items/c79c99535f6f1db2a6a9
https://zenn.dev/kingu/articles/9192b91062841b8e0bba
https://medium.com/flutter-jp/icon-935d637d2da0
https://neos21.net/blog/2020/09/17-02.html

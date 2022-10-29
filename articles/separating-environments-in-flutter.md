---
title: "FlutterでDart-defineのみを使って開発環境と本番環境を分ける"
emoji: "🔨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [flutter, flavor]
published: true
publication_name: "altiveinc"
---
:::message
2021年11月19日
- Android Studioでの設定例についてリンク追加（Daigoさんありがとうございます！）

2021年10月5日
- iOSでのAppIcon対応（Build settings）に追記漏れがあったため追記しました！（ツルオカさんありがとうございます！）

2021年8月29日
- AppIcon(launcher_icon) の切り替えに対応しました👏
- Flavorごとにxcconfigを作成する方式に変えて、シェルスクリプトを軽量化しました。
:::

| 開発環境 | ステージング環境 | 本番環境 |
| ---- | ---- | ---- |
| ![](/images/separating-environments-in-flutter/flavor-dev-ios-screenshot.png =200x) | ![](/images/separating-environments-in-flutter/flavor-stg-ios-screenshot.png =200x) | ![](/images/separating-environments-in-flutter/flavor-prod-ios-screenshot.png =200x) |

こんにちは、Flutterアプリ開発者の村松龍之介（[@riscait](https://twitter.com/riscait)）です😸

Flutterにおいて、その環境分けの方法はいくつかありますが、今回は `Dart-defines` 1つのみを使用して実現する方法で実行してみたのでご紹介します。

:::message
個人開発では無くてもなんとかなるかもしれませんが、複数人が関わったり規模の小さくないアプリを作成する場合、**開発・ステージング（テスト）・本番のように複数環境が必要**になってきます。
:::

---

https://twitter.com/riscait/status/1428150230017400834?s=21

少しXcodeやAndroidのbuild.gradleに追記したりが必要ですが、**ビルドコマンドがシンプルになる**ことや、
パッケージを使う場合と比べて**自動生成ファイルが少なく、取り回しがしやすい**ことが大きな利点です。

pub.devのパッケージを使用したり、 `--flavor` オプションを使用する方法と比べたメリットとデメリットは以下の通りです。

# この記事のメリット
- Flutterの環境ごとのビルド設定切り替え方法が分かる
- Flavor分けにパッケージの導入が不要（アイコン生成には使用します）
- `--dart-define` のみでiOS, AndroidネイティブへもFlavor（環境）を伝播できる
- `--dart-define=FLAVOR` のみで Bundle ID (Package name) 等を切り替えられる
- `main.dart` は1つのままで良い。環境ごとに分ける必要がない
- iOSの `scheme` や `Configuration` を作成する必要がない

# ちょっと面倒なところ
- Androidでは、 `build.gradle` , `AndroidManifest.xml` に追記が必要
- iOSでは、`Build Phase` と `Build Pre action` にスクリプトを追加する必要がある
- Flutter SDKのバージョンアップグレードで `Dart-defines` のエンコード方法が変わり、対応する必要が出てくる可能性がある

# 前提
- この記事では以下の3環境に分けていきます
（あくまで一例なので、数や名前は適宜変更してください）
  - dev (ローカルの開発環境)
  - stg (疑似的なDB等を用意し、テストを行うステージング環境)
  - prod (ストアで実際に公開するアプリで使用する本番環境)
- Flutter 2.2 以上を想定
  - 当記事の手法はFlutter 1.17 以上で利用可能ですが、ネイティブでDart-defineを受け取る際のデコード処理に差異があります。（詳しくは後述）
- 対応OS
  - ✅ Android
  - ✅ iOS
  - ⬜️ Web（ToDo）

# サンプルコード（リポジトリ）
- 環境分けを適用したプロジェクト（リポジトリ）はこちら↓
https://github.com/altive/flutter_app_template

## 当記事で例として使用する環境分け情報
- アプリ名
  - dev: Flutter AT.dev
  - stg: Flutter AT.stg
  - prod: Flutter AT
- アプリID (Bundle ID, Package name)
  - dev: jp.co.altive.fat.dev
  - stg: jp.co.altive.fat.stg
  - prod: jp.co.altive.fat

dev, stg環境の場合はアプリ名とアプリIDにドット繋ぎで環境名を足したいと思います。
もちろん、「アプリ名の方は（dev）のように括弧で表現する」といったことも自由なので、適宜ご調整ください。

# 必要な下準備
- まだの方は、環境分けを行いたいFlutterアプリを作成、Clone等してください。
- Firebaseを使用している場合は、環境の数分Firebaseプロジェクトを作成し、iOS用の `GoogleService-Info.plist` と `google-services.json` をダウンロードしておく。

:::message
Firebaseを使用するための初期化処理などは割愛しますので、公式ドキュメントを参照ください↓
:::
https://firebase.flutter.dev/docs/overview

## Iconを環境別に分ける場合に必要な準備
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
もしくは pubscpec.yaml に追記して `pub get` します。
```yaml
dev_dependencies:
  flutter_launcher_icons: ^0.9.2 # インストール時点での最新版を推奨
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

以上、「Flavor（環境）別アイコンの準備」は終わりです。

# ビルド時にコマンドで環境を指定する
アプリ起動(run)やビルド(build)時に環境を分けるために、 `--dart-define` というオプションを指定します。
下記の例では、 `FLAVOR=dev` （開発環境）という定義を行なっています。

```shell
# アプリ起動
flutter run --dart-define=FLAVOR=dev
# アプリビルド
flutter build ios --dart-define=FLAVOR=dev
```

VS Code や Android Studio を使用している場合は、ボタンやショートカットからアプリ起動することも多いと思います。
その場合は `--dart-define=FLAVOR=dev` の設定を追加してください。

:::message
`--flavor dev` のような指定は不要です。
:::

## VS Code の設定例
VS Code では、 `launch.json` で起動コマンドを編集できます。
すでにファイルがある場合は、既存ファイルを編集し、ない場合は `.vscode` ディレクトリを作成し、 `launch.json` を追加しましょう。

以下のように `args` にて `--dart-define` を指定可能です。
dev, stg, prod 3環境分用意した例です。

:::details launch.json
```dart
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Run dev",
            "request": "launch",
            "type": "dart",
            "args": [
                "--dart-define=FLAVOR=dev"
            ]
        },
        {
            "name": "Run stg",
            "request": "launch",
            "type": "dart",
            "args": [
                "--dart-define=FLAVOR=stg"
            ]
        },
        {
            "name": "Run prod",
            "request": "launch",
            "type": "dart",
            "args": [
                "--dart-define=FLAVOR=prod"
            ]
        }
    ]
}
```
:::

お好みで `--release` を付与したReleaseビルドバージョンも追加してください。

## Android Studio の設定例
Daigoさんが書いてくださいました🙌
Android Studioでの設定方法は↓を確認してください👍
https://zenn.dev/mamushi/scraps/13c0232c88227e

# FlutterアプリでFlavorを取得して使いたい場合
まず、Flutterアプリ側でFlavorを取得したい場合の設定を解説します。
例えば、環境ごとにAPI の Base URL や API key を変更したりするのに使うかと思います。

また、起動/ビルドコマンドで指定した `dart-define` がきちんと反映されているかも確認することができるので試しておきましょう。

:::message
アプリ名やアプリID、Firebaseの環境だけ変わればよい場合は、読み飛ばしてAndroidの設定に移っても大丈夫です。
:::

```dart
// `--dart-define=FLAVOR=dev` と指定した場合
const flavor = String.fromEnvironment('FLAVOR');
print(flavor) // dev
```

:::message
`--dart-define=FLAVOR` を指定し忘れた場合は空文字列が入ってきます。
任意で `defaultValue` を指定することもできます。
:::

# Android対応
単に `dart-define` で環境（Flavor）を指定しただけでは、 Android側に伝わりません。
`build.gradle` と `AndroidManifest.xml` ファイルを編集する必要があります。

## build.gradle を編集して dart-define を受け取る
`android/app/build.gradle` を編集します。

### ビルド時に指定した dart-define をデコードして受け取ります
```groovy
// android/app/build.gradle

// dart-define を入れる変数を宣言しています。
// `Key: Value` 形式で初期値を設定することもできます
def dartEnvironmentVariables = [:];
if (project.hasProperty('dart-defines')) {
    // カンマ区切りかつBase64でエンコードされている dart-defines をデコードして変数に格納します。
    dartEnvironmentVariables = dartEnvironmentVariables + project.property('dart-defines')
        .split(',')
        .collectEntries { entry ->
            def pair = new String(entry.decodeBase64(), 'UTF-8').split('=')
            [(pair.first()): pair.last()]
        }
}
```

:::message alert
上記はFlutter 2.2 以降に対応したデコード方法となります。
Flutter 1.17, Flutter 1.20, Flutter 2.2 で必要な処理が変更されました。
Flutter 2.2以前での書き方については参考記事を参照ください。
https://itnext.io/flutter-1-17-no-more-flavors-no-more-ios-schemas-command-argument-that-solves-everything-8b145ed4285d#5bd3
:::

### defaultConfigに `applicationIdSuffix` とアプリ名を追加します

```diff groovy
 // android/app/build.gradle

 defaultConfig {
     minSdkVersion 16
     targetSdkVersion 30
     versionCode flutterVersionCode.toInteger()
     versionName flutterVersionName
+     if (dartEnvironmentVariables.FLAVOR != 'prod') {
+         applicationIdSuffix ".${dartEnvironmentVariables.FLAVOR}"
+     }
+     resValue "string", "app_name", "FlutterAT" + 
+         (dartEnvironmentVariables.FLAVOR == 'prod' ? '' : ".${dartEnvironmentVariables.FLAVOR}")
 }
```

- 本番環境以外の `applicationIdSuffix` に `.` + `FLAVOR` を設定。
- アプリ名として使う変数 `string/app_name` の後ろにも `.` + `FLAVOR` を追加（同じく本番環境以外）

### defaultConfigで設定したアプリ名を使用するようにする
`android/app/src/main/AndroidManifest.xml` を使用して、 `build.gradle` で設定したアプリ名を使用するようにします。

```diff xml
 <!-- AndroidManifest.xml -->
- android:label="flutter_app_template"
+ android:label="@string/app_name"
```
これで環境によってアプリ名が変わるようになりました。

## アイコンを切り替えるためのタスクを追加する
`flutter_launcher_icons` パッケージを使ってFlavorごとのアイコンを作成しておきます。
`src/{flavor_name}/res` ディレクトリに複数の `mipmap-xxx` ディレクトリがあり、 `ic_launcher.png` が生成されています。

Flavorにより、これらのアイコンを切り替えたいため、 `build.gradle` に以下を追記します。

```diff groovy
// android/app/build.gradle

task copySources(type: Copy) {
    from "src/${dartEnvironmentVariables.FLAVOR}/res"
    into 'src/main/res'
}
tasks.whenTaskAdded { task ->
    task.dependsOn copySources
}
```

やっていることは、ビルド時にFlavorディレクトリの `res` を `src/main/res` にコピーしています。

:::message
build.gradle の書式については知識不足を感じています。
もっと良い記法や間違いなどありましたらご指摘いただけると幸いです。
:::

### `.gitignore` ファイルに追加
`android/app/src/main/res/mipmap*/` は、ビルド時に生成されてFLAVORにより内容が変わるので、Git管理対象外にします。
```diff gitignore
+ **/android/app/src/main/res/mipmap*/
```

## Firebase対応 (Android)
:::message
Firebaseを使用していない場合や、環境ごとにFirebaseプロジェクトを分けない場合は読み飛ばしてください。
:::

### `android/app/src` に各環境（Flavor）と同名のディレクトリ（フォルダ）を作成。
- `android/app/src/dev/` , `android/app/src/stg/` , `android/app/src/prod/`

各Firebaseプロジェクトに作ったAndroidアプリ用の `google-services.json` をそれぞれのディレクトリに配置します。

### `android/app/build.gradle` に追記
環境別ディレクトリに配置した `google-services.json` を `android/app` にコピーするタスクを定義しています。
```groovy
// android/app/build.gradle

task selectGoogleServicesJson(type: Copy) {
    from "src/${dartEnvironmentVariables.FLAVOR}/google-services.json"
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

### `.gitignore` ファイルに追加
`android/app/google-services.json` は、ビルド時に生成されてFLAVORにより内容が変わるので、Git管理対象外にします。
```diff gitignore
+ **/android/app/google-services.json
```

# iOS対応
## Flavorに対応する xcconfig ファイルを作成
Xcodeにて `File -> New -> File (⌘N)` で新規ファイルを追加します。
ファイル形式は `Configuration Settings File` を選択します。（`config` でフィルタリングするとすぐ出てきます）

![](/images/separating-environments-in-flutter/xcode-new-file-config.png)

ファイル保存場所は `ios/Flutter` 、ファイル名は `dev.xcconfig` のように `{Flavor名}.xcconfing` とします。
Flavorの数だけ新規作成しましょう。例： `stg.xcconfig` , `prod.xcconfig` ...

`{flavor}.xcconfig` の中に `FLAVOR`の値とその他使いたい変数を設定します。

![](/images/separating-environments-in-flutter/flavor-xcconfigs.png)

ここでは、 `FLAVOR=` の他に、 Bundle IDやアプリ名の末尾等に追加したい `APP_ID_SUFFIX` を定義しました。
アプリ名の末尾には別のSuffixを追加したい場合等は、適宜定義を増やして使ってください。

※ `DartDefines.xcconfig` は、後ほどシェルスクリプトで生成するファイルなので気にしないでください。

## Dart define を受け取る Pre Actionを追加
ビルド時に指定した `--dart-define` をiOSで受け取るために、ビルド直前に実行されるスクリプトを追加する必要があります。

もちろん、スクリプトをXcode上から直接書き込んでも良いですが、
- `Runner.xcscheme` に改行がない状態で書き込まれるので差分が見にくい
- コメント等で日本語を書くとエンコードされて読めない

というデメリットがあり、新規ファイルを作成して使うことで、好きなエディタ（VS Codeなど）のハイライト機能等を利用しながら編集できる利点もあります。

### スクリプトファイル保存
`ios/scripts/retrieve_dart_defines.sh` というパスとファイル名で以下の内容の `sh` ファイルを保存します。

:::details retrieve_dart_defines.sh
```shell
#!/bin/sh

# Dart-defineを書き込んだり、Flavorごとのxcconfigをincludeするファイル
OUTPUT_FILE="${SRCROOT}/Flutter/DartDefines.xcconfig"

# Flutter 2.2 以降で必要な、Dart-Definesのデコード処理 
function decode_url() { echo "${*}" | base64 --decode; }

# 最初にファイル内容をいったん空にする
: > $OUTPUT_FILE

IFS=',' read -r -a define_items <<<"$DART_DEFINES"

for index in "${!define_items[@]}"
do
    # Flutter 2.2 以降で必要なデコードを実行する
    item=$(decode_url "${define_items[$index]}")
    # FLAVORが含まれるDart Defineの場合
    if [ $(echo $item | grep 'FLAVOR') ] ; then
        # FLAVORの値(=の右側)
        value=${item#*=}
        # FLAVORに対応したXCConfigファイルをincludeさせる
        echo "#include \"$value.xcconfig\"" >> $OUTPUT_FILE
    fi
done
```
:::

ここでは…
1. `$DART_DEFINES` という変数に格納されている `dart-define` を取得して
2. `FLAVOR`の場合、その値（dev等）を取り出し、 `DartDefines.xcconfig` に `{Flavor名}.xcconfig` をインクルードさせています。

`$DART_DEFINES` のすべての変数を書き込んでしまえば楽なのですが、ビルドできなくなってしまったので、 `FLAVOR` に絞って書き込んでいます🧐

:::message alert
上記はFlutter 2.2 以降に対応したスクリプトとなります。
Flutter 1.17, Flutter 1.20, Flutter 2.2 で必要なデコード処理が変更されました。
Flutter 2.2以前の書き方については参考記事を参照ください。
https://itnext.io/flutter-1-17-no-more-flavors-no-more-ios-schemas-command-argument-that-solves-everything-8b145ed4285d#1379
:::

### XcodeのBuild Pre-actions に作成したスクリプトを登録する

1. Scheme (Runner) をクリックして `Edit scheme` -> Build を展開して `Pre-actions` を選択します。
1. 「＋」ボンタンを押して「New Run Script Action」を選択します。
1. 「Provide build settings from」は `Runner` を選択します。
1. 先ほど保存したファイルのパスである `${SRCROOT}/scripts/retrieve_dart_defines.sh` を書き込みます。

![](/images/separating-environments-in-flutter/build-pre-action-for-dart-defines.png)

### スクリプトファイルに実行権限を与える
そのままビルドしてもスクリプトファイルが実行されません。
以下のコマンドで実行限限を与えておきましょう。
```shell
chmod 755 ios/scripts/retrieve_dart_defines.sh
```
https://qiita.com/shisama/items/5f4c4fa768642aad9e06
https://neos21.net/blog/2020/09/17-02.html

### 各種xcconfigファイルで `DartDefines.xcconfig` をインポート
前項のスクリプトで生成される `DartDefines.xcconfig` がDebug, Releaseビルド両方で使われるように、
両方の `***.xcconfig` でインクルードします。

`ios/Flutter/Debug.xcconfig`
```diff xcconfig
 #include? "Pods/Target Support Files/Pods-Runner/Pods-Runner.debug.xcconfig"
 #include "Generated.xcconfig"
+ #include "DartDefines.xcconfig"
```

`ios/flutter/release.xcconfig`
```diff xcconfig
 #include? "Pods/Target Support Files/Pods-Runner/Pods-Runner.release.xcconfig"
 #include "Generated.xcconfig"
+ #include "DartDefines.xcconfig"
```

### `.gitignore` ファイルに追加
`DartDefines.xcconfig` は、ビルド時に生成され、FLAVORにより内容が変わるので、Git管理対象外にします。
```diff gitignore
+ **/ios/Flutter/DartDefines.xcconfig
```

## アプリ表示名を環境によって変える
`ios/Runner/Info.plist` を編集します。

アプリ名に使われる `CFBundleDisplayName` にアプリ名 + `APP_ID_SUFFIX` を指定します。

```diff plist
 <key>CFBundleName</key>
- <string>flutter_app_template</string>
+ <string>FlutterAT$(APP_ID_SUFFIX)</string>
+ <key>CFBundleDisplayName</key>
+ <string>FlutterAT$(APP_ID_SUFFIX)</string>
```

:::message
`CFBundleDisplayName` がiOS端末のホームアイコンに表示されるアプリ名となります。
`CFBundleName` の方はiOSに限れば使用されている箇所を見つけることができませんでした。
:::

環境ごとに以下のようなアプリ名が表示されるようになりました👍
- dev: FlutterAT.dev
- stg: FlutterAT.stg
- prod: FlutterAT

のようになります。

## Bundle IDを環境によって変える
`ios/Runner.xcodeproj/project.pbxproj` を `PRODUCT_BUNDLE_IDENTIFIER` で検索するか、
Xcode > Runner > TARGETS Runner > Build Settings の `Product Bundle Identifier` を表示して、
`$(APP_ID_SUFFIX)` を末尾に追加します。

![](/images/separating-environments-in-flutter/add-bundle-id-suffix-from-build-settings.png)
*画像で使っている　`FLAVOR_SUFFIX` は古い名前です。 `APP_ID_SUFFIX` に変更しました*

忘れずに Debug, Profile, Release すべてに接尾辞を追加して共通の値になるように注意しましょう。

```diff pbxproj
# ios/runner.xcodeproj/project.pbxproj
- PRODUCT_BUNDLE_IDENTIFIER = "jp.co.altive.fat";
+ PRODUCT_BUNDLE_IDENTIFIER = "jp.co.altive.fat$(APP_ID_SUFFIX)";
```

これで、環境ごとにアプリのBundle IDが変わるようになりました👏

## Appアイコンを環境によって変える
`ios/Runner.xcodeproj/project.pbxproj` を `ASSETCATALOG_COMPILER_APPICON_NAME` で検索するか、
Xcode > Runner > TARGETS Runner > Build Settings の `Primary App Icon Set Name` を表示して、
`AppIcon` と指定されている値を `AppIcon-$(FLAVOR)` に変更します。

![](/images/separating-environments-in-flutter/xcode-primary-app-icon-set-name.png)

忘れずに Debug, Profile, Release すべて共通の値になるようにしましょう。

```diff pbxproj
# ios/runner.xcodeproj/project.pbxproj
- ASSETCATALOG_COMPILER_APPICON_NAME = "AppIcon";
+ ASSETCATALOG_COMPILER_APPICON_NAME = "AppIcon-$(FLAVOR)";
```

これで、環境ごとにアプリのアイコンが変わるようになりました👏

## Firebase対応 (iOS)
Firebaseを使用していてかつ環境ごとにプロジェクトを分ける場合は、`GoogleService-Info.plist` を環境ごとに使い分ける必要があります。

:::message
Firebaseを使用していない場合や、環境ごとにFirebaseプロジェクトを分けない場合は読み飛ばしてください。
:::

### `ios` ディレクトリに各環境（Flavor）と同名のディレクトリ（フォルダ）を作成。
- `ios/dev/` , `ios/stg/` , `ios/prod/`

各Firebaseプロジェクトに作ったiOSアプリ用の `GoogleService-Info.plist` をそれぞれのディレクトリに配置します。

### Select GoogleService-Info.plist
ビルド時に、環境に対応した `GoogleService-Info.plist` を使用できるようにするスクリプトを追加します。
1. `Build phases` -> `New run script` を選択して新しい `Run Script` を追加
1. 名前をわかりやすいように `Select GoogleService-Info.plist` に変更
1. スクリプトを記述
1. `Output Files` に `$(SRCROOT)/GoogleService-Info.plist` を追加
1. 既存Scriptである `Copy Bundle Resources` より上に移動

```shell
\cp -f ${SRCROOT}/${FLAVOR}/GoogleService-Info.plist ${SRCROOT}/GoogleService-Info.plist
```

![](/images/separating-environments-in-flutter/select-googleservice-info-plist-to-build-phases.png)

### GoogleService-Info.plistへの参照を追加する
（[@akaboshinit](https://zenn.dev/link/comments/82fafd879f7b7b) さん、追加情報ありがとうございます！）

実際には、スクリプトによってコピーされた `GoogleService-Info.plist` が使用されることになるのですが、このままでは `${SRCROOT}/GoogleService-Info.plist` ファイルへの参照がなくXcodeで認識できません。
Xcode上でRunnerディレクトリへ `GoogleService-Info.plist` ファイル（どの環境のものでも良い）をドラッグ＆ドロップで追加してファイルと参照を追加しましょう。

![](/images/separating-environments-in-flutter/xcode-google-service-info-plist-adding.png)
*Downloadフォルダ等から、Runnerフォルダ内へドラッグ＆ドロップ (`Copy items if needed` にチェック）*

![](/images/separating-environments-in-flutter/xcode-google-service-info-plist-added.png)
*Runnerフォルダ内に追加されました。*

このファイルはビルド時に任意の環境用の `GoogleService-Info.plist` に置き換わります🙆‍♂️

### `.gitignore` ファイルに追加
`ios/GoogleService-Info.plist` は、ビルド時に生成されてFLAVORにより内容が変わるので、Git管理対象外にします。
```diff gitignore
+ **/ios/GoogleService-Info.plist
```

# Flutterアプリを起動して、Flavorがきちんと伝わっているか確かめる
`--dart-define=Flavor` がきちんとネイティブに伝わり、アプリ名やBundle IDがFlavorごとに変更されていることを手軽に確かめるためには、 `package_info_plus` パッケージを使用します。

https://pub.dev/packages/package_info_plus

`PackageInfo` の下記メソッドを使用して確認できます。

- `.appName`
  - iOS: アプリ名 (`CFBundleDisplayName`)
  - Android: `android:label="@string/app_name"`
- `.packageName`
  - iOS: Bundle ID (`CFBundleIdentifier`)
  - Android: `applicationId`

# 宣伝
Riverpod の実践入門本を書きました👍
https://zenn.dev/riscait/books/flutter-riverpod-practical-introduction

# 参考記事
https://itnext.io/flutter-1-17-no-more-flavors-no-more-ios-schemas-command-argument-that-solves-everything-8b145ed4285d
https://medium.com/flutter-jp/flavor-b952f2d05b5d
https://qiita.com/hiromasa-fun/items/c79c99535f6f1db2a6a9
https://zenn.dev/kingu/articles/9192b91062841b8e0bba
https://medium.com/flutter-jp/icon-935d637d2da0
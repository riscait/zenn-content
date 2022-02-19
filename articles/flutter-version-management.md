---
title: "FVMでFlutter SDKのバージョンをプロジェクト毎に管理する"
emoji: "🔄"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [flutter, fvm]
published: true
---

こんにちは、Altive株式会社のFlutterアプリ開発者、村松龍之介（[@riscait](https://twitter.com/riscait)）です😸

Flutter SDKのバージョン管理ツールであるFVM（Flutter Version Management）をご紹介します。

https://fvm.app/

上のリンクの通り、丁寧な公式ドキュメントがありますので、そちらで事足りる方には不要な説明が多いと思います。

※Flutterのバージョン管理というと asdf の名前も上がりますが、併用困難等の利用から筆者は FVM のみを使用しています。

:::message
これより先の記述は筆者の環境であるMacでの説明になります。それ以外のWindows等では当てはまらない説明があるかもしれません。適宜読み替えてください。
:::

# FVMの用途・機能は大きく2つ
1. 複数のFlutter SDKバージョンをインストールできる。
→複数のFlutter SDKをインストールしておくことができるので、切り替えが容易になります。
1. `fvm_config.json` を使用することで、Flutter SDKバージョンを明示してチームで統一できる。

複数のFlutterアプリ開発に携わるならFlutter SDKのバージョン管理は必須とも言えます。

こちらのアプリは stable（安定板）の最新 を使ってるけど、あちらのアプリは v1.17.0 のままだし🙀
あ、あちらは dev（開発版）を使ってる！

といったときに、都度グローバルにインストールしたFlutterのバージョンを上げたり下げたりするのは面倒です。。

FVMを使って、アプリごとにチームでFlutter SDKのバージョンを統一しましょう👍

## FVMを選ぶ利点
- Flutterバージョンの切り替えが簡単（コマンド1つ）
- バージョン情報を含むパスを設定ファイルに記述しなくて良い
- VS CodeならUser Sattingsでパスを指定すればプロジェクトごとの設定は必須ではない

## FVMのちょっと気になるところ
- `fvm flutter xxx` のように、flutterを指定したバージョンで実行したい時はプレフィクスが必要

# FVMのインストール
DartかHomebrewを使ってインストールできます。

## Dartでインストール
```shell
dart pub global activate fvm

fvm --version # -> インストールしたfvmのバージョンが表示されればOK👌
```

## Homebrewでインストール
```shell
brew tap leoafarias/fvm
brew install fvm
```

## その他のインストール方法
GitHubのリポジトリから直接ダウンロードする方法もあります。
Windowsでは `choco` コマンドでインストールできるようです。

https://fvm.app/docs/getting_started/installation

## PATHを通す
環境により違いはあると思いますが、 `fvm` コマンドを使用するために以下のPATHが必要になることが多いです。

```shell:.zshrc
export PATH="$PATH:$HOME/.pub-cache/bin"
```

僕は ZSH を使用しているため `~/.zshrc` に追加しましたが、bashやfish等環境により差異がありますのでご確認ください。

# プロジェクト（アプリ）で使用するFlutterバージョンを指定する方法

## `releases` で使用できるFlutterバージョンを確認する
今現在、どのFlutterバージョンが使えるのか、最新のstable更新されたかな？
といったときには次のコマンドを使用します。

```shell
fvm releases
```

以下の出力の場合、stable（安定板）の最新は `2.5.2` であることが分かります。

出力サンプル
```
Sep 8 21   │ 2.5.0            
Sep 10 21  │ 2.5.0            
Sep 13 21  │ 2.6.0-5.1.pre    
--------------------------------------
Sep 16 21  │ 2.6.0-5.2.pre     beta
--------------------------------------
Sep 17 21  │ 2.5.1            
--------------------------------------
Sep 25 21  │ 2.6.0-11.0.pre    dev
--------------------------------------
--------------------------------------
Sep 30 21  │ 2.5.2             stable
```

:::message
出力されたFlutterバージョンが古い、最新のバージョンが反映されていない。。というときは、FVM自体のバージョンを上げると解決する可能性があります。
`dart pub global activate fvm`
:::

## `use` でFlutterバージョンを指定する
使用するFlutterバージョンを決めたらuseコマンドを使いましょう。
バージョン番号を指定するか、Channelを指定します（`stable` , `beta` , `dev` , `master`）
```
fvm use 2.5.2
or
fvm use stable
```

`.fvm/flutter_sdk` と `.fvm/fvm_config.json` が生成されました👏
前者はSDKへの相対シンボリックリンク、後者は指定したFlutter SDKのバージョンが書き込まれています。

```json:fvm_config.json
{
  "flutterSdkVersion": "2.5.2",
  "flavors": {}
}
```

まだインストールしていないFlutterバージョンだった場合は、「未インストールだよ。インストールする？（意訳）」と訊かれるので、Yesを選べばインストールもしてくれます👍
> Flutter "stable" is not installed.
> Would you like to install it? Y/n: 

## インストールのみを行うコマンド
ここでは使わないけどインストールだけはしておきたい、というときは install コマンドを使用します。
```
fvm install 2.5.2
```

## `list` でインストール済みのFlutter SDKバージョンを確認する
`list` コマンドを使うとインストール済みのバージョンが一覧表示されます。

```
fvm list
```

出力サンプル
```shell
stable (global) # globalで使用されるバージョン（後述）
dev
2.5.2 (active) # 現在のディレクトリで有効なバージョン
2.4.0-4.2.pre
2.0.0
1.22.5
```

# プロジェクトごとに必要な設定+IDEの設定 
FVMを導入したアプリプロジェクトで初回に行う設定は以下の通りです。
## .gitignore に追記
FVMは、選択したバージョンのFlutter SDKへの相対シンボリックリンクをプロジェクトの `.fvm/flutter_sdk` に追加します。
こちらは、プロジェクトのバージョン管理には不要なものとなるので `.gitignore` に追記してバージョン管理外としましょう。
```shell:.gitignore
.fvm/flutter_sdk
```

## Android Studio の設定例
`Preferences > Language & Frameworks > Flutter` の 「SDK」内 `Flutter SDK path`　に以下のパスを入力します。
```
/{プロジェクトまでのパス}/.fvm/flutter_sdk
```

そのアプリで使うFlutter SDKのバージョンを変更するたびに書き換える必要はありません👍

## VS Code
設定 `dart.flutterSdkPath` に `".fvm/flutter_sdk"` を設定します。
```json:settings.json
{
    "dart.flutterSdkPath": ".fvm/flutter_sdk"
}
```

VS Codeには、ユーザーの設定とワークスペースの設定の2つの設定ファイルがあります。
コマンドパレット（⌘ + shift + p ）で「settings」と打ってみると以下の選択肢が表示されます。
- *Preferences:Open Settings (JSON)*
- *Preferences:Open Workspace Settings (JSON)*

上がユーザーごとの設定、下がワークスペース（多くはアプリ）ごとの設定ファイルを開くコマンドです。
自分だけでfvmを使うだけであれば、ユーザー設定に追記するだけで良いです。
チームメンバーにもユーザー設定に追記するようREADME辺りに書いておきましょう。

---

しかし、 `.vscode/settings.json` をGit管理にしても大丈夫な場合は、
Workspaceのsettings.jsonを編集すると、より確実にVS Codeを使用しているメンバーはFVMで指定したFlutterバージョンを使用することになります。

かつ、ユーザー設定よりワークスペース設定の方が優先されるのでチームメンバーが `settings.json` をいじる必要はありません。

`Preferences:Open Workspace Settings (JSON)` を選ぶか、新規に `.vscode/settings.json` ファイルを作成しましょう。

開いた `settings.json` に `"dart.flutterSdkPath": ".fvm/flutter_sdk"` を追記すればOKです。

```json
{
    // 使用するFlutter SDKのパスを指定。
	"dart.flutterSdkPath": ".fvm/flutter_sdk",
    // 検索対象からFVMのファイルを除外します。(任意)
    "search.exclude": {
        "**/.fvm": true
    },
    // ファイル監視対象からFVMのファイルを除外します。(任意)
    "files.watcherExclude": {
        "**/.fvm": true
    },
}
```

:::message
`fvm use` コマンドでバージョンを変更したのにIDEでバージョンが切り替わらない場合は一度プロジェクトのウィンドウを閉じて再度開きましょう。
:::

# 参加したプロジェクトでFVMが使われていた時にすること
FVMが使われ、 `fvm_config.json` のあるプロジェクトに参画した場合にやることはシンプルです。
```shell
fvm install
```
以上のコマンドを実行すれば、指定のFlutter SDKバージョンがなければインストールされます👍

# flutter createでFVMを使う（Flutter以外のディレクトリでFVMを使用する）
Flutter SDKのバージョンを指定してから、`flutter create` コマンドで新規プロジェクトを作成したい場合などに `fvm use x.x.x` としても…

> Not a Flutter project. Run this FVM command at the root of a Flutter project or use --force to bypass this.

のように、「Flutterプロジェクトではありません」と言われてしまいます。

その場合は `--force` オプションを付けることで強制的に fvm コマンドを使用し、Flutter SDKバージョンを指定することが可能です。
（結果、Flutterプロジェクトで行った時と同様、 `.fvm/` が生成されます）

```shell
fvm use 2.5.2 --force
# 後述のglobalを設定していなくても、 指定したfvmコマンドが使える
fvm flutter create .
```

# `global` を使えばどこでも `flutter` コマンドが使用できる
個別に `fvm use x.x.x` でバージョンを指定したプロジェクト内では当然 fvmコマンドが使用できますが、
それ以外の場所で使用するFlutter SDKバージョンを指定するのが `global` コマンドです。

global コマンドで入れたFlutterバージョンは `fvm` を付ける必要がありません👌

```shell
fvm global stable
flutter --version # <- Flutter 2.5.2 • channel stable ...
```

のように global コマンドを使ってバージョンを使用すれば `flutter` コマンドがどこでも使用できるようになります。
単一のFlutter SDKをインストールするのと違いが分かりにくいですが、fvm globalを使用する利点としては、より高速なバージョンの切り替えとキャッシュの恩恵を受けられることにあります。

:::message
FVM以外でインストールしたFlutter SDKがある場合は、競合を避けるため削除しましょう。
Flutter公式ドキュメントにそってインストールした場合は `~/development` にあります👍
:::

## global な `flutter` コマンドを使用するにはPATHを通しましょう。
初めて global コマンドを使用した時、おそらく以下のような表示が出るかと思います。

> Flutter "stable" has been set as global
> However your "flutter" path current points to:
> .
> to use global Flutter SDK through FVM you should change it to:
> /Users/{user_name}/fvm/default/bin

PATHの追加が必要になります。

筆者の場合は以下のPAHTを `~/.zshrc` に追記しています。（`$HOME` が `/Users/{user_name}` を表す変数です）
```shell:.zshrc
export PATH="$PATH:$HOME/fvm/default/bin"
```

# その他のコマンド
## `remove` で不要なSDK削除
もう使用しないFlutter SDKバージョンを削除するには `remove` コマンドを使用します。

```
fvm remove 2.5.2
```

## `doctor` で環境とプロジェクトの構成情報を表示する
現在使用しているFlutter SDKのバージョンやIDEに設定すべきFlutter SDK Pathを見ることができます。

```
fvm doctor
```

:::details 出力例
FVM Version: 2.2.3
___________________________________________________

FVM config found:
___________________________________________________

Project: zenn-content
Directory: /Users/riscait/Projects/Riscait/zenn-content
Version: stable
Project Flavor: None selected
___________________________________________________

Version is currently cached locally.

Cache Path: /Users/riscait/fvm/versions/stable
Channel: true
SDK Version: 2.5.2

IDE Links
VSCode: .fvm/flutter_sdk
Android Studio: /Users/riscait/Projects/Riscait/zenn-content/.fvm/flutter_sdk


Configured env paths:
___________________________________________________

Flutter:
/Users/riscait/fvm/default/bin/flutter

Dart:
/opt/homebrew/Cellar/dart/2.14.3/libexec/bin/dart

FVM_HOME:
not set
:::

# flutterの前にfvmと打つのが嫌な人は、エイリアス追加も1つの方法です
筆者は .zshrc に以下を追記して `fvm flutter` -> `f` , `fvm dart` -> `d` としています。
```shell:.zshrc
alias f="fvm flutter"
alias d="fvm dart"
```

## 筆者のFVMの使い方
Flutterアプリ開発では基本的にFVMを使用してFlutter SDKバージョンを固定してチームで同じバージョンを使用できるようにしています。

一方で、FlutterやDartのOSSなパッケージなどはFVMでのバージョン指定がされていないことがほとんどなので、 `fvm global` で `stable` を指定して使っています👌

# 最後に
## 宣伝
Riverpod の実践入門本を書きました👍
https://zenn.dev/riscait/books/flutter-riverpod-practical-introduction

## 参考
https://fvm.app/
https://flutter.dev/docs/development/tools/sdk/releases
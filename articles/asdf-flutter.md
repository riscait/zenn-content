---
title: "asdfでFlutter SDKのバージョンをアプリ毎に管理する"
emoji: "🔁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [flutter, asdf, fvm]
published: true
---

:::message alert
以下の事由により `asdf` をアンインストールしました。
現在は[FVM](https://fvm.app/)を使用しています。

1. FVM と併用できない。 asdf のパスを通すと `fvm` コマンドが使えなくなってしまう。
1. ローカルパスを `Flutter Sdk Path` に追記しなければいけない
1. プロジェクトで選択したFlutterバージョンが、VS Codeの `.vscode/settings.json` にローカルパスとして記載されてしまうこと

僕の使い方が悪いだけの可能性もありますが、ひとまず asdf の使用は諦めます🙏
:::

https://zenn.dev/riscait/articles/flutter-version-management

---

複数のFlutterプロジェクトに携わるならFlutter SDKのバージョン管理は必須とも言えますよね。

こちらのアプリは v2.0.0 を使ってるけど、あちらのアプリは v1.17.0 のままだし、さらにあちらは devチャンネルの v2.4.0.pre を使ってる！

といったときに、都度グローバルにインストールしたFlutterのバージョンを上げたり下げたりするのは面倒です。。

---

[FVM(Flutter Version Management)](https://fvm.app/)を長らく使っていましたが、[朝日さんの記事](https://blog.dalt.me/2730)を見てから気になっていた asdf を導入してみました！

:::message
FVMは名前の通りFlutter用のバージョン管理ツールでしたが、asdf 自体はFlutter専用ではなくPython等様々なSDKで使うことができます。
:::

# asdf の導入（インストール）
https://asdf-vm.com/guide/getting-started.html#_1-install-dependencies

Macの場合は、 Homebrew で簡単にインストールできます。
```shell
brew install asdf
```
パスを通します。
Homebrewでインストールし、 `ZSH` を使用している場合は以下のコマンドを実行すればパスが通ります。

```shell
# パスを通す
echo -e "\n. $(brew --prefix asdf)/asdf.sh" >> ${ZDOTDIR:-~}/.zshrc
# そのまま同じ Terminal を使うなら、 `.zshrc` を更新して通したパスを反映させる
source ~/.zshrc
```

:::message
BashやFishを使っている場合、またはHomebrewではなく Git等でインストールした場合のパスの通し方は以下をご覧ください。
https://asdf-vm.com/guide/getting-started.html#_3-install-asdf
:::

## インストール可能なFlutterバージョンの一覧を見る
asdfにてインストール可能なFlutterのバージョンが一覧するには、asdf の `list all` コマンドを実行します。
```shell
asdf list all flutter
```

## 使用したいFlutterバージョンをインストールする
asdf の `install` コマンドを実行します。
```shell
# Flutter 2.2.3 (stable) をインストールする場合
asdf install flutter 2.2.3-stable
```

:::message
最新のStableバージョンをインストールしたい場合は
```shell
asdf install flutter latest
```
で最新版（Stable）をインストールできます👍
:::

asdfにてインストール済みのFlutterバージョンを確認するには asdf の `list` コマンドを使用します。
```shell
asdf list flutter
```

# プロジェクトごとにasdfでFlutterバージョンを指定する（変更も同じ）
Flutterバージョンのアップグレードも同じコマンドを使います👌

asdf の `local` コマンドを実行します。
```shell
# プロジェクトのディレクトリに移動していない場合は移動します。
cd アプリプロジェクトのディレクトリ
# アプリプロジェクトで使用したいバージョンを指定します。
asdf local flutter 2.2.3-stable
```
このコマンドを実行すると、 `.tool-version` が作成されます。
`.tool-version` がすでに存在する場合は、指定したFlutterバージョンに書き換えられます👍

以下、作成される `tool-version` 例です。
Flutter のバージョンが指定されていますね👌
```.tool-version
flutter 2.2.3-stable
```

:::message
なお、 `asdf local` コマンドで指定するFlutterバージョンをインストールしていないと使えません。先に `asdf install` コマンドでインストールしておきましょう。
:::

https://asdf-vm.com/guide/getting-started.html#local

:::message
`asdf global` コマンドを使えば、プロジェクト毎ではなく、グローバルで使うFlutterバージョンを指定できますが、複数プロジェクトに参加してたり複数アプリの開発をしている場合は使わない方が混乱しにくく良いと思います。
:::

## 指定したFlutterバージョンが使えているか、動作確認する
```shell
flutter --version
```
asdfで指定したFlutterバージョンになっていればOKです。

# 新しく参加したプロジェクトに `.tool-versions` がある場合にすること

## A. 指定のFlutterバージョンをasdfでインストール済み
何もする必要はありません。そのまま `flutter` コマンドが使用できます。

## B. 指定のFlutterバージョンをasdfでまだインストールしていない
以下のコマンドでインストールしましょう。

```shell
asdf install
```

`.tool-versions` ファイルで指定されているSDK（Flutter）がインストールされます。

:::message
ちなみに…
`.tool-versions` で指定されているバージョンをインストールしていない状態で `flutter` コマンドを使用しようとすると以下の警告が表示されます。

> No preset version installed for command flutter
> Please install a version by running one of the following:
> asdf install flutter {.tool-versionで指定されているFlutterバージョン}

指定のバージョンをインストールする必要があるということを教えてくれていますね。
:::

# VSCodeで必要な設定
:::message alert
最初に当記事を公開した際は、IDEでの設定は不要だと書きましたが、必要でした…🙇‍♂️
:::

## VSCode の User Settings で設定する
1. Macだと `⌘ + ,` か、メニューから `Code -> Preferences -> Settings` で設定画面を開きます。
1. `Search settings` に `flutter sdk` と入力します。
1. `Dart: Flutter Sdk Paths` に asdf でインストールしたFlutter SDKsのパスを入力します。
多くの場合は `/Users/{username}/.asdf/installs/flutter/` で良いかと思います。

以下は筆者の設定例です。ユーザー名は `riscait` の部分を変えてくださいね。
![](/images/asdf-flutter/vscode-flutter-sdk-paths-setting.png)

パスとしてディレクトリを指定しました！
これで、今後複数のFlutterバージョンをインストールしても、設定をいじる必要はありません。

## 複数のFlutterバージョンから、使用したいバージョンを選択する
User Settingsを設定しただけでは、複数のFlutterバージョンをインストールしていた場合はどのバージョンを使って良いのかVSCodeが判断できません。
以下の手順で選択しましょう。

VSCode右下に現在のFlutterバージョンが表示されています。
※表示されていない場合は、 `main.dart` など、 Dartファイルを開くと表示されます。
![](/images/asdf-flutter/vscode-current-flutter-version.png)

ここのバージョンが希望のバージョンになっていない場合は、Flutter 2.0.6 の場所をクリックしましょう。
以下のように選択画面が表示されます。
![](/images/asdf-flutter/vscode-select-an-sdk-paths-setting.png)

:::message
Android Studio は使っていないので現時点で試せていません。。
:::

# ※FVMを使っていた方、使っているプロジェクトがある方の注意点
FVMを導入したときに、 VSCodeの `settings.json` で `dart.flutterSdkPath` を指定している場合は asdf の使用に問題がありそうです。

Macであれば `⌘+,` で `settings.json` を開き、 `dart.flutterSdkPath` を削除しましょう。

![](/images/asdf-flutter/vscode-flutter-sdk-path-setting.png)

## プロジェクトの settings.json で `.fvm/flutter_sdk` が使用されている場合
`.vscode/settings.json` で 以下の指定がある場合、今のところ問題なく `flutter` コマンドは使えていますが、問題があれば追記します。

```
"dart.flutterSdkPath": ".fvm/flutter_sdk"
```

# 参考文献
https://asdf-vm.com/
https://github.com/asdf-vm/asdf
https://blog.dalt.me/2730
https://dev.classmethod.jp/articles/try-asdf-settings/
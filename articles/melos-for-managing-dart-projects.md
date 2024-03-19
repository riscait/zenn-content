---
title: "【v5.2対応】Melosでモノレポ管理"
emoji: "🌋"
type: "tech"
topics: [melos, dart, flutter, cli]
published: true
publication_name: "altiveinc"
# cSpell: words fluttercommunity rroussel postclean Podfile IntelliJ
# cSpell: ignore hoge
---

こんにちは、Flutterでのアプリ開発をメインとしている「[Altive株式会社](https://altive.co.jp)」の村松龍之介（[@riscait](https://x.com/riscait)）です！

![](/images/ProfileBanner_Muramatsu.jpg)

以前にMelos `v1.0.0` についてのチートシートという記事を書きました。

https://zenn.dev/altiveinc/articles/melos-for-multiple-packages-dart-projects


もう `v5.1` がリリースされたとのことで進化を感じますね…！

弊社でもいくつかのアプリプロジェクトでMelosを使ったモノレポ運用を行っております。

弊社 `flutter_app_template` でもMelosを採用しています。
`melos.yaml` はこちら↓

https://github.com/altive/flutter_app_template/blob/main/melos.yaml

# 目次

# Melosとは？

複数のパッケージを持つDartプロジェクト（Monorepoとも呼ばれる）の管理支援CLIツールです。

大規模なプロジェクトをバージョン管理された**パッケージに分割することはコード共有のために非常に有効**です。

しかし、多くのリポジトリにまたがって変更を加えるのは、手間がかかり、追跡が難しく、リポジトリをまたがるテストも複雑になりがちです。

Melosでは、**複数のパッケージが互いに独立しながらも、1つのリポジトリで一緒に作業できる**ように支援してくれます。

また、[Invertase](https://invertase.io/) さんによって構築・メンテナンスが行われています。

## 主な機能
- 複数のパッケージで同時にコマンドを実行
- ローカルパッケージのリンクとインストール
- ローカルパッケージとその依存関係のリストアップ
- 自動バージョン管理とチェンジログの作成
- pub.devへのパッケージの自動公開
- CI/CD環境での自動化支援

## Melosが使われているプロジェクト
Flutterアプリ開発者にとっては馴染みのありそうなリポジトリでMelosは実際に使われています。

READMEの `who is using melos` に多数リストアップされています。

https://pub.dev/packages/melos#who-is-using-melos

以下、いくつかをリストアップします。
それぞれの `melos.yaml` を見るだけでも勉強になりますね👀

- [firebase/flutterfire](https://github.com/firebase/flutterfire)
- [fluttercommunity/plus_plugins](https://github.com/fluttercommunity/plus_plugins)
- [gql-dart/ferry](https://github.com/gql-dart/ferry)
- [flutter-stripe/flutter_stripe](https://github.com/flutter-stripe/flutter_stripe)
- [FlutterGen/flutter_gen](https://github.com/FlutterGen/flutter_gen)
- [rrousselGit/riverpod](https://github.com/rrousselGit/riverpod)

**FlutterFire, PlusPlugins, Riverpod**等々、
個人的に日頃お世話になっているパッケージで使われていました。

## Flutterモノレポ構成例
以下、Flutterアプリでモノレポと**Melos**を採用する場合のリポジトリ（ディレクトリ）構成の一例です。

```
- packages/
  - foo_package
    - lib/
      - src/
      - foo_package.dart
    - pubspec.yaml
  - bar_package
    - lib/
      - src/
      - bar_package.dart
    - pubspec.yaml
  - hoge_app
    - lib
      - src/
      - main.dart
    - pubspec.yaml
- .gitignore
- analysis_options.yaml
- melos.yaml
- pubspec.yaml
```

ルートに、 `melos.yaml` や `pubspec.yaml` を配置します。

:::message
FVMもモノレポに対応しているため、ワークスペース（リポジトリ）のルートで `fvm use` を実行すればOKです👍
:::

上の例でいうと、 `hoge_app` が `foo_package` と `bar_package` に依存している場合は、
`hoge_app` の `pubspec.yaml` にパスで指定しましょう。

```yaml:hoge_app/pubspec.yaml
dependencies:
  foo_package:
    path: ../foo_package/
  bar_package:
    path: ../bar_package/
```

# Melosのセットアップ

## インストール
以下のコマンドで pub.dev からグローバルなパッケージとしてインストールしましょう。

```shell
$ dart pub global activate melos

$ melos --version
5.1.0
```

## melos.yaml ファイルの作成
Melosを使用したいワークスペース（リポジトリ）のルートに `melos.yaml` ファイルを新規作成します。

```yaml:melos.yaml
name: my_project
repository: https://github.com/altive/flutter_app_template
packages:
  # packagesディレクトリとそのexampleパッケージを対象にする例(globeパターン使用)
  - packages/*
  - packages/*/example
ignore: # 除外したいパッケージがあればリストアップ可能
  - packages/foo
```

パスの指定には、 `.gitignore` でも使われている [glob](https://docs.python.org/3/library/glob.html)パターンを使用できます。

## pubspec.yaml ファイルの作成
モノレポ構成の場合、プロジェクトディレクトリのルート（ワークスペース）はDartパッケージではありませんが、 pubspec.yamlを配置する必要があります。

```yaml:pubspec.yaml
name: foobar_workspace # 任意のワークスペース名

environment:
  sdk: ^3.3.0 # このワークスペースで使用したいDartのバージョン

dev_dependencies:
  melos: ^5.1.0 # このワークスペースで使用したいMelosのバージョン
```

指定したMelosのバージョンがワークスペース使用されます。

# Commands

## melos bootstrap
Melosのインストールと `melos.yaml` の作成が完了したら、次は `melos bootstrap` を実行しましょう。

- 全パッケージの依存関係がインストールされます（内部的には `pub get` が使われます）
- 全パッケージがローカルにリンクされます。

```shell
melos bootstrap
# または
melos bs
```

### bootstrapコマンドのHooks
`melos.yaml` で `command.bootstrap.hooks` を指定することでフックを指定することできます。

以下の例では、 `bootstrap` コマンド実行後に `flutter gen-l10n` を実行しています。

```yaml:melos.yaml
command:
  bootstrap:
    hooks:
      post: |
        melos exec --flutter --dir-exists=lib/l10n -- "flutter gen-l10n"
```

### Bootstrapが必要な理由

通常のプロジェクトでは、pubspec.yaml 内でパスを指定することでパッケージをリンクすることができます。
小規模なプロジェクトや公開しないパッケージでは有効ですが、
そうではない場合、パッケージの公開時に手動ですべてのパッケージのpubspec.yamlファイルをバージョンで更新する必要が出てきます。

また、パッケージが密結合（互いに依存している）している場合、どのバージョンを更新すべきかを手動でチェックする必要があり、煩雑です。

https://melos.invertase.dev/commands/bootstrap

## melos clean

プロジェクトの一時ファイルをクリーンアップします。
`bootstrap` の前に実行するなどして完全に新しい状態でスタートできます。

Cleanコマンドについてより詳しくはこちら↓
https://melos.invertase.dev/commands/clean

### cleanコマンドのHooks
`melos.yaml` で `command.clean.hooks` を指定することでフックを指定することできます。

以下の例では、 `clean` コマンド実行後に `flutter clean` と Podfileの削除を行なっています。

```yaml:melos.yaml
command:
  clean:
    hooks:
      post: |
        melos exec --flutter -- "flutter clean"
        melos exec --flutter --file-exists="ios/Podfile.lock" -- "cd ios && rm Podfile.lock"
```

## melos analyze
`v5.0.0` でビルトインコマンドとして追加されました🚀

各パッケージで `flutter analyze` が実行されます。

## melos format
`v5.1.0` でビルトインコマンドとして追加されました🚀

各パッケージで `dart format .` が実行されます。

## melos exec
`exec` コマンドは、すべてのパッケージに対して任意のコマンドを実行するコマンドです。

```shell
# すべてのパッケージで依存関係を取得するコマンドを実行する
melos exec -- "flutter pug get"
```

繰り返し使うコマンドは、 `exec` コマンドではなく後述の `scripts` を定義し、`melos run` コマンドで簡単に呼び出せるようにしましょう👌

:::message
`melos exec` と 実行したいコマンドを区切る `--` は、 "Double Dash` と呼ばれる、コマンドラインオプションのスキャン終了を示すためのものです。
https://www.wikitechy.com/technology/double-dash-mean-also-known-bare-double-dash/
:::

https://melos.invertase.dev/commands/exec

### concurrency (-c)
同時にコマンドを実行するパッケージの最大数を定義します。デフォルト数は実行プラットフォームにより異なります。
```shell
# パッケージ1つずつ順番にコード生成実行を行いたい場合
melos exec -c 1 -- "flutter pub run build_runner build -d"
# パッケージ8つで並行してコード生成監視を行いたい場合
melos exec -c 8 -- "flutter pub run build_runner watch -d"
```

### fail-fast
いずれかのパッケージでスクリプトが失敗した場合に、それ以降のパッケージでスクリプトを実行しないようにします。
デフォルトはfalseです。

```shell
melos exec --fail-fast -- "dart analyze ."
```

## melos list
認識されているパッケージのリストを出力します。

```shell
melos list
foo_package
bar_package
hoge_app
```

https://melos.invertase.dev/commands/list

## melos version
melos version コマンドは、パッケージのバージョンを更新するためのコマンドです。

[Conventional Commits](https://www.conventionalcommits.org/ja/v1.0.0/) を使用してコミットしていれば、
[SemVer（セマンティック バージョニング）](https://semver.org/lang/ja/) に基づいてバージョンを自動で判定してくれます。

:::message
破壊的な変更（breaking change）を含むコミットがある場合、メジャーバージョンが上がり、
新機能（feat）のコミットがある場合、マイナーバージョンが上がります。
それ以外（バグ修正：fix）のみだと、パッチバージョンが上がります。
:::

## melos publish
パッケージを `pub.dev` に公開するためのコマンドです。
デフォルトで `--dry-run` が有効になっており、実際に公開される前に確認ができます。

確認後、本当に公開する場合は `--no-dry-run` をつけて実行します。

https://melos.invertase.dev/commands/publish

## melos run
`melos.yaml` の `scripts:` に定義したスクリプトを実行します。

以下の定義例では、すべてのパッケージで `dart fix` を実行する `fix` という名前のスクリプトを定義しています。

```yaml:melos.yaml
scripts:
  fix:
    run: melos exec -- dart fix --apply lib
```

このスクリプトを使用する際は、
```shell
melos run fix
```
で実行可能です👌

:::message
`run` は省略することも可能ですが、 `melos analyze` など、ビルトインコマンドと被らないように注意しましょう。
`analyze/format` のように今後新たなビルトインコマンドが増える可能性もあるため、CI/CD環境での実行等では `run` を省略しないことをお勧めします。
:::

https://melos.invertase.dev/commands/run

# Hooks

特定のMelosコマンドは、`melos.yaml` の `command` セクションで `hooks` を定義することで、コマンドの前後にスクリプトを実行させられます。
コマンド実行前に処理を差し込む `pre` と、コマンド実行後に処理を差し込む `post` が用意されており、

特定のコマンドでは他にもフックが用意されています。
対応しているコマンドとフックは以下の通りです。

- bootstrap
  - pre
  - post
- clean
  - pre
  - post
- [version](https://melos.invertase.dev/commands/version#hooks)
  - pre
  - post
  - preCommit
- [publish](https://melos.invertase.dev/commands/publish#hooks) (v5.0.0 で追加)
  - pre
  - post

## CommandでのHooks設定例

```melos.yaml
command:
  bootstrap:
    hooks:
      post: dart pub run build_runner build
  clean:
    hooks:
      post: melos exec --flutter --concurrency=3 -- "flutter clean"
  publish:
    hooks:
      pre: dart pub run build_runner build
      post: dart pub run build_runner clean
```

https://melos.invertase.dev/configuration/scripts#hooks

# Scripts
Scriptを定義することで、 `melos run {script_name}` で実行できるようになります。

https://melos.invertase.dev/configuration/scripts

## スクリプトの名前を決めて定義

最小の構成は以下の通りです。
`{名前}: {実行内容}` で定義します。

```yaml:melos.yaml
scripts:
  hello: echo 'Hello World'
```

実際には、以下の拡張構文を使用することも多いです。

```yaml:melos.yaml
scripts:
  hello:
    name: hello-world
    description: 'My first script'
    run: echo 'Hello $TARGET!'
    env:
      TARGET: 'World'
```

- name: スクリプトのユニークIDです。（使い所が分かりません）
- description: 引数なしでmelos runを使用するときに表示される説明文
- run: 実行されるコマンドを記述します
- env: 環境変数をMapで指定できます

:::message
スクリプトで複数のコマンドを実行する場合、あるコマンドが失敗したときに、それ以降のコマンドを実行しないようにするためには、 `&&` でコマンドを繋いでください。
そうしない場合、いずれかのコマンドが失敗しても、最後までコマンドが実行され、最終的に `SUCCESS` が返却されます。
```yaml:melos.yaml
scripts:
  check:
    run: melos run analyze && melos run format
```
:::

## `exec:` で `melos exec` を実行するスクリプトを定義
`v2.4.0` で `exec` が追加されました。

```yaml:melos.yaml
scripts:
  fix:
    run: dart fix
    exec:
      concurrency: 1
```

のように `exec` オプションを指定することで、 `run: melos exec -- "dart fix"` と同じように実行できます。

また、 `exec` のデフォルトオプションのままでよければ、以下のようにシンプルに記述できます👍

```yaml:melos.yaml
scripts:
  fix:
    exec: dart fix
```

## `packageFilters:` で対象のパッケージを絞り込む

`packageFilters` を使用しないスクリプトでは、認識されているすべてのパッケージを対象として実行されます。

`packageFilters` を使用することで、対象のパッケージを絞り込み、実行時にプロンプトで実行対象パッケージを選択することが可能になります。

下記の例では、 `packageFilters` にて3つの条件を指定しています。

```yaml:melos.yaml
scripts:
  gen:
    run: dart run build_runner build -d
    exec:
      concurrency: 1
    packageFilters:
      flutter: true # Flutter SDKに依存するパッケージのみを対象にする
      dirExists: lib # libディレクトリを持つパッケージのみを対象にする
      dependsOn: build_runner # build_runnerに依存するパッケージのみを対象にする
```

実行時には、以下のように選択肢が表示されます。
1を選択したりEnterキーを押せば、全てのパッケージを対象に実行されます。

```shell
Select a package to run the gen script:

1) * [Default - Press Enter]
2) foo_package
3) bar_package
4) hoge_app
```

:::message
CI上など、プロンプトへの対応ができない環境で、全てへパッケージに対して実行したい場合は `--no-select` を使用しましょう。
プロンプトをスキップして、全てのパッケージに対して実行されます。

```shell
melos run gen --no-select
```
:::

`packageFilters` の各フィルターはキャメルケースで指定します。
コマンドラインオプションではケバブケースで使用している同名オプションをキャメルケースとして使うことができます。

例： `--depends-on` は `dependsOn`

`packageFilters` で使用できる、フィルタリング一覧は、後述の [Filters](#Filters) を参照ください👌

## `steps` で複数のスクリプトを実行するスクリプトを定義

通常のスクリプトの `run` や `exec` でも　`&&` で繋げることで複数のスクリプトを実行するスクリプトを定義することができました。
`v5.2.0` で新たな構文 `steps` が追加され、複数のスクリプトを実行するスクリプトを、より明瞭に定義することができるようになりました。

以下は、 iOSを対象とした `pod:ios` スクリプトと、 macOSを対象とした `pod:macos` スクリプトを順番に実行する `pod` スクリプトを定義した例です。

```yaml:melos.yaml
scripts:
  pod:
    description: Run all pod install.
    steps:
      - echo 'pod install を実行します！'
      - pod:ios
      - pod:macos
      - echo 'pod install を実行しました！'

  pod:ios:
    description: Run pod install on iOS.
    exec: cd ios && pod install
    packageFilters:
      dirExists: [lib, ios]
      fileExists: "ios/Podfile"

  pod:macos:
    exec: cd macos && pod install
    description: Run pod install on macOS.
    packageFilters:
      dirExists: [lib, macos]
      fileExists: "macos/Podfile"
```

:::message
`steps` を使用するスクリプトでは、 execオプションやpackageFiltersオプションは使用できません。
全パッケージで実行されることになるため、 `--no-select` オプションの指定も不要です。

対象のパッケージを絞り込みたい場合は、ステップで実行させる各スクリプト側で指定しましょう。
:::

# Filters
パッケージのフィルタリングオプションです。

`bootstrap` , `clean` , `exec` , `list` , `publish` , `version` コマンドなど、
`run` 以外の主要なコマンドで使用できます。

:::message
コマンドオプションではケバブケースですが、scriptの `packageFilters` ではキャメルケースで使用します。
例： `--depends-on` は `dependsOn`
:::

## `--no-private`
`publish_to: none` が指定してある＝プライベートなパッケージを対象から除外します。
（この指定がない場合は、プライベートなパッケージも含まれます）

使用例：
```shell
melos bootstrap --no-private
```

## `--published`
現在（ローカル）のバージョンが `pub.dev` に存在するパッケージのみに絞り込む。

使用例：
```shell
melos bootstrap --published
```

現在（ローカル）のバージョンが `pub.dev` に存在しないパッケージのみに絞り込みたい場合は `--no-published` が使用できます。

## `--scope`
指定した文字列を含むパッケージのみに絞り込む。

使用例：パッケージ名に `app` を含むもののみ `flutter build ios` を実行する。
```shell
melos exec --scope="*app*" -- flutter build ios
```

## `--ignore`
指定した文字列を含むパッケージを対象から除外する。

使用例：パッケージ名に `example` を含まないもののみ `flutter build ios` を実行する。
```shell
melos exec --ignore="*example*" -- flutter build ios
```

## `--diff`
指定の範囲で差分のあるパッケージのみに絞り込む。

```shell
# 指定のコミットと現在のブランチの差分があるパッケージのみで実行する。
melos exec --diff=<commit hash> -- flutter build ios

# リモートのmainブランチとHEAD間での差分があるパッケージのみで実行する。
melos exec --diff=origin/main...HEAD -- flutter build ios
```

## `--dir-exists`
特定のディレクトリが存在するパッケージのみに絞り込む。

使用例： `example` ディレクトリを持つパッケージのみで実行する。
```shell
melos exec --scope="example" -- flutter build ios
```

## `--file-exists`

使用例： `analysis_options.yaml` ファイルが存在するパッケージのみで実行する。
```shell
melos exec --file-exists="analysis_options.yaml" -- flutter build ios
```

## `--flutter`
Flutter SDKに依存するパッケージのみに絞り込む。

使用例：
```shell
melos exec --flutter -- flutter test
```

Flutter SDKに依存しないパッケージのみに絞り込みたい場合は `--no-flutter` を使用できます。

## `--depends-on`
特定の依存関係を持つパッケージのみに絞り込む。

使用例： `build_runner` に依存するパッケージでのみ実行する。
```shell
melos exec --depends-on="build_runner" -- flutter pub run build_runner build
```

特定の依存関係を持たないパッケージのみに絞り込みたい場合は `--no-depends-on` を使用できます。

# 環境変数

いくつかの環境変数が定義されており、 `melos exec` 内で使用することができます。
例： `MELOS_PACKAGE_NAME` など

またいくつかの環境変数を定義することもできます。
例： `MELOS_SDK_PATH` など

すべての環境変数の一覧は以下のリンクを参照ください👀
https://melos.invertase.dev/environment-variables

# IDE Support
## （VS Code）の拡張機能
VS Code向けに便利なエクステンションが用意されています。

https://melos.invertase.dev/ide-support#vs-code

- `melos.yaml` の検証とオートコンプリート
- `CodeLenses` を介したスクリプト実行できる
- VS Codeの Taskとしてスクリプトを実行できる
- コマンドパレットから、以下のようなMelosコマンドが実行可能にな
  - Melos: Bootstrap
  - Melos: Clean
  - Melos: Run script
  - Melos: Show package graph
- パッケージの依存関係グラフを出力できる

:::message
ただし、新しいバージョンのMelosが出た直後は、拡張機能が対応しておらず望まない警告が表示されてしまうこともありました。
その場合は、拡張機能がアップデートされるまで無効化するなどの対応が必要です。
:::

### task.json
VS CodeのTaskとして使いたいスクリプトは、 `.vscode/tasks.json` を作成してスクリプトを指定します。

```json:tasks.json
{
  "version": "2.0.0",
  "tasks": [
    {
      "type": "melos",
      "script": "analyze",
      "label": "melos: analyze"
    }
  ]
}
```

### パッケージの依存関係グラフを出力できる
中〜大規模なアプリでは、何気にこの機能がありがたいです。パッケージ間がどのような依存関係になっているのかがグラフとして出力できます。

![](/images/melos/package-graph-output.png)

公式ドキュメント例はもっと参考になります↓

https://melos.invertase.dev/ide-support#package-graph

## IntelliJ 
https://melos.invertase.dev/ide-support#intellij

# READMEにバッジを表示する
以下を貼り付けることで、Melosを使用したプロジェクトであることを手軽に示すことができます🌋

```
[![melos](https://img.shields.io/badge/maintained%20with-melos-f700ff.svg?style=flat-square)](https://github.com/invertase/melos)
```

## 宣伝

Altive株式会社では、Flutterアプリの開発・運営を承っております。
お気軽にお問い合わせください👍
https://altive.co.jp/contact

---

Riverpod の実践入門本を公開中です📘
https://zenn.dev/riscait/books/flutter-riverpod-practical-introduction

## 参考リンク
GitHub repository
https://github.com/invertase/melos

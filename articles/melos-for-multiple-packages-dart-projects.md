---
title: "【v1.0.0】Melos紹介&チートシート"
emoji: "🌋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [melos, dart, flutter, cli]
published: true
---

こんにちは、Altive株式会社のFlutterアプリ開発者、村松龍之介（[@riscait](https://twitter.com/riscait)）です😸

この記事は「[Flutter Advent Calendar 2021](https://qiita.com/advent-calendar/2021/flutter)」23日目の記事です。

5日前に `v1.0.0` がリリースされた「Melos」という、
「マルチパッケージを使ったDartプロジェクト」の管理ツールをご紹介します🎉
https://melos.invertase.dev/

:::message
筆者はパッケージ・プラグイン開発というより、Flutterアプリ開発者なため、使いこなせていないMelosの機能も多いです。間違い・不足点等ありましたら、コメントいただけると嬉しいです🙏
:::

# Melosとは？

複数のパッケージを持つDartプロジェクト（モノレポとも呼ばれる）の管理支援CLIツールです。

大規模なプロジェクトをバージョン管理された**パッケージに分割することはコード共有のために非常に有効**です。

しかし、多くのリポジトリにまたがって変更を加えるのは、手間がかかり、追跡が難しく、リポジトリをまたがるテストも複雑になりがちです。

Melosでは、**複数のパッケージが互いに独立しながらも、1つのリポジトリで一緒に作業できる**ように支援してくれます。

また、[Invertase](https://invertase.io/) さんによって構築・メンテナンスが行われています。

## 主な機能
- 自動バージョン管理とチェンジログの作成
- pub.devへのパッケージの自動公開
- ローカルパッケージのリンクとインストール
- 複数のパッケージで同時にコマンドを実行
- ローカルパッケージとその依存関係のリストアップ
- CI/CD環境での自動化支援

## Melosが使われているプロジェクト
Flutterアプリ開発者にとっては馴染みのありそうなリポジトリでMelosは実際に使われています。

以下、いくつかをリストアップします。
それぞれの `melos.yaml` を見るだけでも勉強になりますね👀

- [FirebaseExtended/flutterfire](https://github.com/FirebaseExtended/flutterfire)
- [aws-amplify/amplify-flutter](https://github.com/aws-amplify/amplify-flutter)
- [fluttercommunity/plus_plugins](https://github.com/fluttercommunity/plus_plugins)
- [gql-dart/ferry](https://github.com/gql-dart/ferry)
- [flutter-stripe/flutter_stripe](https://github.com/flutter-stripe/flutter_stripe)
- [rrousselGit/river_pod](https://github.com/rrousselGit/river_pod)

**FlutterFire, PlusPlugins, Riverpod**等々、
個人的に日頃お世話になっているパッケージで使われていました。

## （余談）Flutterモノレポ構成例
以下、Flutterアプリでモノレポと**Melos**を採用する場合のリポジトリ（ディレクトリ）構成の一例です。

```
- packages/
  - a_package
    - lib/
      - src/
      - a_package.dart
    - pubspec.yaml
  - b_package
    - lib/
      - src/
      - b_package.dart
    - pubspec.yaml
  - c_app
    - lib
      - src/
      - main.dart
    - pubspec.yaml
- analysis_opsions.yaml
- .gitignore
- melos.yaml
```

ルートに、 `melos.yaml` や `.gitignore` , `analysis_opsions.yaml` ファイルを置きます。
これで各パッケージから共通の `analysis_options.yaml` 等を使用できます🙆‍♂️

:::message
FVMもモノレポに対応しているため、 `.fvm/fvm_config.json` 等もルートに配置するだけでOKです。
:::

上の例でいうと、 `c_app` の `pubspec.yaml` の `dependencies:` にて依存するパッケージをパス指定しましょう。

```yaml:c_app/pubspec.yaml
dependencies:
  a_package:
    path: ../a_package/
  b_package:
    path: ../b_package/
```

モノレポ余談おわり。

# Melosの導入
## インストール
以下のコマンドで pub.dev からグローバルなパッケージとしてインストールしましょう。

```shell
dart pub global activate melos
```

インストールはマシーンごとに1回のみの実行でOKです。

## melos.yaml ファイルの作成
Melosを使用したいプロジェクトディレクトリのルートに `melos.yaml` ファイルを新規作成します。

まずは以下のように `name` と `packages` フィールドを追加しましょう。

```yaml:melos.yaml
name: my_project
packages:
  - packages/**
```

`packages` でプロジェクト内のパッケージ位置を指定します。
上記例では、プロジェクトルートに `packages` というディレクトリを作成して、その中に各パッケージを配置しています。

パスの指定には、 `.gitignore` でも使われている [glob](https://docs.python.org/3/library/glob.html)パターンを使用できます。

これで Melos を使用する準備は整いました…！

詳しい melos.yaml の記法は後述します。

# Melosで使えるコマンド

## melos bootstrap コマンド
Melosのインストールと `melos.yaml` の作成が完了したら、次は `Bootstrap` を実行しましょう。

- 全パッケージの依存関係がインストールされます（内部的には `pub get` が使われます）
- 全パッケージがローカルにリンクされます。

```shell
melos bootstrap
# または
melos bs
```

### Bootstrapが必要な理由

通常のプロジェクトでは、pubspec.yaml 内でパスを指定することでパッケージをリンクすることができます。
小規模なプロジェクトや公開しないパッケージでは有効ですが、
そうではない場合、パッケージの公開時に手動ですべてのパッケージのpubspec.yamlファイルをバージョンで更新する必要が出てきます。

また、パッケージが密結合（互いに依存している）している場合、どのバージョンを更新すべきかを手動でチェックする必要があり、煩雑です。

逆にいうと、公開する必要のないパッケージの場合は、 `pubspec.yaml` でのパス指定で問題ないため、 `Bootstrap` は必要ないとも言えます。

Bootstrapコマンドについてより詳しくはこちら↓
https://melos.invertase.dev/commands/bootstrap

## melos clean コマンド

プロジェクトの一時ファイルをクリーンアップします。
`bootstrap` の前に実行するなどして完全に新しい状態でスタートできます。

また、たとえばHooksの `postclean` で追加で削除したいファイルを指定することもできます。

```yaml:melos.yaml
scripts:
  postclean: rm packages/mypackage/lib/src/generated_file.g.dart
```

`flutter crean` もすべてのパッケージで一緒に実行させる例

```yaml:melos.yaml
scripts:
  postclean: melos exec --flutter -- "flutter clean"
```

Cleanコマンドについてより詳しくはこちら↓
https://melos.invertase.dev/commands/clean

## melos exec コマンド
`exec` コマンドは、すべてのパッケージに対して任意のコマンドを実行するコマンドです。

```shell
# すべてのパッケージで依存関係を取得するコマンドを実行する
melos exec flutter pug get
```

実際は、execコマンドを直接Terminal等で使うというよりは、
後述の `scripts:` に `melos exec` を使用したスクリプトを定義し、
`melos run` コマンドで呼び出す用途が多くなるかと思います。

### concurrency (-c) オプション
同時にコマンドを実行するパッケージの最大数を定義します。デフォルトは5です。
```shell
# パッケージ1つずつ順番でコード生成を実行させたい例
melos exec -c 1 -- "flutter pub run build_runner build"
```

### fail-fast オプション
いずれかのパッケージでスクリプトが失敗した場合に、それ以降のパッケージでスクリプトを実行しないようにします。
デフォルトはfalseです。

```shell
melos exec --fail-fast -- "dart analyze ."
```

## melos run コマンド
`scripts:` に定義したスクリプトを実行します。

以下の定義例では、すべてのパッケージで `dart analyze` を実行する `analyze` という名前のスクリプトを定義しています。

```yaml:melos.yaml
scripts:
  analyze: melos exec -- "dart analyze ."
```

このスクリプトを使用する際は、
`melos run analyze` で実行可能です👌

# melos.yaml ファイルの構成フィールド
## name （必須）
プロジェクトの名前、IDE等での表示目的で使用される。

## repository
プロジェクトがホストされているURL
例: `repository: repository: https://github.com/invertase/melos`

## packages （必須）
各パッケージのリスト。特定のパスか glob パターン拡張形式で指定できる

```yaml:melos.yaml
packages:
  # packagesディレクトリ以下の全てのパッケージを指定する場合の例
  - packages/**
```

## ignore
除外したいパッケージのリスト
```yaml:melos.yaml
ignore:
  # exampleパッケージを除外したい場合の例
  - "packages/**/*example"
```

## scripts
`melos run {script_name}` で実行できるスクリプトを定義する。

# Scripts定義
```yaml:melos.yaml
scripts:
  myscript:
    name: myscript01
    description: 'My first script'
    run: echo 'OH! $GREETING World!'
    env:
      GREETING: 'Hello'
```

## name
スクリプトのユニークIDです。

## description
引数なしでmelos runを使用するときに表示される説明文。

## run
実行されるコマンドを記述します。

## env
実行されたコマンドに渡される環境変数のMap。

## select-package
```yaml:melos.yaml
scripts:
  gen:
    description: 'Generate codes By Build_runner'
    run: melos exec -- "flutter pub run build_runner build"
    select-package:
      flutter: true
```

`select-package` で使用できるフィルタリングについては
後述の [Filtering](#Filtering) を参照ください👌

## Hooks
特定のMelosコマンドは、コマンド実行前後のスクリプト実行が可能です。
コマンド実行前に追加したい処理の名前はコマンドと同じものを、
コマンド実行後に追加したい処理はコマンド名の前に `post` を付けたもので指定可能です。

```yaml:melos.yaml
scripts:
  bootstrap: echo `Melos bootstrap command has been started...`
  postbootstrap: echo `Melos bootstrap command completed.`
```

現在は以下のコマンドがHookをサポートしています。
- bootstrap
- clean
- version

# パッケージのFiltering(グローバルオプション)
パッケージのフィルタリングオプションです。

`bootstrap` , `clean` , `exec` , `list` , `publish` , `version` コマンド等、
`run` 以外の主要なコマンドで使用できます。

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

## `--no-published`
現在（ローカル）のバージョンが `pub.dev` に存在しないパッケージのみに絞り込む。

## `--scope`
指定した文字列を含むパッケージのみに絞り込む。

使用例：パッケージ名に `app` を含むもののみ `flutter build ios` を実行する。
```shell
melos exec --scope="*app*" -- flutter build ios
```

## `ignore`
指定した文字列を含むパッケージを対象から除外する。

使用例：パッケージ名に `example` を含まないもののみ `flutter build ios` を実行する。
```shell
melos exec --ignore="*example*" -- flutter build ios
```

## `since`
指定した Commit Hash や Git tag 以降に変更があるパッケージのみに絞り込む。

使用例：
```shell
melos exec --since="v1.0.0" -- flutter build ios
```

## `dir-exists`
特定のディレクトリが存在するパッケージのみに絞り込む。

使用例： `example` ディレクトリを持つパッケージのみで実行する。
```shell
melos exec --scope="example" -- flutter build ios
```

## `file-exists`

使用例： `analysis_options.yaml` ファイルが存在するパッケージのみで実行する。
```shell
melos exec --file-exists="analysis_options.yaml" -- flutter build ios
```

## `flutter`
Flutter SDKに依存するパッケージのみに絞り込む。

使用例：
```shell
melos exec --flutter -- flutter test
```

## `no-flutter`
Flutter SDKに依存しないパッケージのみに絞り込む。

使用例：
```shell
melos exec --no-flutter -- dart analyze
```

## `depends-on`
特定の依存関係を持つパッケージのみに絞り込む。

使用例： `build_runner` に依存するパッケージでのみ実行する。
```shell
melos exec --depends-on="build_runner" -- flutter pub run build_runner build
```

## `no-depends-on`
特定の依存関係を持たないパッケージのみに絞り込む。

使用例： `firebase_core` に依存しないパッケージでのみ実行する。
```shell
melos exec --no-depends-on="firebase_core" -- flutter build ios
```

使用例：
```shell
melos exec --scope="*app*" -- flutter build ios
```

# IDE（VS Code）の拡張機能
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

## task.json
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

## パッケージの依存関係グラフを出力できる
中〜大規模なアプリでは、何気にこの機能がありがたいです。パッケージ間がどのような依存関係になっているのかがグラフとして出力できます。

![](/images/melos/package-graph-output.png)

↑出力例がしょぼくて参考になりませんね💦公式ドキュメント例はもっと参考になります↓

https://melos.invertase.dev/ide-support#package-graph

:::message
2021年12月23日時点では、 `IntelliJ` の拡張機能は *TODO* となっています。
:::

# READMEにバッジを表示する
以下を貼り付けることで、Melosを使用したプロジェクトであることを手軽に示すことができますね🌋

```
[![melos](https://img.shields.io/badge/maintained%20with-melos-f700ff.svg?style=flat-square)](https://github.com/invertase/melos)
```

## リンク
GitHub repository
https://github.com/invertase/melos
---
title: "altive_lintsのcustom_lint製ルールとアシストをAnalyzer Pluginsに移行してみた"
emoji: "🧵"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["dart", "flutter"]
published: true
# cSpell:ignore remi yniji
---

:::message
この記事は「[Flutter Advent Calendar 2025](https://qiita.com/advent-calendar/2025/flutter)」6日目の記事です🎄

5日目： [[AI x Flutter] Claudeと伴にFlutterアプリを開発してAIがどこまでできるか試してみた](https://qiita.com/yniji/items/c9739de29d9b3a53b05a) by [yniji](https://twitter.com/creativewebjp))さん
6日目： altive_lintsでcustom_lint製カスタムルールをanalysis_server_pluginへ移した理由と工夫 by [村松龍之介](https://x.com/riscait) ← 今ここ
7日目： 「タイトル未定」 by [朝日 大樹さん](https://twitter.com/daiki1003)さん
:::

こんにちは、Flutterでのアプリ開発をメインとしている「[Altive株式会社](https://altive.co.jp)」の村松龍之介（[@riscait](https://x.com/riscait)）です！

![](/images/ProfileBanner_Muramatsu.jpg)

この記事では、altive_lints で提供しているカスタムルール／アシストを custom_lint から analysis_server_plugin を使った Analyzer Plugins へ移行した背景と手順をまとめました。

# ✍️ はじめに

オルティブ株式会社は [altive_lints](https://pub.dev/packages/altive_lints) というリントパッケージを公開しています。

Dart が提供する標準ルールに加えて custom_lint ベースの独自ルールやアシストも同梱していました。どんな内容か簡単に以下にまとめます。

# 📝 altive_lintsのルールとアシスト一覧

## カスタムリントルール

- `avoid_consecutive_sliver_to_box_adapter`: `SliverToBoxAdapter` の連続使用を避ける。
- `avoid_hardcoded_color`: ハードコーディングされた `Color` を検出する。
- `avoid_hardcoded_japanese`: ハードコーディングされた日本語文字列を検出する。
- `avoid_shrink_wrap_in_list_view`: `ListView` 内での過剰な `shrinkWrap` を防ぐ。
- `avoid_single_child`: 子要素が 1 つだけの `MultiChildRenderObjectWidget` を検出する。
- `prefer_clock_now`: `DateTime.now()` の代わりに `clock.now()` を推奨する。
- `prefer_dedicated_media_query_methods`: `MediaQuery` の専用メソッド利用を促す。
- `prefer_space_between_elements`: 行間に空行を挟んで読みやすさを保つ。
- `prefer_to_include_sliver_in_name`: `Sliver` を返す Widget 名に `Sliver` を含める。

詳しくは弊社小林さんの以下の記事をご覧ください！
https://zenn.dev/altiveinc/articles/altive-custom-lint

## Quick fixで使えるアシスト機能

- `add_macro_template_documentation`: クラス定義に Macro テンプレート付き Doc コメントを挿入。
- `add_macro_documentation_comment`: コンストラクタやメソッドに Doc コメントを挿入。
- `wrap_with_macro_template_documentation_comment`: 既存 Doc コメントを Macro テンプレートで包む。

詳しくは弊社小林さんの以下の記事をご覧ください！
https://zenn.dev/altiveinc/articles/dartdoc-macros

今回はこれらのルールとアシスト機能の実装を `custom_lint` から `analysis_server_plugin` に書き換えてみた話です。

# 📂 用語整理

| パッケージ名 | レイヤー / 分類 | 実装時の関わり方 |
| :-- | :-- | :-- |
| **Analyzer Plugin** | **概念・アーキテクチャ** | Analysis Server が外部機能を実行できる仕組みそのものの呼び名。 |
| **analyzer_plugin** | **型定義 & プロトコル** | プラグイン実装に必要なデータ型やユーティリティを提供するパッケージ。 |
| **custom_lint** | **サードパーティ製フレームワーク** | Remi 氏 ([Invertase](https://invertase.io/)) が提供。DX が高く、テストメカニズムも備わっている。 |
| **analysis_server_plugin** | **公式実装** | Dart チーム提供の標準プラグイン基盤。`Plugin` クラス等を備え、Dart 3.10+ が必要。 |

:::message
呼称が似ていて混乱しやすいので、個人メモとして整理しました。
:::

# 🤔 custom_lint から analysis_server_plugin へ移行するかの判断

- 新しいAnalyzer Pluginが開発されるときに、「（Analyzer Pluginが）数ヶ月でリリースされるなら custom_lint の積極的な開発に取り掛からない」という発言を見ました。
その後、「数ヶ月後には公開予定だよ」という旨の返答があったのを見ました。（要出典！）
- [riverpod_lint](https://pub.dev/packages/riverpod_lint) も custom_lint から analysis_server_plugin へ移行するためのプルリクエストがDraftですが存在します。（[Migrate away from custom_lint](https://github.com/rrousselGit/riverpod/pull/4413)）
- custom_lint を使ったパッケージでカスタムリントルールを提供した場合、利用側（アプリ等）でも custom_lint に依存する必要があります。
analysis_server_plugin製であれば、それは不要です。

こうした背景から、altive_lints も公式プラグインベースへ移行し、依存関係をシンプルに保つ方針に決めました。

# 📦 analysis_server_plugin でプラグインを作成する（カスタムルール・アシスト）

ここからは、altive_lints のルール／アシストを analysis_server_plugin ベースへ移行する際に実施した手順を順に紹介します。

## ① 📥 依存パッケージのインストール

`custom_lint` と `custom_lint_builder` を削除し、 `analysis_server_plugin` などを追加しました。

```yaml:pubspec.yaml
environment:
  sdk: ^3.10.0

dependencies:
  analysis_server_plugin: ^0.3.0
  analyzer: ">=8.0.0 <10.0.0"
  analyzer_plugin: ^0.13.11
  collection: ^1.19.1

dev_dependencies:
  analyzer_testing: ^0.1.7
  test: ^1.26.2
  test_reflective_loader: ^0.4.0
```

### [analysis_server_plugin](https://pub.dev/packages/analysis_server_plugin)

> This package offers support for writing Dart analysis server plugins. Analysis server plugins empower developers to contribute their own Dart static analysis in IDEs and at the command line via dart analyze and flutter analyze.

IDE／CLI 双方で動く静的解析プラグインを公式サーバー上で動かすためのライブラリです。ルール（AnalysisRule）、Fixes、Assist をここで提供される `Plugin` クラスを介して登録します。

### テスト系パッケージ

ルールやアシストの振る舞いを自動テストしたかったので、下記 2 つも採用しました。

- [analyzer_testing](https://pub.dev/packages/analyzer_testing): analyzer／analysis_server_plugin 系のテストユーティリティを提供。
- [test_reflective_loader](https://pub.dev/packages/test_reflective_loader): リフレクションでテストスイートを検出・実行できる。

この組み合わせにより、ルールを単体テストに近い書き味で検証できています。

## ② 🔨 main.dart で Plugin を作成

まずは、後ほど作成するルールやアシスト機能を登録する土台となるプラグインを用意します。

custom_lint では `altive_lints.dart`（パッケージ名.dart）を置いていましたが、analysis_server_plugin ではトップレベルで `plugin` を公開する `main.dart` を作るスタイルです。そのため、新たに `main.dart` を追加しました。

```dart:main.dart
import 'dart:async';

import 'package:analysis_server_plugin/plugin.dart';
import 'package:analysis_server_plugin/registry.dart';

// トップレベルのプロパティを定義することでプラグインが提供できます。
final plugin = _Plugin();

// `Plugin` クラスを継承して、プラグインを定義します。
class _Plugin extends Plugin {
  // プラグインの名前を決めましょう。
  @override
  String get name => 'altive_lints';

  // `register` メソッドをオーバーライドして、後ほど作成するルールやアシストを登録します。
  @override
  Future<void> register(PluginRegistry registry) async {
    // 後ほど作ったルールやアシストをここで登録します。
  }
}
```

## ③ 📏 AnalysisRule の作成

ルールやアシストを登録するためのプラグインは作成できたので、次はルールを作成します。

altive_lints に実在する `prefer_clock_now` というルールを例に解説します。

```dart
// `AnalysisRule` クラスを継承して、ルールを定義します。
class PreferClockNow extends AnalysisRule {
  // コンストラクタはパラメータ無しでOKです。
  // super() を使って親クラスに name, description を渡します。
  PreferClockNow() : super(name: _code.name, description: _code.problemMessage);

  // `LintCode` クラスを定義して、ルールの名前と説明を定義します。
  static const _code = LintCode(
    'prefer_clock_now',
    'Avoid using DateTime.now(). '
        'Use a testable alternative like clock.now() or similar instead.',
  );

  // `DiagnosticCode` をオーバーライドして `LintCode` を返します。
  @override
  DiagnosticCode get diagnosticCode => _code;

  // `registerNodeProcessors` メソッドをオーバーライドして、実装内容を登録します。
  // Visitorクラスは別途定義します。
  @override
  void registerNodeProcessors(
    RuleVisitorRegistry registry,
    RuleContext context,
  ) {
    final visitor = _Visitor(this, context);
    registry.addInstanceCreationExpression(this, visitor);
  }
}

// AST 上で指定したノードだけを見てくれる `SimpleAstVisitor` を継承したクラスを定義します。
// Visitor は Dart コードの構文木を歩き回って探索してくれる「訪問者」です。
class _Visitor extends SimpleAstVisitor<void> {
  _Visitor(this.rule, this.context);

  final AnalysisRule rule;
  final RuleContext context;

  @override
  void visitInstanceCreationExpression(InstanceCreationExpression node) {
    // コンストラクタの型と名前を取得し、`DateTime.now` なら report します。
    final constructorName = node.constructorName;
    final type = constructorName.type.name.lexeme;
    if (type != 'DateTime') {
      return;
    }
    if (node.constructorName.name?.name == 'now') {
      rule.reportAtNode(node);
    }
  }
}
```

### Plugin に 作成した AnalysisRule を登録

作ったルールをプラグインに登録しましょう。

ルールの登録方法として、以下の２種類があります。

- `registry.registerWarningRule` 
- `registry.registerLintRule` 

```dart:main.dart
@override
Future<void> register(PluginRegistry registry) async {
  registry.registerLintRule(PreferClockNow());
}
```

`registerWarningRule` は常時オンの「警告」として登録され、利用側で個別に切り替える術がありません。
一方で `registerLintRule` は opt-in の「lint」として登録され、利用側が `diagnostics` でオン／オフできます。

:::message
最初は `registerWarningRule` を使っていましたが、利用側で無効化できず（`false` 指定も効果なし）詰まりました。
`registerLintRule` に切り替えたところ、使う／使わないを明示できるようになりました。
:::

### 登録した AnalysisRule を明示的に有効化する

`registerLintRule` で登録すると利用側が明示的に有効化するまで動きません。
`include: package:altive_lints/altive_lints.yaml` を書くだけで全部使えるようにしたかったので、`altive_lints.yaml` 内で true 指定しています。

```yaml:altive_lints.yaml
plugins:
  altive_lints:
    path: ../../altive_lints
    diagnostics:
      avoid_consecutive_sliver_to_box_adapter: true
      avoid_hardcoded_color: true
      avoid_hardcoded_japanese: true
      avoid_shrink_wrap_in_list_view: true
      avoid_single_child: true
      prefer_clock_now: true
      prefer_dedicated_media_query_methods: true
      prefer_space_between_elements: true
      prefer_to_include_sliver_in_name: true
```

## ④ 🫂 Assist （Correction） の作成

次はアシスト機能の作成です。`Quick Fix` メニューに候補が表示され、選択するとコードを挿入したり書き換える機能を提供できます。

ここでは、コンストラクタやメソッドに Doc コメントを差し込む `add_macro_document_comment` を例にします。

```dart
// `ResolvedCorrectionProducer` クラスを継承して、アシストを定義します。
class AddMacroDocumentComment extends ResolvedCorrectionProducer {
  // ルールとは異なり `context` をコンストラクタパラメータとして受け取ります。
  // 直接利用はしないので `super` に渡せばOKです。
  AddMacroDocumentComment({required super.context});

  // AssistKindクラスで、ID、優先度、メッセージを定義します。
  static const _kind = AssistKind(
    'dart.assist.addMacroDocumentComment',
    DartFixKindPriority.standard,
    'Add a macro document comment',
  );

  @override
  CorrectionApplicability get applicability => .singleLocation;

  @override
  AssistKind get assistKind => _kind;

  @override
  Future<void> compute(ChangeBuilder builder) async {
    // computeメソッド内でQuick Fixメニューに当アシストを表示する判定ロジックや、
    // 適用する修正を定義します。（長いので割愛します）
  }
}
```

### Plugin に Assist を登録

アシストは `registry.registerAssist` で登録します。

```dart:main.dart
@override
Future<void> register(PluginRegistry registry) async {
  registry.registerLintRule(SomeAnalysisRule());

  registry.registerAssist(SomeAssist.new); // 追加
}
```

`registerAssist` は `{required CorrectionProducerContext context}` を受け取るファクトリ関数を渡す仕様なので、アシストクラスのコンストラクタも `context` を受ける形にして `.new` をそのまま渡しました。

## ⑤ 💡 Plugin の有効化

[登録したルールを明示的に有効化する](#登録したルールを明示的に有効化する) にて、プラグイン自体も有効化済みですが、
プラグインを有効化するためには以下の記述が必要なので追加してあります。

```yaml:altive_lints.yaml
plugins:
  altive_lints:
    path: ../../altive_lints
```

# 🤝 Plugin をアプリやパッケージから利用する方法

## pubspec.yaml に altive_lints を追加

```yaml:pubspec.yaml
environment:
  sdk: ^3.10.0 # altive_lintsを含む Analyzer Plugin の利用には Dart 3.10 以上が必要です。

dev_dependencies:
  altive_lints: ^2.0.0-dev.2
```

custom_lint を直接入れる必要がなくなり、`dev_dependencies` は altive_lints だけで済むようになりました。

## ➕ analysis_options.yaml で altive_lints をインクルード

```yaml:analysis_options.yaml
include: package:altive_lints/altive_lints.yaml
```

`altive_lints.yaml` にはプラグイン有効化とルール有効化済みなので、利用側では `include` するだけで OK です。

逆に、Plugin だけ使いたい／設定を細かく書きたい場合は `plugins` セクションに `altive_lints` を追加します。

```yaml:analysis_options.yaml
include: package:flutter_lints/flutter_lints.yaml
plugins:
  altive_lints:
```

## 🔕 ルールを個別に無効化する方法

altive_lints のカスタムルールで無効化したいものがあれば、`diagnostics` で `false` を指定します。

```yaml:analysis_options.yaml
include: package:altive_lints/altive_lints.yaml

plugins:
  altive_lints:
    diagnostics:
      avoid_hardcoded_japanese: false
```

## 🙈 ファイルやコード単位でルールをignoreする方法

通常の lint と同様に `ignore` コメントも使えますが、`プラグイン名/ルール名` 形式で書きます。

```dart
// ignore: altive_lints/prefer_clock_now

// ignore_for_file: altive_lints/prefer_clock_now
```

複数のプラグインを使っていたら名前の重複もあり得るからですね。

# 🚀 Analyzer Plugin版 altive_lints プレリリース公開中

まずはプレリリース版として公開しました。ぜひ触ってフィードバックいただけると嬉しいです🚀

[altive_lints 2.0.0-dev.2](https://pub.dev/packages/altive_lints/versions/2.0.0-dev.2)

# ✍️ おわりに

今回の移行で Rules と Assists は一通り移せたものの、`Fixes`（自動修正）はまだ手付かずです。
`DateTime.now()` を `clock.now()` に差し替えたり、要素が 1 つしかない `Column` をRemoveしたりと、Fixes 化できそうな題材は多いので順次チャレンジ予定です。

Analyzer Plugin を触るのはほぼ初めてだったため、AI に相談しながら試行錯誤しました。まだ改善できる余地が多いと思うので、引き続き altive_lints を育てていきます。

最後までご覧いただきありがとうございました！😊

## 📚 関連リンク

### altive_lints analysis_server_plugin移行プルリクエスト

- [feat: Migrate to analysis_server_plugin from custom_lint](https://github.com/altive/altive_lints/pull/119)
- [fix: Change AnalysisRule from Warning to Lint](https://github.com/altive/altive_lints/pull/120)

### 公式・パッケージ
- [Analyzer Plugins](https://dart.dev/tools/analyzer-plugins)
- [analysis_server_plugin](https://pub.dev/packages/analysis_server_plugin)
- [analyzer](https://pub.dev/packages/analyzer)
- [analyzer_testing](https://pub.dev/packages/analyzer_testing)
- [test_reflective_loader](https://pub.dev/packages/test_reflective_loader)

---
title: "custom_lintからanalysis_server_pluginへ書き換えてみた [altive_lints]"
emoji: "🧵"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["dart", "flutter"]
published: true
---

こんにちは、Flutterでのアプリ開発をメインとしている「[Altive株式会社](https://altive.co.jp)」の村松龍之介（[@riscait](https://x.com/riscait)）です！

![](/images/ProfileBanner_Muramatsu.jpg)

# はじめに

弊社は [altive_lints](https://pub.dev/packages/altive_lints) というリントパッケージを公開しています。 

Dartが提供している通常のリントルールの他に、独自のルールを custom_lint で作成して提供していました。

参考までに以下のルールがあります。

:::details altive_lintsのルールとアシスト一覧

**カスタムリントルール**

- avoid_consecutive_sliver_to_box_adapter
- avoid_hardcoded_color
- avoid_hardcoded_japanese
- avoid_shrink_wrap_in_list_view
- avoid_single_child
- prefer_clock_now
- prefer_dedicated_media_query_methods
- prefer_space_between_elements
- prefer_to_include_sliver_in_name

**Quick fixで使えるアシスト機能**

- add_macro_template_documentation
- add_macro_documentation_comment
- wrap_with_macro_template_documentation_comment
:::

今回はこれらのルールとアシスト機能を custom_lint から analysis_server_plugin に書き換えてみた話です。

# 用語整理

| パッケージ名               | レイヤー / 分類                       | 実装時の関わり方（重要）                                                                                                                                                                                                                                   |
| :------------------------- | :------------------------------------ | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Analyzer Plugin**        | **概念・アーキテクチャ**              | 「Analysis Serverが外部機能を動かす仕組み」そのものの名前。                                                                                                                                                                                                |
| **analyzer_plugin**        | **共通言語 (型定義)**<br>& プロトコル | **【必須】 データ型を使う。**<br>`analyzer_plugin/utilities` にある型定義等をプラグインを作る際にインポートして使う。                                                                                                                                      |
| **custom_lint**            | **高機能ライブラリ**                  | RiverpodやFreezedの作者であるRemi氏 ([Invertase](https://invertase.io/)) が作成したパッケージ。<br>analyzer_pluginに似ているが、より優れた開発者エクスペリエンスを提供することを目指したサードパーティーパッケージ。<br>組み込みのテストメカニズムもある。 |
| **analysis_server_plugin** | **公式標準実装**                      | **公式標準ツール** `Plugin`クラスなどを提供する。Dart 3.10以上が必要                                                                                                                                                                                       |

:::message
自身、理解を整理するために表にしましたが、いまいち自信がありません…
:::

# custom_lint から analysis_server_plugin へ移行するかの判断

- 新しいAnalyzer Pluginが開発されるときに、「（Analyzer Pluginが）数ヶ月でリリースされるなら custom_lint の積極的な開発に取り掛からない」という発言を見ました。
その後、「数ヶ月後には公開予定だよ」という旨の返答がありました。
- [riverpod_lint](https://pub.dev/packages/riverpod_lint) も custom_lint から Analyzer Plugin へ移行するためのプルリクエストがDraftですが存在します。（[Migrate away from custom_lint](https://github.com/rrousselGit/riverpod/pull/4413)）
- custom_lint を使ったパッケージでカスタムリントルールを提供した場合、利用側（アプリ等）でも custom_lint に依存する必要があります。
analysis_server_plugin製であれば、それは不要なのでありがたいですね。
`analyzer` のバージョンを他のパッケージと揃えたりといったことは引き続き必要ですが…

# analysis_server_plugin でプラグインを作成する（カスタムルール・アシスト）

というわけで、 altive_lints は custom_lint から analysis_server_plugin へ移行してみることにしました！

## 依存パッケージのインストール

### pubspec.yaml

`custom_lint` と `custom_lint_builder` を削除し、 `analysis_server_plugin` などを追加しました。

```yaml
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

> This package offers support for writing Dart analysis server plugins.
Analysis server plugins empower developers to contribute their own Dart static analysis in IDEs and at the command line via dart analyze and flutter analyze. Analysis server plugins can offer the following static analyses:
このパッケージは、Dart analysis server plugin の作成を支援します。
開発者は IDE 内やコマンドラインの dart analyze や flutter analyze を通じて独自の「Dart静的解析」を提供できます。

このパッケージを使って、Rule, Fixes, Assist をPluginとして作成します。

### テスト系パッケージ

可能な限り単体テストも書きたかったので以下のパッケージを追加しました。

- [analyzer_testing](https://pub.dev/packages/analyzer_testing)
- [test_reflective_loader](https://pub.dev/packages/test_reflective_loader)

## main.dartでプラグインを作成

まずは、後ほど作成するルールやアシスト機能を登録し、提供するためのプラグインを作成します。

custom_lint では `altive_lints.dart` のようにパッケージと同名のDartファイルを作成しましたが、
analysis server pluginでは `main.dart` でプラグインを作成します。

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

## ルール（AnalysisRule）の作成

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

// ここでは、指定したノードだけを見てくれる `SimpleAstVisitor` を継承したクラスを定義します。
// Visitor は Dartコードのツリー構造を歩き回って探索してくれる「訪問者」
class _Visitor extends SimpleAstVisitor<void> {
  _Visitor(this.rule, this.context);

  final AnalysisRule rule;
  final RuleContext context;

  @override
  void visitInstanceCreationExpression(InstanceCreationExpression node) {
    // node のコンストラクタの型と名前を取得し、 'DateTime' + `now` だった場合に報告します。
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

### プラグインにルールを登録

前項で作ったルールをプラグインに登録しましょう。

登録方法として、 `registry.registerWarningRule` と `registry.registerLintRule` があります。

```dart:main.dart
@override
Future<void> register(PluginRegistry registry) async {
  registry.registerLintRule(PreferClockNow());
}
```

`registerWarningRule` で登録すると警告ルールとなり、デフォルトで有効となります。今の所利用側で無効にする術もなさそうです。

`registerLintRule` で登録するとリントルールとなり、デフォルトでは無効となります。

:::message
最初は `registerWarningRule` でAnalysisRuleを登録していましたが、プラグイン利用側でルールを個別に無効化できず、（falseにしても効果なし）
少々つまりました。 `registerLintRule` で登録することで、プラグイン利用側で個別に無効化できるようになりました。
:::

### Ruleを明示的に有効化する

前項でルールを `registerLintRule` で登録したことにより、デフォルトでは無効となっています。
アプリ等のプラグイン利用側で `include: package:altive_lints/altive_lints.yaml` を指定するだけで、ルールが使えるようにしたいです。

なので、下記のように `altive_lints.yaml` でルールを明示的に有効化しました。

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

## アシスト（ResolvedCorrectionProducer）の作成

次はアシストを作成します。
Quick Fixで候補が出てきて、それを適用することでコードに修正を加えることができる機能です。

`add_macro_document_comment` という、コンストラクタやメソッドにDoc Commentを挿入するアシストを例に説明します。

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

## プラグインにアシストを登録

アシストは `registry.registerAssist` メソッドで登録します。

```dart:main.dart
@override
Future<void> register(PluginRegistry registry) async {
  registry.registerLintRule(SomeAnalysisRule());

  registry.registerAssist(SomeAssist.new); // 追加
}
```

`registerAssist` メソッドは、
`Function({required CorrectionProducerContext context}) generator)`　というパラメータを持っています。

なので前項で定義したアシストクラスではパラメータで context を受け取るように定義したのでした。

アシストクラスのコンストラクタ引数にそのまま渡せば良いので `.new` を使っています。

## プラグインの有効化

[Ruleを明示的に有効化する](#ruleを明示的に有効化する) にて、プラグイン自体も有効化済みですが、
プラグインを有効化するためには以下の記述が必要なので追加してあります。

```yaml:altive_lints.yaml
plugins:
  altive_lints:
    path: ../../altive_lints
```

## アプリやパッケージからの利用

### pubspec.yaml に altive_lints を追加

```yaml:pubspec.yaml
environment:
  sdk: ^3.10.0 # altive_lintsを含む Analyzer Plugin の利用には Dart 3.10 以上が必要です。

dev_dependencies:
  altive_lints: ^2.0.0-dev.2
```

`dev_dependencies` には altive_lints のみ追加すれば良くなったのでシンプルになりました。

### analysis_options.yaml で altive_lints をインクルード

```yaml:analysis_options.yaml
include: package:altive_lints/altive_lints.yaml
```

altive_lints.yaml にて、プラグインを有効化してあるので、利用側の analysis_options.yaml 側では不要です。

逆にいうと、`altive_lints` を include せずPluginだけ使いたい場合は、以下のようにplugins指定が必要かと思われます。

例：
```yaml:analysis_options.yaml
include: package:flutter_lints/flutter_lints.yaml
plugins:
  altive_lints:
```

### ルールを個別に無効化する方法

altive_lintsのカスタムルールの内、無効化したいルールがある場合は、以下のようにfalse指定で無効化できます。

```yaml:analysis_options.yaml
include: package:altive_lints/altive_lints.yaml

plugins:
  altive_lints:
    diagnostics:
      avoid_hardcoded_japanese: false
```

### ファイルやコード単位でルールをignoreする方法

通常のリントルールと同じく、ラインごとやファイルごとに無効化（ignore）できます。

ただし、 `プラグイン名/ルール名` の形式で指定する必要があります。

```dart
// ignore: altive_lints/prefer_clock_now

// ignore_for_file: altive_lints/prefer_clock_now
```

複数のプラグインを使っていたら名前の重複もあり得るからですね。

# Analyzer Plugin版 altive_lints プレリリース版公開中

まずはPrerelease版として公開したので、もし良かったら使っていただければ嬉しいです🚀

[altive_lints 2.0.0-dev.2](https://pub.dev/packages/altive_lints/versions/2.0.0-dev.2)

# おわりに

`Rules` , `Assists` は今回作成できましたが、 `Fixes` タイプは作れなかったので作ってみたいです。
今回移行した `Rules` の中にも `Fixes` タイプで作れるものがありそうなので挑戦したいです。
`DateTime.now()` を `clock.now()` に置き換えたり、要素が1つしかない `Column` を削除したり。

Analyzer Pluginを触るのはほぼ初めてだったので、AIに助けてもらい勉強しながら、custom_lint から移行しました。
今回実装したルールも、より良い書き方が絶対あると思うので改善しながらFixesやAssists含めて追加していきたいです。

最後までご覧いただきありがとうございました！😊

## 関連リンク

### altive_lints analysis_server_plugin移行プルリクエスト

- [feat: Migrate to analysis_server_plugin from custom_lint](https://github.com/altive/altive_lints/pull/119)
- [fix: Change AnalysisRule from Warning to Lint](https://github.com/altive/altive_lints/pull/120)

### 公式・パッケージ
- [Analyzer Plugins](https://dart.dev/tools/analyzer-plugins)
- [analysis_server_plugin](https://pub.dev/packages/analysis_server_plugin)
- [analyzer](https://pub.dev/packages/analyzer)
- [analyzer_testing](https://pub.dev/packages/analyzer_testing)
- [test_reflective_loader](https://pub.dev/packages/test_reflective_loader)

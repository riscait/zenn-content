---
title: "Flutterでフォントを追加して使う"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

# リファクタリング方針
新しい機能や追加事項などはより良い書き方があれば、可能な限りより良い書き方を行う。
既存コードは余力があれば行うことを推奨するが、あくまで任意。

改修範囲が広大になる場合は、別途リファクタリングIssueとして行う。

# Dart
Effective Dart
https://dart.dev/guides/language/effective-dart
↓
pedatic
`package:pedantic/analysis_options.yaml`
https://github.com/google/pedantic
↓
pedatic_mono：
pedantic最新ルール + Flutterサンプルリポジトリなどのルールで設定されているものはすべて取り入れてある
`package:pedantic_mono/analysis_options.yaml`

## 静的解析の有用性
- 事前の型チェックを厳密化できる
- 統一的なコーディングスタイルが矯正できる
- （パフォーマンス等が）より良いコードに誘導される

[Lintルール一覧](https://dart-lang.github.io/linter/lints/)

自動フォーマットをONに設定していれば、細かいルールを覚えていなくても、保存時に自動で整形してくれる。

AnalyzerやFormatルールに設定できるものは設定し、正とする。

インデントが崩れたりする場合、ほとんどのケースでは、カンマの有無が大事になってくる。

## インデント
フォーマッタのデフォルト設定に従う。

```dart
// ◎
if (isEnabled) {
  excute();
}
// x
if (isEnabled) {
    excute();
} 
```

## 末尾カンマ&改行

引数が1つの場合は、1行で書いても良い。
```dart
// ◎
Expanded(child: Text('Hello world')),
```

引数が2つ以上のWidgetは複数行で書く。
```dart
// ◎
Container(
  color: Colors.red,
  child: Text('Hello world'),
),
```

Widgetの引数が1つでも、その子Widgetが複数行の場合は、改行する。
つまり、1行でWidgetを書くのは、Widgetツリーの末端のみとなる。
```dart
// ◎
Container(
  child: Text(
    'Hello',
    style: TextStyle(fontSize: 16.0),
  ),
),
// x
Container(
    child: Text(
    'Hello',
    style: TextStyle(fontSize: 16.0),
));

例外的に、 `EdgeInsets` など、複数の引数があっても1行で可読性が担保できる（複数行表示が冗長）な場合は1行で良い。

```dart
Container(
  padding: const EdgeInsets.fromLTRB(1, 2, 3, 4),
),
```

関数の呼び出しで「名前なし引数」で開始する場合、同じ行でWidgetの括弧を開始する。
Widgetの閉じる括弧と関数呼び出しの閉じる括弧は同じ行。

```dart
Row(childlen: [
  const Text(''),
  const Text(''),
])

## デフォルト引数
イコールで設定する。
※以前のDartではコロンだったが Deprecated のためイコールを使う

```dart
void someFunc({String text = 'Hello'}) {
  ...
}
```

## 行数
80文字
## 文字列
`prefer_single_quotes`
シングルクォートを使用する。

## 文字列展開
`$` or `${}` を使用する
```dart
'Hello world from $place';
'Hello world from ${user.place}';
```

## import
フォーマッターによりアルファベット順を強制させる。

import は以下のグループごとにブロックを分ける。

- Dart
- サードパーティのパッケージ（Flutterなど）
- プロジェクトの他のパッケージ
- 同パッケージ内

```dart
// Dart
import 'dart:core';
import 'dart:ui';

// Third party
import 'package:flutter/foundation.dart';
import 'package:flutter/material.dart';

// Project
import 'package:yourapp/widget.dart';
import 'package:yourapp/text.dart';

// Same package
import '../foo.dart';
import 'bar.dart';
```

## クラス
省略可能な `this` は省略する

```dart
// titleはインスタンス変数
Text(title), // OK
Text(this.title), // NG
```

オブジェクト生成時の `new` は省略する。
 `const` は省略しない。
```dart
const SizedBox(child: Text('Hello'));
```

## const / final / var
- const > final > var の順番で優先して宣言する
- クラス変数は型を併記する。

`omit_local_variable_types`
初期化済みのローカル変数は型名を省略する。
（初期値が null の場合は型を明記する）

```dart
const String = 'Hello';

class Sample {
  Sample(this.label);

  final String label;

  int count;

  void execute() {
    final value = checkResult();

    var number = checkNumber();
    if (number == 0) {
      number = 10;
    }
    count = number;
  }
}
```

## 三項演算子
条件によってWidgetを出し分けるときなど、可読性を損なわない範囲で使用できる
長くなると可読性が損なわれるため、適宜メソッドやWidgetに切り出す

```dart
child: isEnabled ? Text('Hello') : Icon(Icons.play);
```

## TODO(name):
後で何か対応する必要がある場合はTODOを書き残しておく。
誰が見ても分かりやすいように具体的に、かつリンクがあればリンクを付け加えておく。

現状、TODOが多数あり、リントを見逃しやすくなっているため `todo: ignore` する。

## Flutter
画面を構成するWidgetは特に長くなりがちなので、適宜Widgetやメソッドに切り出す。

```dart
class SampleScreen extends StatelessWidget {
  const SampleScreen({Key key}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: ListView(
        children: const [
          Headline(),
          SubTitle(),
          Description(),
          PurchaseButton(),
          Caption(),
        ],
      ),
    );
  }
}
```

## unawaited
非同期処理だが `await` する必要がない場合（ロギングなど）は
`package:pedantic` の `unawaited()` メソッドを使用する

## 参考
参考にさせていただきました🙏
[Dart/Flutter の静的解析強化のススメ](https://medium.com/flutter-jp/analysis-b8dbb19d3978)
[Flutter/Dart コーディング スタイル](https://qiita.com/najeira/items/6982eb926f57ae0f2143)
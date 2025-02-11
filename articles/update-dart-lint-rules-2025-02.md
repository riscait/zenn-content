---
title: "[2025年2月] 追加されたDart Linter rules"
emoji: "🧵"
type: "tech"
topics: [flutter, dart]
published: true
publication_name: "altiveinc"
# cSpell:words nonobvious
---

[オルティブ株式会社](https://altive.co.jp)では、[altive_lints](https://github.com/altive/altive_lints)というリントパッケージを公開しています。
このパッケージでは、すべてのルールをデフォルトで有効化した上で、競合や不要なルールのみを無効化しています。

定期的にDartの「[All linter rules](https://dart.dev/tools/linter-rules/all)」を確認して新しいルールや削除されたルールをリストアップするようにしています。

### all_lint_rules.yaml ファイル

https://github.com/altive/altive_lints/blob/c58dbc78d3d88ba85a9ce3a8558a185ad96b6e2d/packages/altive_lints/lib/all_lint_rules.yaml

### 今回の、all_lint_rules.yaml更新プルリクエスト

https://github.com/altive/altive_lints/pull/87/files


---

## 今回追加・削除されたリントルール一覧

- Added
  - [omit_obvious_property_types](https://dart.dev/tools/linter-rules/omit_obvious_property_types)
  - [specify_nonobvious_property_types](https://dart.dev/tools/linter-rules/specify_nonobvious_property_types)
  - [strict_top_level_inference](https://dart.dev/tools/linter-rules/strict_top_level_inference)
  - [unnecessary_async](https://dart.dev/tools/linter-rules/unnecessary_async)
  - [unnecessary_ignore](https://dart.dev/tools/linter-rules/unnecessary_ignore)
  - [unnecessary_underscores](https://dart.dev/tools/linter-rules/unnecessary_underscores)
  - [unsafe_variance](https://dart.dev/tools/linter-rules/unsafe_variance)
- Removed
  - [package_api_docs](https://dart.dev/tools/linter-rules/package_api_docs)


## [omit_obvious_property_types](https://dart.dev/tools/linter-rules/omit_obvious_property_types)

トップレベルおよび静的変数に対する明白な型注釈を省略する。
初期化されたトップレベルまたは静的変数の型が明らかな場合は、型注釈を付けないでください。

:::message
このルールは Experimental であり、評価中のため変更または削除される可能性があります。
:::

### Bad

```dart
final int topLevelVariable = 42;

class A {
  static String staticVariable = 'Hello world';
}
```

### Good

```dart
final topLevelVariable = 42;

class A {
  static staticVariable = 'Hello world';
}
```

推論された型が期待するものではない場合は、望む型を明示してください。

```dart
final num topLevelVariable = 42;

class A {
  static String? staticVariable = 'Hello world';
}
```

:::message alert
[always_specify_types](https://dart.dev/tools/linter-rules/always_specify_types) とは同時に有効化できません。
:::

### altive_lintsで採用するか

**採用する**

同じく追加される [specify_nonobvious_property_types](https://dart.dev/tools/linter-rules/specify_nonobvious_property_types) と併用すると良い塩梅になりそうです。

## [specify_nonobvious_property_types](https://dart.dev/tools/linter-rules/specify_nonobvious_property_types)

型が明らかでない場合は、初期化されたトップレベル変数または静的変数に型注釈を付けます。

:::message
このルールは Experimental であり、評価中のため変更または削除される可能性があります。
:::

### Bad

```dart
final topLevelVariable =
    genericFunctionWrittenByOtherFolks(with, args);

class A {
  static var staticVariable =
      topLevelVariable.update('foo', null);
}
```

### Good

```dart
final Map<String, Widget?> topLevelVariable =
    genericFunctionWrittenByOtherFolks(with, args);

class A {
  static Map<String, Widget?> staticVariable =
      topLevelVariable.update('foo', null);
}
```

### altive_lintsで採用するか

**採用する**

同じく追加される [omit_obvious_property_types](https://dart.dev/tools/linter-rules/omit_obvious_property_types) と併用すると良い塩梅になりそうです。

## [strict_top_level_inference](https://dart.dev/tools/linter-rules/strict_top_level_inference)

型が推論されないトップレベルおよびクラスのメンバー宣言に型注釈を要求します。

### Bad

```dart
var _zeroPointCache;
class Point {
  get zero => ...;
  final x, y;
  Point(x, y) {}
  closest(b, c) => distance(b) <= distance(c) ? b : c;
  distance(other) => ...;
}
_sq(v) => v * v;
```

### Good

```dart
Point? _zeroPointCache;
class Point {
  Point get zero => ...;
  final int x, y;
  Point(int x, int y) {}
  closest(Point b, Point c) =>
      distance(b) <= distance(c) ? b : c;
  distance(Point other) => ...;
}
int _sq(int v) => v * v;
```

### altive_lintsで採用するか

**採用する**

`dynamic` になることを避けます。他のリントルールにより達成されていそうです。

## [unnecessary_async](https://dart.dev/tools/linter-rules/unnecessary_async)

`await` のない関数では `async` は必要ありません。

:::message
このルールは Experimental であり、評価中のため変更または削除される可能性があります。
:::

### Bad

```dart
void f() async {
  print(0);
}
```

### Good

```dart
void f() {
  print(0);
}
```

### altive_lintsで採用するか

**採用する**

awaitが不要な関数になった場合、asyncを削除し非Futureにしたいのでこのルールは理にかなっていると思います。

## [unnecessary_ignore](https://dart.dev/tools/linter-rules/unnecessary_ignore)

生成されていない診断コードを無視しないでください。

:::message
このルールは Experimental であり、評価中のため変更または削除される可能性があります。
:::

Lintルールの指摘は個別に `ignore` できますが、そもそも指摘されていないのに `ignore` していると指摘しれくれるようです。

### altive_lintsで採用するか

**採用する**

何らかの理由でもう指摘される理由がなくなったのに残ってしまった `ignore` を見たことがあるので、このルールは有用だと思います。

## [unnecessary_underscores](https://dart.dev/tools/linter-rules/unnecessary_underscores)

不要なアンダースコアは削除しましょう。
単一のワイルドカードで済む場合は、複数のアンダースコアを使用しないでください。

### Bad

```dart
void function(String _, int __) { }
```

### Good

```dart
void function(String _, int _) { }
```

### altive_lintsで採用するか

**採用する**

Dart 3.7からアンダースコアがワイルドカードになり、複数回使用できるようになります。
従来は、 アンダースコアの数を増やして（ `__` , `___` ...）対応していましたが、それが不要になりますね👍

## [unsafe_variance](https://dart.dev/tools/linter-rules/unsafe_variance)

:::message
このルールは Experimental であり、評価中のため変更または削除される可能性があります。
:::

### Bad

```dart
class C<X> {
  final bool Function(X) fun; // LINT
  C(this.fun);
}

void main() {
  C<num> c = C<int>((i) => i.isEven);
  c.fun(10); // Throws.
}
```

[unsafe_variance](https://dart.dev/tools/linter-rules/unsafe_variance) ページにて、

Better, Good, Honest の3つの対処が書かれています、詳しくはそちらをご覧ください。

### altive_lintsで採用するか

**採用する**

遭遇することはおそらく少ないと思いますが、Goodな方法で対処したいです。

## [package_api_docs](https://dart.dev/tools/linter-rules/package_api_docs)

Dart 2.0 以降は機能していないため削除されました。

現在は、[public_member_api_docs](https://dart.dev/tools/linter-rules/public_member_api_docs) が近しい役割を果たしていそうです。

## 最後に

最後まで閲覧いただきありがとうございました！

今後も新しいルールが追加された際には、どんなルールなのかを調べて採用の検討を続けていきたいと思います💪

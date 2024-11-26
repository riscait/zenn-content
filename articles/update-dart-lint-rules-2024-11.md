---
title: "[2024年11月] 追加されたDartリントルール"
emoji: "🧵"
type: "tech"
topics: [dart, flutter]
published: true
publication_name: "altiveinc"
# cSpell:words futureor nonobvious
---

## 概要

[オルティブ株式会社](https://altive.co.jp)では、[altive_lints](https://github.com/altive/altive_lints)というリントパッケージを公開しています。
このパッケージでは、すべてのルールをデフォルトで有効化した上で、競合や不要なルールのみを無効化しています。

定期的にDartの「[All linter rules](https://dart.dev/tools/linter-rules/all)」を確認して新しいルールや削除されたルールをリストアップするようにしています。

### all_lint_rules.yaml ファイル

https://github.com/altive/altive_lints/blob/c58dbc78d3d88ba85a9ce3a8558a185ad96b6e2d/packages/altive_lints/lib/all_lint_rules.yaml

### 今回の、all_lint_rules.yaml更新プルリクエスト

https://github.com/altive/altive_lints/pull/76/files


---

https://x.com/riscait/status/1860858181909635481

## 今回追加・削除されたルール一覧（公式リンク）

- Added
  - [avoid_futureor_void](https://dart.dev/tools/linter-rules/avoid_futureor_void)
  - [avoid_null_checks_in_equality_operators](https://dart.dev/tools/linter-rules/avoid_null_checks_in_equality_operators)
  - [omit_obvious_local_variable_types](https://dart.dev/tools/linter-rules/omit_obvious_local_variable_types)
  - [specify_nonobvious_local_variable_types](https://dart.dev/tools/linter-rules/specify_nonobvious_local_variable_types)
  - [use_truncating_division](https://dart.dev/tools/linter-rules/use_truncating_division)
- Removed
  - [unsafe_html](https://dart.dev/tools/linter-rules/unsafe_html)

それでは、1つずつ見ていきまます！

`analysis_options.yaml` で有効化（あるいは無効化）するかどうかの検討の一助となれば幸いです。

参考までに `altive_lints` で採用するかどうかの検討も記載しています。

## avoid_futureor_void

https://dart.dev/tools/linter-rules/avoid_futureor_void

- Experimental（実験段階）
- Dart 3.6から利用可能。

| 項目                                                                | 内容     |
| ------------------------------------------------------------------- | -------- |
| 利用可能バージョン                                                  | Dart 3.6 |
| 実験段階かどうか（Experimental）                                    | Yes      |
| 互換性のないルール                                                  | なし     |
| [quick fix](https://dart.dev/tools/linter-rules#quick-fixes) の有無 | なし     |

### 概要

戻り値に `FutureOr` を使用するのを避けましょう。

### 詳細

FutureOrは、Futureまたは非Futureのいずれかを表す型です。
活用した経験はないのですが、用途としては以下のように、抽象クラスで使用するものだと理解しています。

```dart
abstract class DataProvider {
  // Future<String>かStringを返すように実装してほしい
  FutureOr<String> fetchData();
}

class SyncDataProvider implements DataProvider {
  @override
  String fetchData() {
    // 同期的に結果を返す
    return "データをすぐ返すよ";
  }
}

class AsyncDataProvider implements DataProvider {
  @override
  Future<String> fetchData() async {
    // 非同期的に結果を返す
    await Future.delayed(Duration(seconds: 1));
    return "データを非同期で返すよ";
  }
}
```

### altive_lintsで採用するか

**採用する**

初期のRiverpodのAsyncNotifierクラスを定義するときに `build` メソッドの戻り値で `FutureOr` を使ってしまっていた気がします。

`FutureOr` である必要はなく、使いたいときは ignore すれば良いと思うので採用予定です。

## avoid_null_checks_in_equality_operators

https://dart.dev/tools/linter-rules/avoid_null_checks_in_equality_operators

| 項目                                                                | 内容     |
| ------------------------------------------------------------------- | -------- |
| 利用可能バージョン                                                  | Dart 2.0 |
| 実験段階かどうか（Experimental）                                    | No       |
| 互換性のないルール                                                  | なし     |
| [quick fix](https://dart.dev/tools/linter-rules#quick-fixes) の有無 | あり     |

### 概要

`==` のカスタム演算子で `null` チェックを行わないようにしましょう。

### 詳細

nullは特別な値なので、Null以外のインスタンスは同等になり得ないため。

```dart
class Person {
  final String? name;

  @override
  operator ==(Object? other) =>
      // Nullチェックは冗長
      other != null && other is Person && name == other.name;
}
```

### altive_lintsで採用するか

**採用する**

冗長なチェックは避けたいですね。

## omit_obvious_local_variable_types

https://dart.dev/tools/linter-rules/omit_obvious_local_variable_types

| 項目                                                                | 内容                                                                             |
| ------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| 利用可能バージョン                                                  | Dart 3.6                                                                         |
| 実験段階かどうか（Experimental）                                    | Yes                                                                              |
| 互換性のないルール                                                  | [always_specify_types](https://dart.dev/tools/linter-rules/always_specify_types) |
| [quick fix](https://dart.dev/tools/linter-rules#quick-fixes) の有無 | あり                                                                             |

### 概要

型が明らかな場合は、初期化されたローカル変数に型注釈を付けないでください。

### 詳細

スコープが狭いローカル変数で型を省略すると、より重要な変数名や初期化された値に焦点を当てることができます。
右辺の式で、変数の型が明らかな場合は、 `const` , `final` または `var` キーワードを省略して型を省略します。

```dart
List<List<Ingredient>> possibleDesserts(Set<Ingredient> pantry) {
  // ローカル変数の冗長な型注釈
  List<List<Ingredient>> desserts1 = <List<Ingredient>>[];
  // 省略した場合
  var desserts2 = <List<Ingredient>>[];
  // ...
}
```

ただ、推論された型が、変数に持たせたい型とは異なる場合は、型注釈を付ける必要があります。

```dart
Widget build(BuildContext context) {
  Widget result = someCondition ? Text('You won!') : Icon(Icons.bad);
  if (applyPadding) {
    result = Padding(padding: EdgeInsets.all(8.0), child: result);
  }
  return result;
}
```

似たルールである `omit_local_variable_types` はすべてのローカル変数に適用されます。
対して `omit_obvious_local_variable_types` は、型が「明らか」である場合にのみ適用されます。

### altive_lintsで採用するか

**採用する**

`altive_lints` ではすでに `omit_local_variable_types` を採用しているため、`omit_obvious_local_variable_types` も採用できます。

原則、スコープの狭いローカル変数に型を明示したくありません。
ただ、型の明示が必要なときは ignoreと共に明示します。

`omit_local_variable_types` は無効化し、 `omit_obvious_local_variable_types` のみにした場合の挙動も確認してみたいです。

## specify_nonobvious_local_variable_types

https://dart.dev/tools/linter-rules/specify_nonobvious_local_variable_types

| 項目                                                                | 内容                                                                                       |
| ------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| 利用可能バージョン                                                  | Dart 3.6                                                                                   |
| 実験段階かどうか（Experimental）                                    | Yes                                                                                        |
| 互換性のないルール                                                  | [omit_local_variable_types](https://dart.dev/tools/linter-rules/omit_local_variable_types) |
| [quick fix](https://dart.dev/tools/linter-rules#quick-fixes) の有無 | あり                                                                                       |

### 概要

型が明らかでない場合は、初期化されたローカル変数に型注釈を付けましょう。

### 詳細

以下、公式リンクから引用した例です。
`genericFunctionDeclaredFarAway`メソッドの実際の中身がわからないのでなんとも言えないところですが、Genericな型を返すメソッドで、戻り値の型が曖昧なのかもしれません。
`for-in` の中の変数も型を名刺していますね👀

```dart
// Bad
var desserts = genericFunctionDeclaredFarAway(<num>[42], 'Something');
for (final recipe in cookbook) {
  if (pantry.containsAll(recipe)) {
    desserts.add(recipe);
  }
}

// Good
List<List<Ingredient>> desserts = genericFunctionDeclaredFarAway(
  <num>[42],
  'Something',
);
for (final List<Ingredient> recipe in cookbook) {
  if (pantry.containsAll(recipe)) {
    desserts.add(recipe);
  }
}
```

### altive_lintsで採用するか

**採用しない**

altive_lintsでは、`omit_local_variable_types` を採用しているため、`specify_nonobvious_local_variable_types` は採用できません。

しかし、`omit_local_variable_types` を無効化し、 `specify_nonobvious_local_variable_types` と `omit_obvious_local_variable_types` の組み合わせも試してみたいです。

## use_truncating_division

https://dart.dev/tools/linter-rules/use_truncating_division

| 項目                                                                | 内容     |
| ------------------------------------------------------------------- | -------- |
| 利用可能バージョン                                                  | Dart 3.6 |
| 実験段階かどうか（Experimental）                                    | No       |
| 互換性のないルール                                                  | なし     |
| [quick fix](https://dart.dev/tools/linter-rules#quick-fixes) の有無 | あり     |

### 概要

除算 ('/') の後に 'toInt()' を続けず、切り捨て除算 '~/' を使用してください。

### 詳細

Dart には「切り捨て除算」演算子があります。

これは、除算の後に切り捨てを行うのと同じ操作ですが、より簡潔で表現力に富み、特定の入力に対して一部のプラットフォームでよりパフォーマンスが向上する場合があります。

```dart
// Bad
final x = (2 / 3).toInt();

// Good
final x = 2 ~/ 3;
```

### altive_lintsで採用するか

**採用する**

用意された、より簡潔な構文を使用します。

## 最後に

最後まで閲覧いただきありがとうございました！

今後も新しいルールが追加された際には、どんなルールなのかを調べて採用の検討を続けていきたいと思います💪

---
title: "初歩的なリファクタリング"
emoji: "💬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [flutter, dart]
published: false
---

### 関数（メソッド）内の変数を、型を明示して宣言している。

再代入しない場合は `final` 再代入する場合は `var` で宣言する。

```diff dart
void main() {
-  String fullName = '$firstName $lastName';
+  final fullName = '$firstName $lastName';
}
```

### ジェネリクスの型を明示していない

```diff dart
- final response = await Dio().get('/request');
+ final response = await Dio().get<Map<String, dynamic>>('/request');
```

### Navigator.push等のジェネリクスに遷移先のクラスを指定している
```diff dart
- await Navigator.of(context).push<DetailScreen>(...);
+ await Navigator.of(context).push<void>(...);
```

遷移先画面からの戻り値の型を指定します。
遷移先画面から特に何も受け取らない場合は、 `void` を指定しましょう。

### 意味の無いWidgetでラップしている
```diff dart
- Container(child: const Text('保存'));
+ const Text('保存');
```
何もしていないWidgetは削除しましょう。

### とりあえず「Container」状態。責務の小さいWidget
```diff dart
- Container(
+ Padding(
-   padding: const EdgeInsets.all(16),
-   child: const Text('保存'),
- ),
```

`Container` はコストが高いです。
より適したシンプルなWidgetがあればそちらを使いましょう。
他には、
- width, height -> `SizedBox`
- `DecoratedBox`
- align -> `Align`
- color -> `ColoredBox`

### 

### 

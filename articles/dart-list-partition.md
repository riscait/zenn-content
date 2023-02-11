---
title: "Flutter/DartでListの要素を分割する方法４選"
emoji: "🔪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [dart, flutter]
published: true
publication_name: "altiveinc"
---

こんにちは！Altive株式会社でFlutterアプリ開発を行なっている村松龍之介([@riscait](https://twitter.com/riscait))です😸

---

DartのList型の中身を分割しつつ、リスト化できる方法を備忘録がてら書き残します。

# 結論
`collection` パッケージの `slices` 関数か、 `quiver` パッケージの `partition` 関数を使おう！
パッケージに依存したくない場合は、 `take` と `skip` で要素を取り出そう！

# 手法4選

## 1. List.sublist(start, end) を使用する方法
for文で回して、内部で `sublist` 関数を使って範囲内の要素をリスト化しています。

@[gist](https://gist.github.com/Riscait/f4c95ade1546c9c040566212532aa8ad)

## 2. List.skip(count), List.take(count) を使用する方法
今度は `do-while` 文を使用しています。
`do` の中で `skip` と `take` を使用して分割したい要素を取得しています。
こっちの方が読みやすい感じがしますね。

@[gist](https://gist.github.com/Riscait/50d2a17b7212e889dbfd64a81fa31fe7)

## 3. quiverパッケージの partition() を使用する方法
Google製の Quiver というDart用のユーティリティパッケージがあります。
https://pub.dev/packages/quiver

### パッケージを追加します。
最新版の `quiver` パッケージを `pubspec.yaml` の `dependencies` に追加しましょう。

```
flutter pub add quiver
```

`partition(iterable, size)` メソッドを使いましょう👍

```dart
import 'package:quiver/iterables.dart';

final items = ['a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'i', 'j', 'k'];
final chunkSize = 3;
// [[a, b, c], [d, e, f], [g, h, i], [j, k]]
final chunkedItems = partition(items, chunkSize);
```

1行で期待通りの動きをしてくれました🥺

（内部的には Iteratorを実装した _PartitionIteratorで `while` を使用しているようでした）
https://github.com/google/quiver-dart/blob/master/lib/src/iterables/partition.dart

## 4. `collection` パッケージの `slices(length)` メソッドを使用する方法
:::message
@k9i さん、教えていただきありがとうございます🙌
https://zenn.dev/link/comments/f2450662db0663
:::

Dart.dev製の `collection` という、コレクションを簡単に扱うためのパッケージがあります。
https://pub.dev/packages/collection

`firstOrNUll` など `Null Safety` で有用なユーティリティが入っているので、すでに採用しているプロジェクトも少なくないのでしょうか？

### パッケージを追加します。
最新版の `collection` パッケージを `pubspec.yaml` の `dependencies` に追加しましょう。

```
flutter pub add collection
```

`slices(length)` メソッドを使いましょう👍

```dart
import 'package:package:collection/collection.dart';

final items = ['a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'i', 'j', 'k'];
  final sliced = items.slices(3);
  print(sliced); // ([a, b, c], [d, e, f], [g, h, i], [j, k])
```

1行で期待通りの動きをしてくれました🥺

https://pub.dev/documentation/collection/latest/collection/IterableExtension/slices.html

# 最後に
個人的には、すでに採用していることが多く、Dart.devのパッケージでも `collection` の `slices(length)` メソッドを利用していきたいと思いました！
しかし、何らかの理由でパッケージに依存したくない場合は `do-while, take, skip` を使うと思います。

最後までご閲覧ありがとうございました🙌

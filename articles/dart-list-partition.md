---
title: "Flutter/DartでListの要素を分割する方法３選"
emoji: "🔪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [dart, flutter]
published: true
---

こんにちは！Altive株式会社でFlutterアプリ開発を行なっている村松龍之介([@riscait](https://twitter.com/riscait))です😸

---

DartのList型の中身を分割しつつ、リスト化できる書き方を調べて3つの書き方を学んだので、備忘録がてら書き残します。

# 結論
`quiver` パッケージの `partition` 関数を使おう！
パッケージに依存したくない場合は、 `take` と `skip` で要素を取り出そう！

# 手法３選

## 1. List.sublist(start, end) を使用する方法
for文で回して、内部で `sublist` 関数を使って範囲内の要素をリスト化しています。

@[gist](https://gist.github.com/Riscait/f4c95ade1546c9c040566212532aa8ad)

## 2. List.skip(count), List.take(count) を使用する方法
今度は `do-while` 文を使用しています。
`do` の中で `skip` と `take` を使用して分割したい要素を取得しています。
こっちの方が読みやすい感じがしますね。

@[gist](https://gist.github.com/Riscait/50d2a17b7212e889dbfd64a81fa31fe7)

## 3. quiverパッケージの partition() を使用する方法
これが一番簡単です。
Google製の Quiver というDart用のユーティリティパッケージがあります。
https://pub.dev/packages/quiver

### パッケージを追加します。
下記のコマンドを実行すると、 `pubspec.yaml` の `dependencies` にインストール可能な最新バージョンのパッケージがインストールされます。
（もしくは、直接 `dependencies` に `quiver: any` を追加して `flutter pub get` しましょう）

```
flutter pub add quiver
```

パッケージをアプリに追加したら、使いたいファイルに `import 'package:quiver/iterables.dart'` を書いて、
`partition` 関数を使いましょう👍

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

# 最後に
個人的には、他にも便利な関数が多数使える `quiver` パッケージを利用していきたいと思いました。
しかし、何らかの理由でパッケージに依存したくない場合は `do-while, take, skip` を使おうと思います。

最後までご閲覧ありがとうございました🙌

---

Altive株式会社では、Flutterアプリ開発を承っております。
また、ともに働くFlutterアプリ開発者も募集中です😊

Riverpod x Flutterの本も書きました👍
https://zenn.dev/riscait/books/flutter-riverpod-practical-introduction

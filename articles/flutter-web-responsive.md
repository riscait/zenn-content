---
title: "Flutterでフォントを追加して使う"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

`LayoutBuilder` を使用することで、制約を取得することができる。
`maxWidth` （最大幅）によって分岐させられる。
高さ・アスペクト比等に基づいて分岐させることもできる。

スマートフォンを回転させたり、すると、 `build` 関数が再実行される。

`MediaQuery.of()` を使用することで、アプリのサイズや向きが分かる。
ユーザーがなんらかの方法でサイズを変更すると、 `build` 関数が再実行される。

## レスポンシブUIの構築に役立つWidgets
AspectRatio
CustomSingleChildLayout
CustomMultiChildLayout
FittedBox
FractionallySizedBox
LayoutBuilder
MediaQuery
MediaQueryData
OrientationBuilder

## 参考文献
https://flutter.dev/docs/development/ui/layout/adaptive-responsive
---
title: "Flutterでフォントを追加して使う"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

[Use a custom font - Flutter.dev](https://flutter.dev/docs/cookbook/design/fonts)

[Google Fonts](https://fonts.google.com/?subset=japanese)

[M PLUS Rounded 1c](https://fonts.google.com/specimen/M+PLUS+Rounded+1c?subset=japanese) 

## フォントファイルをアプリのディレクトリに入れる
アプリの名前が `restock` の場合の例
```
restock/
  assets/
    fonts/
      MPLUSRounded1c-Regular.ttf
```

ここでは、アプリルートディレクトリ直下に`assets`ディレクトリ、`fonts`ディレクトリを作り、
その中にフォントの`ttf`ファイルを格納しました。
      
## pubspec.yaml に追記
追加したフォントファイルのパスを`pubspec.yaml`に追記します。
```
flutter:
  uses-material-design: true

  fonts:
    - family: M_PLUS_Rounded_1c
      fonts:
        - asset: assets/fonts/MPLUSRounded1c-Regular.ttf
```

## 追加したフォントを使用する

`Text`ウィジェットでの使用してみます。

フォントは`fontFamily`で指定します。

```
Text(
    'リストック',
    style: TextStyle(
      fontFamily: 'M_PLUS_Rounded_1c',
    ),
),
```

既存の`TextStyle`を指定した上でフォントを変更する場合は`copyWith()`を使用します。

```
Text(
    'リストック',
    style: Theme.of(context).textTheme.headline6.copyWith(
      fontFamily: 'M_PLUS_Rounded_1c',
    ),
),
```
---
title: "Flutter/Riverpodで、ProviderScopeのoverridesを使ってFlavorを伝播する"
emoji: "🌬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [flutter, riverpod, flavor, dart]
published: true
publication_name: "altiveinc"
---
こんにちは、Flutterでのアプリ開発をメインとしている「[Altive株式会社](https://altive.co.jp)」の村松龍之介（[@riscait](https://x.com/riscait)）です！です。
FlutterとiOSネイティブアプリ開発を仕事を行なっております。

Riverpodでは、Providerはグローバルで定義しますが、 `ProviderScope` で囲った箇所でのみProviderが使用になります。

`ProviderScope` の `overrides` は、テストの際などに Provider をスタブに置き換えたりするのに使うと思います。
ただ、 僕の実装では Flavor を通常の方法で Provider に渡すことができず、ルートで override するしかなかったため、以下の方法を使っています。

今回は、Flavor（開発・本番環境などを分ける用途に使っている）をProviderとしてアプリ全体で使いたい場合を例に、
`ProviderScope` の `overrides` を使った方を書きます。

:::message
monoさんにコメントに補足していただきましたが、 `override` 使用する用途としては以下があるそうです🙏
- アプリで(普通の)Providerをoverride(ルートでしかできないことに注意) <-今回の用途
- テスト時に(普通の)Providerをoverride
- アプリでScopedProviderを特定のWidgetツリー配下でProviderScopeをネストしてoverride(普通のProviderはネストのProviderScopeでoverrideできないことに注意)

詳しくはコメントをご覧ください 
https://zenn.dev/riscait/articles/flutter-riverpod-provider-scope-overrides#comment-b7845837440e0c479d19
:::

## Flavor の定義
今回の例では、開発環境用の `development` 、テスト環境用の `staging` 、本番環境用の `production` 
の3種類のFlavorを用意したとします。

```dart
enum Flavor {
  development,
  staging,
  production,
}
```

## Provider の定義
Flavor用のProviderを定義します。

定義場所は、 特にこだわりが無ければ、 `enum Flavor` を定義したのと同じファイルで良いかと思います。

```dart
// 初期値を `null` にする場合
final flavorProvider = Provider<Flavor>((ref) => null);
```
初期値を `null` にする場合は、型を明示する必要があります

```dart
// 初期値を `development` にする場合
final flavorProvider = Provider((ref) => Flavor.development);
```

Flavorはアプリ内で変更されない＝状態を持たない
なので、普通の `Provider` で良いですね。

## アプリ起動時に指定した Flavor を受け取ってセットする
Flavorの指定の仕方として、 

1. `dart-define` を使う
1. `main.dart` ファイルを環境別に用意する

の2通りありますので順に書いていきます。

### ① dart-define で Flavor を分けている場合
`--dart-define` を使う場合は、 `main.dart` ファイルを複数作成する必要がない代わりに、
main関数の中で flavor を判別する必要があります。

```shell
# ビルドモード：debug, Flavor：development の場合の起動コマンド
flutter run --debug --flavor development --dart-define=FLAVOR=development
```
`--dart-define` で Flavor を指定します。

#### main.dart ファイルで Flavor を判定
`const String.fromEnvironment('FLAVOR')` で、--dart-defineで指定したFlavor文字列を取得できます。
以下のように、 if文やswitch-case で判定して Flavor を返せばOKです。

```dart:main.dart
void main() {
  final flavor = () {
    switch (const String.fromEnvironment('FLAVOR')) {
      case 'development':
        return Flavor.development;
      case 'staging':
        return Flavor.staging;
      case 'production':
        return Flavor.production;
    }
    throw AssertionError('Invalid Flavor');
  }();

  runApp(
    ProviderScope(
      overrides: [
        // Flavor Provider にセット
        flavorProvider.overrideWithValue(flavor),
      ],
      child: const MyApp(),
    ),
  );
}
```

:::message
記事公開時は以下のように書いていましたが、monoさんにコメントで簡潔に書く方法を教えていただいたため、以上のように修正しました🙏
```dart
flavorProvider.overrideWithProvider(
  Provider((_) => flavor),
),
```
:::

#### EnumToStringパッケージを使用した例
`enum_to_string` パッケージを使用すると、 enum から文字列を取得したり、文字列から enum を取得したりが簡単になります。
[enum_to_string - pub.dev](https://pub.dev/packages/enum_to_string)

```yaml:pubspec.yaml
dependencies:
  enum_to_string: ^1.0.14
```
パッケージ導入例

EnumToStringを使用した場合の書き方は以下の通りです。
少し簡潔になりました。

```dart:main.dart
void main() {
  final flavor = EnumToString.fromString(
    Flavor.values,
    const String.fromEnvironment('FLAVOR'),
  );

  runApp(
    ProviderScope(
      overrides: [
        // ここで実際の Flavor をセットする
        flavorProvider.overrideWithValue(flavor),
      ],
      child: const MyApp(),
    ),
  );
}
```

EnumToString を使えば、 enum ごとに自分で extension を書く必要が無くて楽ですね！

### ② main ファイルで Flavor を分けている場合
```shell
# ビルドモード：debug, Flavor：development の場合の起動コマンド
flutter run --debug --flavor development --target lib/main_development.dart
```
`--target` で main ファイルを指定しています。

```dart:main_development.dart
void main() => run(flavor: Flavor.development);
```
※ここでは、 `run.dart` という全環境共通のファイルを別途作っている場合を想定します。

```dart:run.dart
Future<void> run({@required Flavor flavor}) async {
  return runApp(
    ProviderScope(
      overrides: [
        // ここで実際の Flavor をセットする
        flavorProvider.overrideWithValue(flavor),
      ],
      child: const MyApp(),
    ),
  );
}
```
`overrides` を使って、 main ファイルから受け取った Flavor を Provider にセットしています。

## 参考リンク
[Flutterで環境ごとにビルド設定を切り替える — iOS編](https://medium.com/flutter-jp/flavor-b952f2d05b5d)

ご閲覧ありがとうございました！

Twitterでも、主に Flutter, Firebase、iOS/Swift について呟いております。
フォローしていただければ嬉しいです☺️ → 🐦村松龍之介[@riscait](https://twitter.com/riscait) 

## 宣伝

Altive株式会社では、Flutterアプリの開発・運営を承っております。
お気軽にお問い合わせください🫡 
https://altive.co.jp/contact

---

Riverpod の実践入門本を公開中です📘
https://zenn.dev/riscait/books/flutter-riverpod-practical-introduction

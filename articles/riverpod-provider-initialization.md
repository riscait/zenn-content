---
title: "Riverpodの非同期プロバイダの初期化方法"
emoji: "♾️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [flutter,dart,riverpod]
published: false
---

## はじめに

Flutterアプリの開発では、設定や外部パッケージ情報など、非同期処理が必要なデータを効率的に管理することが重要です。Riverpodは、これらのデータを簡潔かつ安全に扱うための強力なフレームワークですが、非同期プロバイダを使用する場合、いくつかの課題が存在します。

本記事では、非同期プロバイダを初期化する際の課題を解説し、それを解決するための3つの方法を提案します。それぞれの方法の利点・欠点を比較し、プロジェクトに最適な選択肢を見つけられるようサポートします。

## Riverpodを使う上での非同期プロバイダの課題

`SharedPreferences` や `PackageInfo` のインスタンスを提供するプロバイダを用意する場合に存在する課題として…

`SharedPreferences.getInstance()` や `PackageInfo.fromPlatform()` などの非同期処理を行いたい場合、非同期なプロバイダ（`FutureProvider`）になってしまうというものがあります。

---

詳しくいうと、`SharedPreferences` のインスタンスを提供するプロバイダを定義しようと思うと、 `getInstance()` が非同期メソッドのため、`Future` を返す必要があります。

```dart
@riverpod
Future<SharedPreferences> sharedPreferences(Ref ref) async {
  return SharedPreferences.getInstance();
}
```

この場合は利用する側で `async/await` を使ったり、 `AsyncValue` として扱わなければなりません。

```dart
// Futureとして
final sp = await ref.read(sharedPreferencesProvider.future);
sp.getString('key');

// AsyncValueとして
final asyncValue = ref.watch(sharedPreferencesProvider);
asyncValue.when(
  loading: () => CircularProgressIndicator(),
  error: (e, s) => Text('error: $e'),
  data: (sp) => SomeWidget(title: sp.getString('key')),
);
```

## 対策① main.dartで初期化し、ProviderScope.overridesを使う

`ProviderScope.overrides` を使い、初期化した値でプロバイダを上書きする方法があります。

まず、上書きすることを前提としたプロバイダを定義します。

```dart
@Riverpod(keepAlive: true)
SharedPreferences sharedPreferences(Ref ref) {
  // 上書きを忘れた場合は例外が投げられるようにしておく。
  throw UnimplementedError();
}
```

`runApp` 直下の `ProviderScope` で `overrides` を使い、初期化した値でプロバイダを上書きします。

```dart
Future<void> main() async {
  late final SharedPreferences sp;
  late final PackageInfo pi;
  // 並列実行。Recordsを使った書き方も可能。
  await Future.wait([
    // main関数を非同期関数にし、awaitを使ってインスタンスを取得。
    Future(() async => sp = await SharedPreferences.getInstance()),
    Future(() async => pi = await PackageInfo.fromPlatform()),
  ]);

  runApp(
    ProviderScope(
      overrides: [
        // 先ほどawaitして取得したインスタンスでプロバイダを上書き。
        sharedPreferencesProvider.overrideWithValue(sp),
        packageInfoProvider.overrideWithValue(pi),
      ],
      child: MyApp(),
    ),
  );
}
```

こうしておくことで、 `sharedPreferencesProvider` を `async/await` or `AsyncValue` なしで使用できるようになります。

```dart
final sp = ref.read(sharedPreferencesProvider);
sp.getString('key');
```

### main.dartで初期化を行う場合の注意点

`main.dart` で `runApp` 前に初期化することになるため以下の課題があります。

- 初期化を `try-catch` で囲んで適切にエラーハンドリングする必要がある。その際、 `runApp` や `ProviderScope` 等を複数回書かなくてはいけない
- `runApp` で最初のFlutter Widgetを返す前なので、初期化中はネイティブの起動画面（Splash）が表示されることになり、細かい画面表示の制御が難しい

## 対策② 最初に表示するWidgetで初期化し、以降では `.requireValue` を使用する方法

2つ目の方法として、 `main.dart` での初期化・オーバーライドをやめ、 `runApp` 後Flutterで最初に表示するWidgetで初期化を行う方法があります。

また、初期化の必要なプロバイダをまとめた初期化用のプロバイダを作成しておくことで、責務が明確になります。

この方法についてはAndrea Bizzottoさんの記事を参考にさせていただきました🙏

https://codewithandrea.com/articles/robust-app-initialization-riverpod/

### 初期化用プロバイダの定義例

```dart
@Riverpod(keepAlive: true)
Future<void> initialization(Ref ref) async {
  ref.onDispose(() {
    // ref.invalidate(initializationProvider);
    // を実行した時に諸々のプロバイダも破棄するようにする。
    // ログアウト時などに使う想定。
    ref
      ..invalidate(sharedPreferencesProvider)
      ..invalidate(packageProvider);
  });
  // 並列実行しても良い初期化処理はここに書く。
  await Future.wait([
    ref.watch(sharedPreferencesProvider.future),
    ref.watch(packageProvider.future),
  ]);
  // 他、任意の初期化処理を行う。
}
```

### 初期化用プロバイダを使うWidgetの例

```dart
/// アプリで最初に表示するFlutterページ
class InitializationPage extends ConsumerWidget {
  const InitializationPage({super.key, required this.onInitialized});

  final WidgetBuilder onInitialized;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final asyncValue = ref.watch(initializationProvider);
    return asyncValue.when(
      // 初期化中に表示するWidgetを指定。
      loading: () => const _LoadingPage(),
      // エラーが発生した場合はエラーページを表示してリトライボタンも設置したい。
      error: (e, st) => _ErrorPage(
        message: e.toString(),
        // 初期化でエラーが発生した場合は初期化用プロバイダを再読み込みする。
        onRetry: () => ref.invalidate(initializationProvider),
      ),
      // 初期化成功した場合は、引数で指定されたWidgetを表示できるようにする。
      data: (_) => onInitialized(context),
    );
  }
}
```

`runApp` でchildに指定するMaterialApp等、あるいはRouterの初期画面のWidgetを `InitializationPage` でラップします。

```dart
void main() {
  runApp(
    const ProviderScope(
      child: MaterialApp(
        home: InitializationPage(
          onInitialized: () => MainApp(),
        ),
      ),
    )
  );
}
```

### MainApp以降では `.requireValue` を使用する

初期化は `InitializationPage/Provider` で完了しているので、それ以降は `requireValue` を使ってFutureではない値を取得することが可能です。

```dart
final sp = ref.read(sharedPreferencesProvider).requireValue;
final someValue = sp.getString('some_key');
```

### `requireValue` を使う場合の課題

- 毎回 `requireValue` を使う必要がある
- どのプロバイダが初期化済みのもので `requireValue` が使えるのかが分かりにくい（命名規則で補助することは可能）

## 対策③ 初期化用のプロバイダと使用するプロバイダを分ける

```dart
import 'package:hooks_riverpod/hooks_riverpod.dart';
import 'package:package_info_plus/package_info_plus.dart';
import 'package:riverpod_annotation/riverpod_annotation.dart';

part 'package_info_provider.g.dart';

/// 初期化用のプロバイダ。
@Riverpod(keepAlive: true)
Future<PackageInfo> packageInfoInitializing(Ref ref) async =>
    PackageInfo.fromPlatform();

/// 初期化後、アプリで実際に使用するプロバイダ。
@Riverpod(keepAlive: true)
PackageInfo packageInfo(Ref ref) =>
    ref.watch(packageInfoInitializingProvider).requireValue;
```

```dart
@Riverpod(keepAlive: true)
Future<void> initialization(Ref ref) async {
  ref.onDispose(() {
    // 初期化用のプロバイダを破棄する。
    // 実際に使用するプロバイダも、初期化用プロバイダに依存しているので破棄される。
    ref
      ..invalidate(sharedPreferencesInitializingProvider)
      ..invalidate(packageInfoInitializingProvider)
      ..invalidate(userDeviceInitializingProvider);
  });
  await Future.wait([
    ref.watch(sharedPreferencesInitializingProvider.future),
    ref.watch(packageInfoInitializingProvider.future),
    ref.watch(userDeviceInitializingProvider.future),
  ]);
  // 他、任意の初期化処理を行う。
}
```

初期化は `InitializationPage/Provider` で完了しているので、 `someInitializingProvider` ではなく、 `someProvider` を使って値を取得することが可能です。
`requiredValue` を使う必要はありません。

```dart
final sp = ref.read(sharedPreferencesProvider);
final someValue = sp.getString('some_key');
```

### `someInitializingProvider` を使う場合の懸念点

- 何らかの理由で `someInitializingProvider` を破棄したい場合、 `someProvider` ではなく `someInitializingProvider` を破棄する必要がある
  - でないと、例えば `SharedPreferences.getInstance` や `PackageInfo.fromPlatform()` が再実行されません
  - しかし、こういった初期化を必要とするプロバイダは `InitializingProvider` の `ref.onDispose` にまとめておきそれを利用するようにすることで、ある程度解消可能です

## ディープリンクやURLナビゲーションに対応する場合

`runApp` の `App(MaterialApp)` を `InitializationPage` で囲む方法だと、ディープリンクやURLナビゲーションを正しく処理することが難しくなります。

以下、 `go_router` を使った場合の例です。

`MaterialApp.router` の `builder` プロパティを使用することで、 `GoRouter` インスタンスがを指定した `MaterialApp.router` を使用でき、その `child` を `InitializationPage` でラップすることができます。

```dart
class MainApp extends ConsumerWidget {
  const MainApp({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return MaterialApp.router(
      routerConfig: ref.watch(routerProvider),
      builder: (context, child) {
        return InitializationPage(onInitialized: (_) => child!);
      },
    );
  }
}
```

## まとめ

非同期プロバイダを効率的に初期化する方法として、本記事では以下の3つのアプローチを紹介しました。

1. **main.dartでの初期化とProviderScope.overridesの利用**  
  初期化処理をアプリのエントリポイントで行う方法。シンプルで簡単ですが、初期化中のエラーハンドリングや細かい画面制御に課題があります。

2. **初期画面で初期化し、requireValueを使用する方法**  
  アプリの最初のページで初期化を行う方法。細かいUI制御が可能ですが、`.requireValue` の使用が煩雑になる可能性があります。

3. **初期化用プロバイダと実際に利用するプロバイダの分離**  
  初期化と使用のプロバイダを明確に分ける方法。設計が明確になりますが、初期化プロバイダの管理に注意が必要です。

各方法はそれぞれ特有の利点と欠点を持つため、プロジェクトの規模や要件に応じて選択すると良さそうです。

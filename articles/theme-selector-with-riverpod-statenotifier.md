---
title: "Riverpod, StateNotifierでアプリ内テーマの切り替え機能を実装する"
emoji: "🌓"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [flutter, dart, riverpod, theme, StateNotifier]
published: true
publication_name: "altiveinc"
---
:::message
12月22日(火)更新
・起動可能なサンプルリポジトリを作成しました。
 → [flutter_theme_mode_selector_example | GitHub](https://github.com/Riscait/flutter_theme_mode_selector_example)
:::
こんにちは、[村松龍之介@Riscait](https://twitter.com/riscait)です。

:::message
この記事は、 [Flutter #1 Advent Calendar 2020 | Qiita](https://qiita.com/advent-calendar/2020/flutter) 21日目の記事です💪
:::

4月に、Mediumにて「[FlutterのTheme切り替え機能をChangeNotifierProviderで実装する](https://medium.com/@riscait/implementing-flutters-theme-switching-feature-in-changenotifier-428b665e5ac2)」という記事を書きました。

8ヶ月経ち、Flutter自体のアップデートや、僕自身が使う状態管理の移り変わりがありました。
2020年12月現在使用している状態管理方法で書き直ししてみたいと思います。

```
前回：ChangeNotifier + Provider
今回：StateNotifier + Riverpod + Flutter Hooks
```

:::message
Flutter Hooks を使用していない方は、適宜 `HookWidget` を `ConsumerWidget` に置き換える等していただければと思います。
:::

また、前回はThemeModeとしてデフォルトで定義されている、System / Light / Dark の他にもテーマを追加する例として書きました。
今回はよりシンプルで使う方が多そうなデフォルトで用意されている前述の3種類のテーマ切り替えに絞って実装します。

4種類以上のテーマ状態を実現したい方は、前回の記事を参照していただければ幸いです。

## 概要
System, Light, Dark の3種類をアプリ内で切り替えて、その設定を記憶できるようにする

## 詳しく解説しないこと
テーマの設定詳細（色の指定方法や詳しいテーマデータの定義方法）

## 必要なパッケージを利用する
ユーザーが選択したテーマを記憶しておくために shared_preferences を使用します。

`pubspec.yaml` に追記して `flutter pub get` しましょう。

```yaml:pubspec.yaml
dependencies:
  flutter:
    sdk: flutter

    flutter_hooks: any # 問題無ければ最新バージョンを指定（以下同様）
    flutter_state_notifier: any
    hooks_riverpod: any
    shared_preferences: any
```

## テーマの設定データを用意
ライトテーマ用、ダークテーマ用のThemeDataを定義します。

```dart:light_theme_data.dart
import 'package:flutter/material.dart';
// デフォルトのテーマで問題無い（カスタムしない）場合は `copyWith()` は不要です。
final ThemeData lightThemeData = ThemeData.light().copyWith(
  // 好みの色やテキストのテーマを設定する
  colorScheme: /* 定義 */,
  textTheme: /* 定義 */,
  buttonTheme: /* 定義 */,
);
```

`dark_theme_data.dart` も同じように作成しておきます。

## テーマ選択用のStateNotifierを定義
`Riverpod` と `StateNotifier` を使って、選択されたテーマでアプリ全体を更新できるようにします。

`ThemeSelector` クラスの生成時に、保存されたテーマ情報があれば読み込んで状態に反映させています。
ユーザーによるテーマ変更時には `change` メソッドで状態を更新し、 `shared_preferences` で保存しています。

```dart:theme_controller.dart
import 'package:flutter/material.dart';
import 'package:hooks_riverpod/hooks_riverpod.dart';
import 'package:shared_preferences/shared_preferences.dart';
import 'package:state_notifier/state_notifier.dart';

/// テーマ選択のProvider
final themeSelectorProvider = StateNotifierProvider(
  (ref) => ThemeSelector(ref.read),
);

/// テーマの変更・記憶を行うStateNotifier
class ThemeSelector extends StateNotifier<ThemeMode> {
  ThemeSelector(this._reader) : super(ThemeMode.system) {
    initialize();
  }
  /// SharedPreferences で使用するテーマ記憶用のキー
  static const themePrefsKey = 'selectedTheme';

  // 現状、他のProviderを読み込むことは無いので削除しても良い
  // ignore: unused_field
  final Reader _reader;

  /// 選択されたテーマの記憶があれば取得して反映
  Future initialize() async {
    final themeIndex = await _themeIndex;
    state = ThemeMode.values.firstWhere(
      (e) => e.index == themeIndex,
      orElse: () => ThemeMode.system,
    );
  }

  /// テーマの変更を行い、永続化
  Future change(ThemeMode theme) async {
    await _save(theme.index);
    state = theme;
  }

  /// 現在選択中のテーマを`SharedPreferences`から取得
  Future<int> get _themeIndex async {
    final prefs = await SharedPreferences.getInstance();
    return prefs.getInt(themePrefsKey);
  }

  /// 選択した`SharedPreferences`に保存
  Future<void> _save(int themeIndex) async {
    final prefs = await SharedPreferences.getInstance();
    await prefs.setInt(themePrefsKey, themeIndex);
  }
}
```

### MaterialAppでテーマを指定する
テーマの変更をアプリ全体に反映させるために、 `MaterialApp` でテーマの選択状態を取得し、設定します。

```dart:main.dart
class MyApp extends HookWidget {
  const MyApp();
  @override
  Widget build(BuildContext context) {
    // 現在のテーマを取得
    final themeMode = useProvider(themeSelectorProvider.state);
    return MaterialApp(
      // ダークモードであればダークテーマを指定、
      // ライトモードかシステムモードであればライトテーマを指定することで、
      // ダークモード時は常時ダークテーマが使用されるようになります。
      theme: themeMode == ThemeMode.dark
          ? darkThemeData
          : lightThemeData,
      // 反対に、ライトモードであればライトテーマを指定、
      // ダークモードかシステムモードであればダークモードを指定することで、
      // ライトモード時は常時ライトテーマが使用されることになります。
      darkTheme: themeMode == ThemeMode.light
          ? lightThemeData
          : darkThemeData,
          // 以下略
    )
  }
}
```

## テーマ選択用ページを作成
アプリ内で、ユーザーがテーマを変更できるように、専用のページを作成します。
ここでは、 `ThemeSelectionPage` という名前の `StatelessWidget` にしました。
（この画面のメインパーツである `ThemeListView` は別Widgetとして次に定義します）

```dart:theme_selection_page.dart
import 'package:flutter/material.dart';

import 'theme_list_view.dart';

class ThemeSelectionPage extends StatelessWidget {
  const ThemeSelectionPage();

  static const String routeName = '/theme-selection';

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('テーマ設定')),
      body: const SafeArea(
        child: ThemeListView(),
      ),
    );
  }
}
```

次は、テーマ選択部分のWidgetです。
その前に、デフォルトで用意されている `ThemeMode（システム・ライト・ダーク）` には当然ながら日本語のラベルや説明などは定義されていないため、表示用に extension 機能で定義しておきましょう。

```dart:theme_mode_ext.dart
import 'package:flutter/material.dart';

extension ThemeModeExt on ThemeMode {
  /// タイトル文字列
  String get title {
    switch (this) {
      case ThemeMode.system:
        return 'System';
      case ThemeMode.light:
        return 'Lignt';
      case ThemeMode.dark:
        return 'Dark';
    }
    throw AssertionError();
  }

  /// サブタイトル文字列
  String get subtitle {
    switch (this) {
      case ThemeMode.system:
        return '端末のシステム設定に追従';
      case ThemeMode.light:
        return '白を基調とした明るいテーマ';
      case ThemeMode.dark:
        return '黒を基調とした暗いテーマ';
    }
    throw AssertionError();
  }

  /// アイコン
  IconData get iconData {
    switch (this) {
      case ThemeMode.system:
        return Icons.autorenew;
      case ThemeMode.light:
        return Icons.wb_sunny;
      case ThemeMode.dark:
        return Icons.nightlife;
    }
    throw AssertionError();
  }
}
```

では、テーマ選択部分のWidgetです。

```dart:theme_list_view.dart
import 'package:flutter/material.dart';
import 'package:flutter_hooks/flutter_hooks.dart';
import 'package:hooks_riverpod/hooks_riverpod.dart';

import 'theme_selector.dart';

/// テーマを選択できるリストWidget
class ThemeListView extends HookWidget {
  const ThemeListView();

  @override
  Widget build(BuildContext context) {
    final themeSelector = useProvider(themeSelectorProvider);
    final currentThemeMode = useProvider(themeSelectorProvider.state);
    return ListView.builder(
      itemCount: ThemeMode.values.length,
      itemBuilder: (_, index) {
        final themeMode = ThemeMode.values[index];
        return RadioListTile<ThemeMode>(
          value: themeMode,
          groupValue: currentThemeMode,
          onChanged: (newTheme) {
            themeSelector.change(newTheme);
          },
          title: Text(themeMode.title),
          subtitle: Text(themeMode.subtitle),
          secondary: Icon(themeMode.iconData),
          controlAffinity: ListTileControlAffinity.platform,
        );
      },
    );
  }
}
```

ThemeModeの数（システム・ライト・ダーク）だけリストを作成し選べるように表示しています。

現在選択中のテーマは `useProvider(themeSelectorProvider.state)` で取得できます。

`RadioListTile` Widgetの行タップ時に、`useProvider(themeSelectorProvider).change(newTheme)` メソッドを使ってテーマの変更を反映・記憶させています。

---

最後までご閲覧いただき、ありがとうございました！

Twitterでは、主にFlutter・Firebase・iOS/Swift について呟いております。
フォローしていただければ嬉しいです☺️ → 🐦村松龍之介[@Riscait](https://twitter.com/riscait) 

---
## 修正履歴
2020年12月27日
`light_theme_data.dart` なのに、 `ThemeData.dark()` を使用していたところを修正しました！
ずみこうさん、ありがとうございます🙏

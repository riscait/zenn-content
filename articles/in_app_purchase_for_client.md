---
title: "Flutter x Firebaseでサブスクリプション課金を実装【アプリコード編】"
emoji: "🍣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

## 実装には2つの方法がある
`in_app_purchase.dart`は一般的なFlutterAPI。
最も基本的な機能が使える。
読み込みと購入のほとんどのユースケースを処理できる。

プラットフォームのAPIを可能な限り直接公開する、`store_kit_wrapper.dart`, `billing_client_wrapper.dart` を使用する
細かい制御を可能にするが、使用プラットフォームに応じて大幅に差異のある購入処理ロジックをコーディングしなければならない。

## プラグインの初期化
```dart
void main() {
  // このアプリが Android での[保留中の購入]をサポートしていることをプラグインに通知します。
  // この呼び出しを行わずに `instance` にアクセスすると、Android 上でエラーが発生します。
  // iOSではこれはダメです。
  InAppPurchaseConnection.enablePendingPurchases();
  
  runApp(MyApp());
}
```

```dart
  /// アプリの初期化時に着信する購入を購読します。
  /// これらはどちらのストアフロントからも伝搬する可能性があるので、
  /// イベントを失うことを避けるためにも、できるだけ早く聞くことが重要です。
  StreamSubscription<List<PurchaseDetails>> _subscription;
```

```dart
  @override
  void initState() {
    /// 課金のリアルタイム更新を購読
    /// アプリユーザーによって購入がトリガーされたとき
    /// Storeでユーザーが購入を開始した時
    /// 前回のセッションから購入が完了していない場合
    /// アプリ起動したら直ちに、（App Widgetを返す前 例: `main()`）でこのStreamを購読する必要がある
    /// でなければ、更新された情報を見逃してしまう
    final purchaseUpdates =
        InAppPurchaseConnection.instance.purchaseUpdatedStream;
    _subscription = purchaseUpdates.listen((purchases) {
      _handlePurchaseUpdates(purchases);
    });
    super.initState();
  }
```

```dart
  @override
  void dispose() {
    // 課金情報の購読を破棄
    _subscription.cancel();
    super.dispose();
  }
```

```dart
final bool available = await InAppPurchaseConnection.instance.isAvailable();
if (!available) {
  // ストアにアクセスできない、またはアクセスできない。それに応じてUIを更新する
}
```

```dart
```

```dart
```

```dart
```

## Firebaseの役割
## Firestore
- 定期購入の製品IDの保存
- ユーザーの課金状態の保存
- レシート情報の保存

ユーザーコレクションのサブコレクションとしてレシート情報を保存する

Firestore/users/{uid}/appleReceipt
Firestore/users/{uid}/googleReceipt

Collection Groupで全体のiOSまたはAndroidユーザーのレシートを抽出できる
持ち主を特定しやすくするために、レシート保存時にレシート情報に`uid`も入れておく

Appleは最新レシートの有効期限が切れる24時間前に、自動更新が有効になる。
Pub/Subスケジューラの頻度はFirebaseの料金とも相談しながら設定する。

## Cloud Functions
- onCallトリガー
  - 購入と復元の処理
- onRequestトリガー
  - Apple定期購入のステータス変更通知を処理する
  - ユーザーIDでレシートを検証する
- Pub/Subトピック
  - Google定期購入のステータス変更通知を処理
- Pub/Subスケジューラ
  - スケジューラ実行時間の前後n時間以内に有効期限が含まれるレシートを検証する

## Cloud Storage
- Pub/Subスケジューラで実行したレシートの検証結果をテキストで保存

## 参考リンク
[アプリ内課金の定期購入（サブスクリプション）をFlutterとFirebaseで実装するときのポイント](https://tech.studyplus.co.jp/entry/2020/04/13/102204)
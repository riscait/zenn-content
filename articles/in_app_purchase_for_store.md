---
title: "Flutter x Firebaseでサブスクリプション課金を実装【Store設定編】"
emoji: "🍣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

## パッケージ
https://pub.dev/packages/in_app_purchase

### 対応課金方法

- Consumables(消耗品)
- Permanent Upgrades（永久アップグレード）
- Subscriptions（定期購入）

## 公式ドキュメント
[アプリ内購入（App Store）](https://developer.apple.com/in-app-purchase/)

[Google Play 請求サービスの概要](https://developer.android.com/google/play/billing/billing_overview)

> 定期購入: ユーザーの支払い方法に対して定期的に請求が行われるアプリ内アイテム。定期購入の例としては、オンライン雑誌や音楽ストリーミング サービスなどがあります。定期購入は、Google Play 請求サービス ライブラリでは「SUBS」と呼ばれます。

> 定期購入の場合、初回購入時に購入トークンとオーダー ID が作成されます。後続の各請求対象期間では、購入トークンは同じままで、新しいオーダー ID が発行されます。アップグレード、ダウングレード、再登録の場合は、必ず新しい購入トークンとオーダー ID が作成されます。

## Google Play Billing Library の追加

「tweet](https://twitter.com/riscait/status/1309647355692462081)

[Google Play Billing Library の使用](https://developer.android.com/google/play/billing/billing_library_overview)

```android/app/build.gradle
dependencies {
    ...
    implementation 'com.android.billingclient:billing:2.1.0'
    implementation 'com.android.billingclient:billing-ktx:2.1.0'
}
```

上記 `build.gradle` の変更を含んだAPKをConsoleにアップロードします。

App Bundleではダメでした。

apkを生成するコマンド例：
```
flutter build apk --release
```

[サンプルアプリのREADME](https://github.com/flutter/plugins/blob/master/packages/in_app_purchase/example/README.md)

## Google Play Console
[Play Console ヘルプ / 定期購入の作成](https://support.google.com/googleplay/android-developer/answer/140504)

https://play.google.com/apps/publish/

1. アプリを選択
1. 収益化/商品/定期購入
1. 「販売アカウントをセットアップ」
1. 設定/デベロッパーアカウント/お支払い設定
1. お支払いプロファイルを作成

### アイテム ID
小文字の英数字、アンダースコア、ピリオドで
例： `pro_plan`

### 名前:
例：Proプラン

### 説明
例：お気に入りや保管場所、期限の通知上限数の無制限化、広告の非表示など、より快適にアプリを使うことができます

### 特典:
> 最大4つのメリットを含めることで、ユーザーが購読した際に得られるものを理解できるようになります。

### 請求対象期間
例：月別

### デフォルトの価格
例： JPY 480

### 無料試用
> 支払いが発生する前にユーザーに定期購入を体験してもらうことができます

例：有効・30日

### お試し価格
> 新規に定期購入を申し込んだユーザーは、一定の期間、割引価格で利用できます。
例：なし

### 猶予期間
> ユーザーがお支払いに関する問題を解決するための期間（この間、定期購入は有効のままとなります）
例：7日間

### 再度定期購入
> ユーザーは解約後に Play ストアから定期購入を再開できます。
> アプリの有効な APK の一部で Billing Library 2.0 が使用されていないため、現在、ユーザーは定期購入の再開機能を利用できません。

例：無効

## App Store Connect
App内課金/管理 でサブスクリプションを作成

### 参照名
例：Proプラン

### 製品ID
例： `pro_plan`

### サブスクリプショングループ（グループ参照名）
サブスクリプショングループ参照名
ローカリゼーション

### サブスクリプション期間
例：1ヶ月

### サブスクリプション価格
通過と価格を選択肢の中から選びます。
例えば、日本円で価格を選択すると、その価格を基準として各国の通過で自動計算してくれます。

他国または地域の価格も自動計算されるのでそのまま作成すれば大丈夫。

### App Store情報 / ローカリゼーション
> App Store に表示する App 内課金の表示名および説明を提供してください。
言語ごとに表示名と説明文を設定できます。
#### サブスクリプション表示名
> App Storeに表示されるApp内課金の名前
例：Proプラン

#### 説明
> App 内課金によっては、この説明はカスタマーにも表示される場合があります。
例：より快適にアプリを利用できるプロ向けプラン。各種登録制限数の解除、広告の非表示を含みます。

### App Store プロモーション（任意）
1024x1024pxの販促用画像を追加できます。
Appのプロダクトページ・検索結果への表示、編集チームの特集掲載が期待できます。

### 審査に関する情報
Appleが行う審査に役立つApp内課金に関する情報を追加できます。
スクリーンショットとメモを提示可能。

### お試しオファー
無料お試し期間を提供できる？

開催期間を設定する。
最初に実装するときは当日からで問題ないと思われる。
無料お試しを提供するのが期間限定でなければ、「終了日なし」を選ぶ。

無料トライアルを提供したい場合は「無料」を選択し、「期間」を選択する。
例：無料 - 1ヶ月

### Sandboxテスターの追加
App Store Connect の ユーザーとアクセス に「Sandbox / テスター」という項目があります。
Apple IDと紐づいていない新しいメールアドレスを使ってテスターを追加します。

## Firestore
ユーザーコレクションのサブコレクションとしてレシート情報を保存する

Firestore/users/{uid}/appleReceipt
Firestore/users/{uid}/googleReceipt

Collection Groupで全体のiOSまたはAndroidユーザーのレシートを抽出できる
持ち主を特定しやすくするために、レシート保存時にレシート情報に`uid`も入れておく

Appleは最新レシートの有効期限が切れる24時間前に、自動更新が有効になる。
Pub/Subスケジューラの頻度はFirebaseの料金とも相談しながら設定する。

## 参考リンク
[In App Purchase Example / README.md](https://github.com/flutter/plugins/tree/master/packages/in_app_purchase/example)
[In App Purchase Example / main.dart](https://github.com/flutter/plugins/blob/61775d32f1d1f0997e0e93bb1e8676b96be9d848/packages/in_app_purchase/example/lib/main.dart#L350)
ちょっと古い
[iTunes Connect で App 内課金の設定を行う | Xamarin.iOS](https://itblogdsi.blog.fc2.com/blog-entry-65.html)
[In-App Purchase で 購入履歴を復元する際の注意点 | Xamarin.iOS](https://itblogdsi.blog.fc2.com/blog-entry-72.html)

[【Flutter + Firebase】アプリ内課金(IAP)のステップバイステップ実装ガイド【レシート検証】](https://tkzo.jp/blog/flutter-iap-implementation/)

[Android(Google Play)にてアプリ内定期購入を実装したときに困ったポイント](https://qiita.com/amymd/items/f3f10a5e953d53653010)

Google
課金のテストはリリースビルドで行わなければいけない
アプリ内課金のテストをするには、ベータ版もしくはアルファ版のテスターから実施する

課金リクエストのテストをする場合は、事前に課金で使用するテストアカウントをGoogle Developer Consoleに登録する
『設定 ⇒ テスターのリスト』から追加
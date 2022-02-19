---
title: "Google Play Store向けサービスアカウントの作り方"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [googlecloudplatform,googleplayconsole]
published: true
---

今回、「RevenueCat」というアプリ内課金サービスで使用するために Google Cloud のサービスアカウントが必要だったのですが、いつも忘れてしまったり、合っているか不安になってしまうので作り方、権限の与え方を書き残します✍️

以下、2020年11月4日最新の情報で作成しましたが、今後Googleによるアップデートで画面構成が変更される可能性があります。
その際はコメント等でお教えいただければ大変嬉しいです。

## Google Play Console
Google Play Console にログインし、
すべてのアプリ → 設定 → デベロッパーアカウント → API アクセス を開きます。

![](https://storage.googleapis.com/zenn-user-upload/s2scnusnw2riixuczekg9d9j11r0 =300x)

Google Play アカウントと Google Developer Project をまだ接続していない場合は、画面上のボタンを押して「接続」します。

### サービスアカウントの作成
![](https://storage.googleapis.com/zenn-user-upload/8fx5fmia7nl8x1vh6dassrvactul =600x)

「新しいサービスアカウントを作成」ボタンを押して、

![](https://storage.googleapis.com/zenn-user-upload/9stpubgn7o797y618tzpe0r4pfsc =600x)

表示されるポップアップの「Google Cloud Platform」リンクを選択します。

![](https://storage.googleapis.com/zenn-user-upload/t765x1fy0wg7fo8vot7nl0fvvjzm =600x)

Google Cloud Platform に移動したら、「＋サービスアカウントを作成」を選択します。

![](https://storage.googleapis.com/zenn-user-upload/5hd07ave3bw7afi7ozkpcvsxscao =600x)

サービスアカウント名・サービス アカウント ID・サービス アカウント ID を入力します。

サービスアカウントIDは、サービスアカウント名を入力すると自動入力されますが、任意の文字列へ変更可能です。

![](https://storage.googleapis.com/zenn-user-upload/n3ra6n5jif8lbbgv3t7wfymv0c45 =600x)

「作成」で権限設定へ移ります。

適したロールを選択しましょう。
RevenueCatの場合はOwner（オーナー）が必要です。

![](https://storage.googleapis.com/zenn-user-upload/uhwoplbomxuoe58ncbcoo7w7f5ee =600x)

RevenueCatの場合は、こちらは入力不要です。「完了」しましょう。

### 秘密鍵の作成とJSONダウンロード
![](https://storage.googleapis.com/zenn-user-upload/4vfh5zqkcz1b887ri6dz7qaeojtq =600x)

作成したキーのメニューボタンを押し、

![](https://storage.googleapis.com/zenn-user-upload/5j304j9wrf43cewfc3njxkwfjb0c =600x)

JSONを選択して作成します。

作成するとダウンロードされてPCに保存されます。
紛失しないように適宜、移動しておきましょう。

## RevenueCatへの財務アクセスを許可する
![](https://storage.googleapis.com/zenn-user-upload/7lirmt948m8eav5fcueq3v2l4pcy =600x)

Google Playコンソールにて、作成したサービスアカウントの「アクセスを許可」を押します。

![](https://storage.googleapis.com/zenn-user-upload/h0hzl1hfqazvcd2whq7gywtknxnk =600x)

「ユーザーと権限/ユーザーを招待」画面へ移動しました。

「アカウントの権限」設定をしていきます。

```
一例として、RevenueCatの場合は、

- アプリ情報の閲覧、一括レポートのダウンロード（読み取り専用）
- 売上データ、注文、解約アンケートの回答の閲覧
- 注文と定期購入の管理

の3つが必要な権限なので、上記3つにチェックを入れます。
```

チェックを入れたら、「ユーザーを招待」ボタンを押し、
「招待状を送信」しましょう。

![](https://storage.googleapis.com/zenn-user-upload/kl4vjfwwluyb9hgbbs2z5kjfsd48 =600x)

最後に「変更を保存」するのを忘れないようにして、ユーザーの権限を更新しましょう。

### アプリに権限を適用する
![](https://storage.googleapis.com/zenn-user-upload/eo2wc49r3shdl2klhadua77lxpwa =600x)

「ユーザーと権限」画面で、作成したユーザーを選択します。

![](https://storage.googleapis.com/zenn-user-upload/6ydkpwnjww8wedtaf3knsx7muvle =600x)

「ユーザーの詳細」画面に移動したら、「アプリの追加」をタップ。
使用したいアプリにチェックします。

![](https://storage.googleapis.com/zenn-user-upload/iz7shlaek9646a28i8b8au5i980e =600x)

権限の確認画面が表示され、先ほど選択した権限にチェックが入っていることを確認します。

これにて Google Play Console 側での準備は完了です！

## 使いたいサービスでサービスアカウントの情報を入力する
![](https://storage.googleapis.com/zenn-user-upload/3ja285lx3mri0q85m66dm8l91tzt =600x)

例として、RevenueCatでは、Android設定として、アカウント作成時に↑のように「Service Account credentials JSON」に、JSONファイルの内容（文字列）を入力します。

以上です。

## 参考リンク
https://docs.revenuecat.com/docs/creating-play-service-credentials#2-create-service-account

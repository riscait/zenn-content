---
title: "個人開発スマホアプリの振り返り2020/09"
emoji: "🐿"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: [firebase]
published: true
publication_name: "altiveinc"
---

こんにちは、 村松龍之介[@Riscait](https://twitter.com/riscait) です。

平日は業務としてiOSネイティブアプリの開発を行うかたわら、
個人開発としてFlutter x Firebase でiOS & Androidアプリを作っています。

**Flutter x Firebaseでの個人開発**の実績を振り返り、次月の目標を立てていきたいと思います。

**つまり、「ポエム」です。** 
ZennのIdeaに投稿して良いよね…？

前回は note に投稿しました。[前月の記事はこちら - note](https://note.com/riscait/n/n9de617bdc14d)

## リストック - 備蓄管理アプリの更新頻度
![リリース履歴](https://storage.googleapis.com/zenn-user-upload/wph1q466go8u3gsm94jp2z2hfvgm =320x)

9月は4回のアプリアップデートリリースを行うことができました！
約1週間に1度なので、良いペースを維持できました🙌

### 新しいメイン機能である「みんなのストック」を追加しました！
![](https://storage.googleapis.com/zenn-user-upload/lkpnvlj57emubol3srd0468d5wxl =320x)

他のユーザーが登録した商品が時系列で見られるタイムライン機能です。

Firebase Cloud Functionsを使いました。
ユーザーのストック登録をトリガーに、新規ドキュメントを作成していきます。

アプリでの表示は20件のみの表示+手動更新としています。

本当はTwitterみたいに、新しい投稿があった場合は画面上部に通知し、スクロールすると更新させたかったのですが…
Streamにするとユーザーが増えたときにReadがやばそうだし、Futureだと更新が難しくて、折衷案にしました🤔

#### ニックネーム
最初の実装では、ユーザーに関係なくランダムなニックネームを表示させていただけでした。
アップデートで、実際にユーザーにランダムなニックネームを付与して、
さらに変更できるようにしていきました。

素で豊富なニックネームパターンを用意するのは大変すぎるので、
自然な組み合わせになるような 称号×役職 を作成してパターンを増やしました💡

想像以上に面白い組み合わせが出てきたりして気に入っています😊

### 保管場所
![](https://storage.googleapis.com/zenn-user-upload/z6w4hndx7b7ierrmii5oo9c26hh7 =320x)

ユーザーさんから

> 今はメモに保管場所を記入しているが、それとは別に「保管場所」を登録したい

という旨の要望が届きましたので、追加しました！

ストック追加・編集時に「保管場所」を選ぶことができ、
最新バージョンでは、ストック一覧・ストック詳細で保管場所を確認できます。

しかし、保管場所による絞り込みor並び替え機能はまだ実装できていないので、今後追加したいと思っています。

保管場所の名前は、Cloud Firestoreのユーザードキュメントに配列（String）で保存しています。

しかし、これだと後からユーザーが「保管場所」の名前を変更した時に、すでに登録済みストックの保管場所名を更新するのが手間なので、
辞書（Index: String）で保存した方が良かったかなぁなどと考えました…🤔

### お気に入り機能
![](https://storage.googleapis.com/zenn-user-upload/3a4a90v7htqel7p5kb9sx0zz8dx3 =320x)

ずっと追加したかったお気に入り機能を追加できました。
商品を検索した時など、「今すぐ登録できないけどあとで登録したい」と思った商品などを登録しておけます。

アイコンに悩みました。
- ❤️（Heart）
- ⭐️（Star）
- 🔖（Bookmark）

最終的には、一番よく見る⭐️に落ち着きました🧐
⭐️は評価、🔖はページ保存のイメージかな…と

## イラストを挿入
![空の場合の画面](https://storage.googleapis.com/zenn-user-upload/cvdf71frzu7nlw1w7mrl90j26ol3 =320x)
Twitterで見かけた「[loosedrawing](https://loosedrawing.com/)さんのイラスト
（統一されたテイストの素敵イラストがなんと無料…！）を使って、
無味乾燥だった空の画面に追加して彩りました。

![レビュー催促画面](https://storage.googleapis.com/zenn-user-upload/ugdhkeol0t93a4bfxlzxm9jlurx1 =320x)

その他、いくつかの不具合や表示崩れを修正しました

## リストックの利用ユーザー数
![ユーザー数](https://storage.googleapis.com/zenn-user-upload/e1ve3ogsv5q1lpo1699sy3y8taz0 =480x)

前回の利用ユーザー: 1030
前回の新規ユーザー: 985

今回の利用ユーザー: 560
今回の新規ユーザー: 413

利用ユーザー・新規ユーザー共に下がっている…😿
これは離脱ユーザーが多いってことですね。すっかり入れ替わっているのかもしれない。

![ユーザー数の推移](https://storage.googleapis.com/zenn-user-upload/3szo0h3qouw9xwo6g32lde6650n7)

9月20日でガツンと下がっているのはなんだろうか🧐
強制アップデートと関係あったり…？

![ユーザーのロイヤリティ](https://storage.googleapis.com/zenn-user-upload/gsdrosh1wvupo1wasbjy3hi04ysa =320x)
DAU/MAU 7.6% って少ないですよね…？🙀

プラットフォーム内訳としては、変わらずiOSが3/4を占めています。

## リストックのレビュー
![評価とレビュー4.2](https://storage.googleapis.com/zenn-user-upload/xa31e8qg9hqs9wjazpqdr1i9600v =400x)
前回は⭐️3.9 でしたが、⭐️4.2 まで上昇しました！
⭐️5レビューしていただいた方、もし見ていただいていたらありがとうございます！

![5レビュー](https://storage.googleapis.com/zenn-user-upload/i6uviehnxgpznbvzffzsq8gsufen =320x)
![5レビュー](https://storage.googleapis.com/zenn-user-upload/me5igloldqty54ktbujxa872uwwz =320x)

## リストックのASO
![備蓄の検索結果画面](https://storage.googleapis.com/zenn-user-upload/7w9zc4zuim0476bsm0wh1owo72vc =320x)
リストックは「備蓄」キーワードで1位表示されます。
しかし、ダウンロードは伸びないので、あまり良いキーワードとは言えないのかもしれないですね🤔
関連キーワードも表示されないですし…

「期限」とか「賞味期限」や「防災」など検索数が多そうなワードを狙っていった方が良いのだろうか🧐

## 収益
9月の収入は0円でした！（App内課金・広告がないので当然です）
今サブスクリプション（定期購入）を実装中です💪

## 支出
![Google Cloud請求書](https://storage.googleapis.com/zenn-user-upload/9ekhjbt071kg3sica5v2be2bv2nq)
初めてGoogle Cloud Platformから請求が届きました！
3円クレジットカードから引き落とされます…！
超小額といえど、感慨深いですし、ちょっと怖いですね😸💦

いずれは最適化しないといけない日が来そうですね✍️

## 9月の目標結果
先月は以下の目標を立ててリストックの開発に臨みました。

- アプリのTwitterアカウントを開設して活用
　　- [リストックAppのTwitterアカウント](https://twitter.com/ReStockApp)
- お気に入り機能のフル活用
- 保管方法の登録機能を実装

Twitterは妻に運用を任せています！フォローしていただいた方ありがとうござます！
備蓄に関する有用情報やアプリのアップデート情報を呟いていきます。

お気に入り機能の追加実装と保管場所の登録機能も実装できました！🎉
目標達成です！！

> 備蓄は常温保存のものが多いと思いますが、冷蔵・冷凍と区別できたら使いやすくなるかな、と考えているので実装予定です💪

先月はこんな感じで保存方法を登録できるようにしようと思っていましたが、より柔軟な「保管場所」という形で実装しました。

保管場所の設定は3カ所までとしています。
これは、App内課金を実装した際に制限解放機能として持たせるためです。

## 10月の目標

- App内課金を実装する

今月の目標はこれに限ります！
アプリ内実装だけでなく、サーバー側（Functions）の実装があるので、難しい…
しかしこの壁を越えて課金機能を実装できれば、大きな一歩だと思って頑張ります！！

以上、今月の個人開発月報でした！

最後までご閲覧ありがとうございます🙇‍♂️

Twitterでも、主にFlutterとFirebaseについて呟いております！
フォローしていただければ嬉しいです☺️ → 🐦村松龍之介[@Riscait](https://twitter.com/riscait) 
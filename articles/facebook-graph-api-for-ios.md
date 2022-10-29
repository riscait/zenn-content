---
title: "iOSでFacebook/InstagramのGraph APIを使用して写真を取得する"
emoji: "😊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["UIKit", "Swift"]
published: false
publication_name: "altiveinc"
---

## Facebook for Developers

https://developers.facebook.com/

## iOS用Facebook SDK - スタートガイド
https://developers.facebook.com/docs/ios/getting-started

iOS用SDKのリポジトリ
https://github.com/facebook/facebook-ios-sdk

Xcode 11.2以上であればSwift Package Managerを使用可能。
SDKバージョンは、v5.11.0以上を指定しましょう。

Objective-CからFacebook SDKを使用したい場合は、残念ながらSPMは使用できないので、
CocoaPodsを利用しましょう。

```PodfilePodfile
 pod 'FBSDKCoreKit'
 pod 'FBSDKLoginKit'
 pod 'FBSDKShareKit'
```


## ユーザー情報を取得する

> /meノードは特別なエンドポイントで、現在アクセストークンがAPI呼び出しに使われている利用者のuser_id(またはFacebookページのpage_id)に変わります。ユーザーアクセストークンを持っていれば、次の> コードでユーザーのすべての写真を取得できます。

ドキュメント - User
https://developers.facebook.com/docs/graph-api/reference/user

## グラフAPIエクスプローラ
[グラフAPIエクスプローラ](https://developers.facebook.com/tools/explorer)

[iOSでのグラフAPIの使用](https://developers.facebook.com/docs/ios/graph)

[iOS SDK - エラーの処理](https://developers.facebook.com/docs/ios/errors)

## SDK Version, API Versionの確認
「アプリダッシュボード」ページの設定/詳細設定から確認できます。



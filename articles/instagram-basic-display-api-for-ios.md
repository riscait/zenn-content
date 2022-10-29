---
title: "iOSでInstagram基本表示APIを使用して写真を取得する"
emoji: "😊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["UIKit", "Swift"]
published: false
publication_name: "altiveinc"
---

## Facebook for Developers

https://developers.facebook.com/

## Instagram用のAPIは2種類ある
- ビジネス向けの「InstagramグラフAPI」
- 消費者向けの「Instagram基本表示API」
今回はアプリユーザーの写真を使用することが目的なので、「Instagram基本表示API」を使用する

## 基本表示APIとグラフAPIの違い

[Instagram基本表示API](https://developers.facebook.com/docs/instagram-basic-display-api)

- 基本表示APIは、個人ユーザー向け。アクセストークンの有効期限が1時間かつリフレッシュトークンがない？
- Instagram Graph APIはFacebookアカウントを利用する
- Instagram Graph APIは1080pxの画像が取得できる

## Instagram基本表示APIはHTTPベースのAPI
ベースURL
- api.instagram.com — Instagramユーザーアクセストークンの取得用
- graph.instagram.com — Instagramユーザープロフィールとメディアの取得用

## 必要なもの
- Facebook開発者アカウント
- メディアがあるInstagramアカウント

## Graph API
- エンドポイントは、graph.facebook.comホストからアクセス

前提条件
始める前に、以下を用意する必要があります。

Facebookページ
ページの役割
Instagramアカウント

Facebookからアクセストークンを取得するには、最終的にFacebookログインを実装する必要があります。FacebookのSDKを使用すると簡単に実装できます
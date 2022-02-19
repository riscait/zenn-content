---
title: "iOSにおける「触覚タッチ」と「3D Touch」についてのおさらいと無効化方法"
emoji: "☝️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["UIKit", "Swift"]
published: true
---
こんにちは、[村松龍之介@Riscait](https://twitter.com/riscait)です。

この記事は、 [iOS Advent Calendar 2020 | Qiita](https://qiita.com/advent-calendar/2020/ios) 19日目の記事です💪

今回、3D Touch と触覚タッチについて差異を意識しながら無効化しなければいけない機会がありましたので、備忘録がてらまとめてみました。

## 3D Touchとは
3D Touchは、圧力（ディスプレイの押し込み）を検知できる機能です。
特定の端末のみ利用できる機能で、直近のiPhone端末では廃止されています。

### 3D Touchが使える端末
* iPhone 6s, iPhone 6 Plus
* iPhone 7, iPhone 7 Plus
* iPhone 8, iPhone 8 Plus
* iPhone X,
* iPhone XS, iPhone XS Max

上記の端末は、iOS 13以上であれば後述の触覚タッチも利用可能です。

## 触覚タッチ (Haptic Touch)とは
長押しを検知し振動させる（圧力は検知できない）機能です。
振動させることにより、擬似的に押し込みを表現しています。
3D Touchと違い、特定の端末のみというわけでなく、iOS 13以上で利用可能です。

例えば、

* iPhone XR や、
* iPhone 11, iPhone 11 Pro, iPhone 11 Pro Max 含む新しい端末

は、**「3D Touch」には対応していない**ため、「触覚タッチ」のみ利用可能です。

## iPhone 6 以前の端末かつ iOS 13 未満は、どちらも使えない
iPhone 6, iPhone 6 Plus とそれ以前の端末では、3D Touch非対応です。
さらに iOS 13未満の場合は触覚タッチも使えないため、どちらも使えないことになります。

## 3D Touchが使えるデバイスなのかチェックする方法
3D Touchは特定の端末のみ利用可能です。
さらに設定でOFFにもできるため、利用可能状態にあるか、コード上で確認する必要があります。

[`UITraitCollection`](https://developer.apple.com/documentation/uikit/uitraitcollection) の [`forceTouchCapability`](https://developer.apple.com/documentation/uikit/uitraitcollection/1623515-forcetouchcapability)で確認できます。

UIViewController上であれば、 `.traitCollection` がそのまま使えます。

結果として返ってくるenumは以下の3つです。
[`UIForceTouchCapability`](https://developer.apple.com/documentation/uikit/uiforcetouchcapability)

```swift
switch traitCollection.forceTouchCapability {
case .unknown:     // 0: unknown（不明）
case .unavailable: // 1: unavailable（利用不可）
case .available:   // 2: available（利用可能）
@unknown default:
}
```

[UIButtonに3DTouch判定を追加してアナログにコントロールしたい](http://tazk.hatenablog.com/entry/2015/09/26/231430)
また、こちらの記事にはObjective-Cだと、 `viewDidLoad` では判定ができないとありましたが、僕の環境ではObjCでも判定可能でした。

## 3D Touchと触覚タッチによるプレビューを無効化する方法
[`allowsLinkPreview`](https://developer.apple.com/documentation/webkit/wkwebview/1415000-allowslinkpreview)
を使用します。

iOS 10以降では、デフォルトでtrueとなっているので、 `allowsLinkPreview = false` とすればOKです。

タッチ対象によりプレビューを表示させるかどうか動的に振り分けたい場合は、　`iOS 10.0–13.0 (Deprecated)` ですが、いかのAPIがあります。
[`optional func webView(_:shouldPreviewElement:) -> Bool`](https://developer.apple.com/documentation/webkit/wkuidelegate/1648359-webview)

## 余談
### WKWebViewで開けるActionSheetを無効にする
WKWebViewでは、3D Touchや触覚タッチを使うとリンクをプレビューできますが、こちらも `allowsLinkPreview = false` で無効化できます。

3D Touch, 触覚タッチともに非対応な端末ではどうなるかというと、代わりに ActionSheet が開きます。
これを無効にする方法は以下になります。

WKWebView初期化時にスクリプトを注入します。

```swift
// 3D Touchと触覚タッチに未対応のデバイスの長押しで表示されるActionSheetを無効化
let contentController = WKUserContentController()
// 注入したいJavaScriptを定義してコントローラーに追加
let userScript = WKUserScript(
    source: "document.documentElement.style.webkitTouchCallout='none';",
    injectionTime: .atDocumentStart,
    forMainFrameOnly: true
)
contentController.addUserScript(userScript)
// WKWebViewの初期化時に指定するConfigurationクラスにコントローラーをセット
let webViewConfig = WKWebViewConfiguration()
webViewConfig.userContentController = contentController
// Configurationを使用してWKWebViewを生成
webView = WKWebView(frame: .zero, configuration: webViewConfig)
```

```objective-c
// 3D Touchと触覚タッチに未対応のデバイスの長押しで表示されるActionSheetを無効化
WKUserContentController *contentController = [[WKUserContentController alloc] init];
// 注入したいJavaScriptを定義してコントローラーに追加
WKUserScript *script = [[WKUserScript alloc]
                        initWithSource:@"document.documentElement.style.webkitTouchCallout='none';"
                        injectionTime:WKUserScriptInjectionTimeAtDocumentStart
                        forMainFrameOnly:YES];
[contentController addUserScript:script];
// WKWebViewの初期化時に指定するConfigurationクラスにコントローラーをセット
WKWebViewConfiguration * config = [[WKWebViewConfiguration alloc] init];
[config setUserContentController:contentController];
// Configurationを使用してWKWebViewを生成
self.webView = [[WKWebView alloc] initWithFrame:CGRectZero configuration:config];
```

## 参考リンク
[Making Javascript feel like native iOS with WKWebView](https://medium.com/@maxcampolo/making-javascript-feel-like-native-ios-with-wkwebview-d92dfefe8d56)

ご閲覧ありがとうございます！

Twitterでは、主にFlutter・Firebase・iOS/Swift, について呟いております。
フォローしていただければ嬉しいです☺️ → 🐦村松龍之介[@Riscait](https://twitter.com/riscait) 
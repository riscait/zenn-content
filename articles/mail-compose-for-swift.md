---
title: "iOSアプリ内でメール画面を開く [MFMailComposeViewController]"
emoji: "📧"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Swift"]
published: true
publication_name: "altiveinc"
---
「アプリ内でメーラーを立ち上げて、ユーザー情報とともにサポートへ送信してもらう」
よくある実装だと思うのですが、久しぶりにメールを作成する処理の実装を行ったので備忘録として書き残しておきます。

## 公式ドキュメント
[MFMailComposeViewController](https://developer.apple.com/documentation/messageui/mfmailcomposeviewcontroller)
[MFMailComposeViewControllerDelegate](https://developer.apple.com/documentation/messageui/mfmailcomposeviewcontrollerdelegate)

とにもかくにもメール送信画面を立ち上げる処理を書きます。
メーラーを表示させたい画面 (ViewController) にメソッドを作成します。

```swift:ViewController.swift
private func composeMail() {
    // まずは、 `canSendMail()` でメールが送信できる状態か確かめる
    guard MFMailComposeViewController.canSendMail() else {
        // 有効なメールアドレスがないため、メール送信はできない
        return
    }
    // メール送信画面を作成
    let composer = MFMailComposeViewController()
    // あとで実装するデリゲートを自分 (ViewController) に設定
    composer.mailComposeDelegate = self
    // 宛先 (TO・CC・BCC) を配列で指定
    composer.setToRecipients(["to@example.com"])
    composer.setCcRecipients(["cc@example.com"])
    composer.setBccRecipients(["bcc@example.com"])
    // 件名
    composer.setSubject("件名文字列")
    // 本文
    composer.setMessageBody("本文文字列", isHTML: false)
    // 表示
    present(composer, animated: true, completion: nil)
}
```

## メーラーのデリゲートメソッドを実装します。

1. メーラーを表示させたい `ViewController` を extension で `MFMailComposeViewControllerDelegate` に準拠させます。
2. `mailComposeController(_:didFinishWith:error:)` メソッドを実装します。

```swift:ViewController.swift
extension ViewController: MFMailComposeViewControllerDelegate {
    // メール送信結果をハンドリング
    func mailComposeController(_ controller: MFMailComposeViewController,
                               didFinishWith result: MFMailComposeResult,
                               error: Error?) {
        if let error = error {
            // エラーハンドリング
        }
        // 結果をハンドリング
        switch result {
        case .cancelled: // キャンセル
            // 任意の処理
        case .saved: // 下書き保存
            // 任意の処理
        case .sent: // 送信された
            // 任意の処理
        case .failed: // 送信失敗
            // 任意の処理
        @unknown default:
            break
        }
        // メール画面を閉じる
        controller.dismiss(animated: true, completion: nil)
    }
}
```

## コード全体像

```swift:ViewController.swift
import UIKit

class ViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        // 省略
    }
}

// MARK: MFMailComposeViewController

extension ViewController: MFMailComposeViewControllerDelegate {
    /// メール送信画面を開く
    private func composeMail() {
        guard MFMailComposeViewController.canSendMail() else {
            // 有効なメールアドレスがないため、メール送信できない
            return
        }
        // メール作成
        let composer = MFMailComposeViewController()
        composer.mailComposeDelegate = self
        // 宛先 (TO・CC・BCC)
        composer.setToRecipients(["to@example.com"])
        composer.setCcRecipients(["cc@example.com"])
        composer.setBccRecipients(["bcc@example.com"])
        // 件名
        composer.setSubject("件名文字列")
        // 本文
        composer.setMessageBody("本文文字列", isHTML: false)
        present(composer, animated: true, completion: nil)
    }
    
    // メール送信結果をハンドリング
    func mailComposeController(_ controller: MFMailComposeViewController,
                               didFinishWith result: MFMailComposeResult,
                               error: Error?) {
        if let error = error {
            // エラーハンドリング
        }
        
        switch result {
        case .cancelled: // キャンセル
            break
        case .saved: // 下書き保存
            break
        case .sent: // 送信された
            break
        case .failed: // 送信失敗
            break
        @unknown default:
            break
        }
        // メール画面を閉じる
        controller.dismiss(animated: true, completion: nil)
    }
}
```
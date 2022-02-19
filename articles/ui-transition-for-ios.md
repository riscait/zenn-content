---
title: "UIKitのカスタムビューで作成したダイアログをアニメーションを伴って表示する方法"
emoji: "😊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["UIKit", "Swift"]
published: false
---

UIKitで用意されているダイアログでカバーできない独自デザインのダイアログを表示したいときの実装例になります。
- ダイアログを画面中央に表示する

## 表示したいView
今回表示するダイアログは`UIViewController`を継承したものとします

```swift
final class ExampleDialog: UIViewController {
    /// ポップアップするダイアログのベースとなるView
    let contentView = UIView()

    // 以下、Viewの実装が続く
}
```

## 表示用の遷移アニメと閉じる用の遷移アニメを定義する
```swift
    final class PresentedTransition: NSObject, UIViewControllerAnimatedTransitioning {
        
        func transitionDuration(using transitionContext: UIViewControllerContextTransitioning?) -> TimeInterval {
            // アニメーションの持続時間
            return 0.4
        }
        
        func animateTransition(using transitionContext: UIViewControllerContextTransitioning) {
            guard
                // 遷移先のViewController
                let vc = transitionContext.viewController(forKey: .to) as? ExampleDialog,
                // 遷移先ViewControllerのView
                let view = vc.view else {
                    transitionContext.completeTransition(true)
                    return
            }
            // アニメーションを描画するためのコンテナービューを取得
            let containerView = transitionContext.containerView
            // コンテナービューに遷移先のViewを乗せる
            containerView.addSubview(view)
            // 遷移先Viewの本来の背景色を記憶しておく
            let backgroundColor = view.backgroundColor
            // 遷移先のダイアログコンテンツ部分
            let contentView = vc.contentView
            // 遷移先のViewの背景色を透明にする
            view.backgroundColor = .clear
            // 遷移先のダイアログコンテンツ部分を透明化
            contentView.alpha = 0.0
            // バウンス表示のため、あらかじめ少し縮小しておく
            contentView.transform = CGAffineTransform(scaleX: 0.8, y: 0.8)
            // コンテナービュー上でアニメーションを描画
            UIView.animate(
                withDuration: transitionDuration(using: transitionContext),
                delay: 0.0,
                usingSpringWithDamping: 0.6,
                initialSpringVelocity: 0.0,
                options: [],
                animations: {
                    // アニメーションを伴って遷移先の背景色を元に戻す
                    view.backgroundColor = backgroundColor
                    // アニメーションを伴ってダイアログ部分を可視化する
                    contentView.alpha = 1.0
                    // アニメーションを伴ってダイアログのサイズを元に戻す
                    contentView.transform = .identity
                },
                completion: { finished in
                    // トランジションが完了したことを通知
                    transitionContext.completeTransition(finished)
                }
            )
        }
    }
```

## UIViewControllerTransitioningDelegateプロトコルに適合する
ViewController間の固定長・またはインタラクティブな遷移を管理するために使用されるオブジェクトを提供するメソッドのセット。

ViewControllerを表示するさいに使用する遷移アニメーターオブジェクトを指定する
```swift
    func animationController(
        // 画面上に表示されようとしているViewControllerのオブジェクト
        forPresented presented: UIViewController,
        // 表示されたViewController
        presenting: UIViewController,
        // `present(_:animated:completion:)`メソッドが呼び出されたViewController
        source: UIViewController
    ) -> UIViewControllerAnimatedTransitioning? {
        return PresentedTransition()
    }
```
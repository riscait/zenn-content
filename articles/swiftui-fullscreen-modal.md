---
title: "SwiftUIでフルスクリーンで画面遷移して閉じる"
emoji: "👏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---
##

```swift
import SwiftUI
struct ContentView: View {
    @State private var isPresented = false
    var body: some View {
        Button("Present") {
            isPresented.toggle()
        }
        .fullScreenCover(isPresented: $isPresented) {
            ModalView()
        }
    }
}
struct ModalView: View {
    @Environment(\.presentationMode) var presentationMode
    var body: some View {
        Button("Dismiss") {
            presentationMode.wrappedValue.dismiss()
        }
        .frame(maxWidth: .infinity, maxHeight: .infinity)
        .background(Color.red)
        .foregroundColor(Color.white)
        .edgesIgnoringSafeArea(.all)
    }
}
```

## 参考
[Full Screen Model in SwiftUi](https://developer.apple.com/forums/thread/650453)
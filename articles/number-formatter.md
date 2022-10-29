---
title: "NumberFormatter"
emoji: "👏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
publication_name: "altiveinc"
---

```swift
let formatter = NumberFormatter()
```

### `formatter.numberStyle`

`.currency`
通貨のフォーマット。通貨記号を表示する
表示例： `$1,234.57`, `1 234,57 €`

`.currencyAccounting`
通貨のフォーマット。負の数値を () で囲むこと以外は `.currency` と同じ。
表示例： `($1,234.57)`, `(1 234,57 €)`

`currencyPlural`
通貨のフォーマット。通貨記号ではなく通過文字列を表示する以外は `.currency` と同じ。
表示例： `1,234.57 US dollars`, `1 234,57 euros.`

`currencyISOCode`
通貨のフォーマット。ISO 4217 通貨コードを表示する
表示例： `USD1,234.57`, `1 234,57 EUR`


### `formatter.percentSymbol`
デフォルトの `%` 記号を変更できる
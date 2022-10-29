---
title: "firestore_refの使い方"
emoji: "🐈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
publication_name: "altiveinc"
---

[pub.dev](https://pub.dev/packages/firestore_ref)
[GitHub](https://github.com/mono0926/flutter_firestore_ref)

## Riverpod
### Provider
```dart
final appUserProvider = StreamProvider.autoDispose((ref) {
  final user = ref.watch(authProvider.state);
  if (user == null) {
    return const Stream<Document<AppUserModel>>.empty();
  }
  return appUserRef.docRefWithId(user.uid).document();
});
```

## 
`docRefWithId([String id])`

## Flutter 1.22.0
```dart
TextField(
  // Before
  inputFormatters: [WhitelistingTextInputFormatter.digitsOnly],
  // After
  inputFormatters: [FilteringTextInputFormatter.digitsOnly],
)
```
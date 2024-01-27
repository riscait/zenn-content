---
title: "OpenAPIからGoとDart(Flutter)のコードを生成し、横断的かつスピーディーにアプリ開発する"
emoji: "🐈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [go, dart, flutter, openapi]
published: true
publication_name: "altiveinc"
# cSpell:words oapi codegen redocly GORM println Sprintf sqlx openapi sqlc
---
## はじめに

こんにちは、Flutterでのアプリ開発をメインとしている「[Altive株式会社](https://altive.co.jp)」の村松龍之介（[@riscait](https://x.com/riscait)）です！

![](/images/ProfileBanner_Muramatsu.jpg)

この記事は、 [Go 言語 Advent Calendar 2023 | Qiita](https://qiita.com/advent-calendar/2023/go) 17日目の記事です🚀

弊社では、Flutterアプリだけでなく、サーバーサイドも自前で作る方針となりました。

そのさい言語の選定を行ない、Goを採用しました。

## アプリ開発者がサーバーサイドを学習しようと思った理由

### アプリ（フロント）サイドとサーバーサイドが分かれているデメリットを感じて

アプリを作る際、弊社にはサーバーサイド人員がいなかったため、サーバーサイドは別会社やパートナーさんが担当してくださることが多かったです。

そうなると、Web APIの仕様策定や実装完了を待つ必要があったり、リクエストやレスポンスのフィールド修正の依頼→修正→動作確認のサイクルが発生したり、
スピーディーな開発の妨げともなり得るボトルネックに感じていました。

また、アプリサイドがサーバーサイドの実装を理解していないと、APIの仕様や実装の修正を依頼することが難しかったりするので両方やれることに越したことはないですね。

## サーバーサイドの実装に「Go」を選んだ理由

### 静的型付け言語であること

僕たちAltiveはアプリ開発の会社ということもあり、JavaやiOSのSwift、FlutterのDartなど、主要な経験が静的型付け言語に集中していました。
コンパイラによる型チェックの恩恵を引き続き受けることができるGoは第一候補として上がりました。

### FirebaseやGoogle Cloudとの親和性

また、特にアプリ開発と相性の良いFirebaseや、Google CloudでGoが扱いやすそうだと思ったのも理由の一つです。

まだ実際に利用できていませんが、AWSでも使えると思うのでその点も良いですね。

:::message
DartもGoもGoogle製の言語なので、Googleへの依存度が高くなるのは正直少し心配ではあります🥺
:::

Go言語の学習には以下の書籍と資料に助けていただきました。この場を借りて感謝申し上げます。

- [Go言語プログラミングエッセンス エンジニア選書](https://www.amazon.co.jp/dp/B0BVZCJQ4F?ref_=cm_sw_r_cp_ud_dp_3AZHTYJC6ZCD5HBR7BR3)
- [詳解Go言語Webアプリケーション開発](https://www.amazon.co.jp/dp/4863543727?ref_=cm_sw_r_cp_ud_dp_9XWJGPHZNXV4D7HN8B72)
- [Gopher道場｜プログラミング言語Go完全入門](http://tenn.in/go)

## Dartと比べたGoの記法や文化の違い

簡単にですが、DartとGo片方しか触ったことがない人向けに、DartとGoの記法や文化の違い比較してみました。

### 変数宣言・代入

```dart
// Dart

const condition = true;

void main() {
  const name = 'Alice';

  int age;
  if (condition) {
    age = 30;
  } else {
    age = 40;
  }
}
```

```go
// Go

const condition = true

func main() {
	name := "Alice"

	var age int
	if condition {
		age = 30
	} else {
		age = 40
	}
}
```

Goの初期化と代入を一緒にやる `:=` には最初驚きましたが、慣れると便利ですね。

また、Goの方がセミコロンやifの括弧が不要だったりと、モダンな印象を受けました。

### 関数・メソッド

```dart
// Dart

String greet(String name) {
  if (condition) {
    throw Exception('error');
  }
  return 'Hello, $name';
}

void main() {
  greet('riscait');
}
```

```go
// Go

func greet(name string) (*string, error) {
  if condition {
    return nil, errors.New("error")
  }
  r := "Hello, " + name
	return r, nil
}

func main() {
  greet("riscait")
}
```

Goでは関数の戻り値に多値を返すことができ、それが標準になっているのが面白いと感じました。

多相・多値・Exceptionどれを使うかなどを迷う必要がなくて個人的に気に入りました。

まだ慣れていないからか、文字列の変数埋め込みは欲しいと感じてしまいます。

### クラス・構造体

```dart
// Dart

class Person {
  String name;
  int age;

  const Person(this.name, this.age);

  void greet() {
    print('Hello, $name!');
  }
}
```

```go
// Go

type Person struct {
	Name string
	Age  int
}

func (p Person) greet() {
	println("Hello, " + name)
}

func main() {
	p := Person{"riscait", 35}
	p.greet()
}
```

Dart (Flutter)でのアプリ開発には `class` を使用し、 Goでは `struct` を使用します。

Swiftでは `class` と `struct` を適切に使い分けることが求められたのを思い出しました。

Goではメソッドを定義する場合、構造体の中に書くのではなく、構造体の外に書き、レシーバとして構造体を受け取る形式なのがとても面白いと感じました。

## Web API実装の技術選定

Goを使ったWeb APIの構築は初めてということもあり、技術選定には苦労しました。
最終的に、現段階では以下のフレームワーク・ライブラリを採用しました。

### Echo: Web framework
https://echo.labstack.com

APIを作るためのフレームワークには、軽量かつ高性能と名前をよく[Echo](https://echo.labstack.com/)を使用しています。

安心感がありますね。

### bun: ORM, SQL Client
https://bun.uptrace.dev

SQLを使ったデータベース操作には、[bun](https://bun.uptrace.dev/)を使用しています。

比較対象として、 `GORM` , `ent` , `sqlx` `sqlc` などたくさんの候補がありました。

最終的には、軽量で `SQL-first` を謳う `bun` を採用しました。

### OpenAPI (oapi-codegen)

https://github.com/deepmap/oapi-codegen
https://pub.dev/packages/swagger_parser

秩序あるAPI開発のために、OpenAPIを採用しました。

また、 `Single Source of Truth(信頼できる唯一の情報源)を実現するために、
サーバーサイドとアプリサイドで共通のOpenAPI定義ファイルを使用してコードを生成することにしました。

Goのコードは[oapi-codegen](https://github.com/deepmap/oapi-codegen)を、
Dartのコードは[swagger_parser](https://pub.dev/packages/swagger_parser)を使用して生成しています。

## GoとDartのコード生成の流れ

### 1. 分割したOpenAPIの定義Yamlファイルを1ファイルにまとめる

まずOpenAPIに則ってYAMLファイルを作りますが、1ファイルにまとめて書くと非常に行数が多くなってしまうので、分割して書いています。

しかし、分割したYAMLファイルをそのままコードジェネレーションすると、Goのコードがうまく生成できませんでした。

そこで `@redocly/cli` を使用して、分割したYAMLファイルを1つにまとめます。

```shell
npx @redocly/cli bundle open_api/api.yaml -o open_api/api.gen.yaml
```

### 2. OpenAPIの定義Yamlファイルから、サーバーサイドのファイルを生成する

次に、統合したOpenAPIの定義Yamlファイルから、サーバサイドのGoファイル（api.gen.go）を生成します。

コード生成には[oapi-codegen](https://github.com/deepmap/oapi-codegen)を使用しました。

```shell
oapi-codegen -config {server_package}/oapi_codegen_config.yaml open_api/api.gen.yaml
```

### 3. OpenAPIの定義Yamlファイルから、アプリサイドのDartファイルを生成する

最後に、OpenAPIの定義Yamlファイルから、アプリサイドのDartファイル（api.gen.dart）を生成します。

最初は[openapi-generator](https://openapi-generator.tech/)を利用してDartコードを生成していましたが、
途中で[swagger_parser](https://pub.dev/packages/swagger_parser)に乗り換えました。

2つの方法を比較した `swagger_parser` のメリット・デメリットは以下の通りです。

- `swagger_parser` のメリット
  1. `Freezed` に対応している
- `swagger_parser` のデメリット
  1. Multipartリクエストに幾つかの問題がある（対処は可能）

また、 openapi-generatorは、Dartパッケージごと生成しますが、 swagger_parserは任意のDartパッケージの `dependencies` として記述して使用します。

コードを生成するには `dart run swagger_parser` を実行するだけですが、
`Freezed` を使用する場合は、 `build_runner` によるコード生成も行う必要があります。

```shell
cd {api_client_package} && dart run swagger_parser && dart run build_runner build -d
```

VS Codeのタスクとしてtasks.jsonに登録して呼び出せるようにしています。

```json
{
	"label": "Generate OpenAPI files for Go/Dart",
	"type": "shell",
	"group": {
		"kind": "build",
		"isDefault": false
	},
	"dependsOn": [
		"Bundle OpenAPI files",
		"Generate API Interface code from OpenAPI specifications",
		"Generate API Client code from OpenAPI specifications",
		"Generate API Client code with build_runner"
	],
	"dependsOrder": "sequence"
},
```

## サーバーサイド（Go）とアプリサイド（Flutter）のローカル開発フロー

ローカルでの現在の開発フローは以下の通りいくつかの下準備をしてからのアプリ起動が必要です。

### 1. Dockerコンテナの立ち上げ（compose.yaml）: tasks.json

データベースと pgAdmin の設定を記述した `compose.yaml` を使ってコンテナを立ち上げるタスクを `tasks.json` に登録してあります。

これは次のステップで使用する launch.json の `preLaunchTask` として指定してあるため、普段意識して実行することはありません。

### 2. Cloud Runのローカルデバッグ起動（Dockerfile）: launch.json

現在制作中のアプリでは、Google CloudのCloud Runを使用しています。

Cloud Runのローカルデバッグ起動を行うための `Dockerfile` を参照した起動設定を `launch.json` に登録してあります。

サーバーディレクトリを別ウィンドウで開いて `F5` を押すことで、サーバーサイドのローカルデバッグ環境が開始されます。

### 3. Firebase Emulatorの起動 (firebase.json): tasks.json

:::message
あらかじめ `firebase init` コマンドで使用したいエミュレータスイートを選択して構成ファイルを生成しておきます。
https://firebase.google.com/docs/emulator-suite?hl=ja
:::

Firebase Emulatorを起動するタスクを `tasks.json` に登録してあります。

コマンドパレット（⌘ + shift P）の「Tasks: Run Task」から `tasks.json` に登録したタスクを選択することで、Firebase Emulatorが起動します。

このタスクは `preLaunchTask` として次のステップのアプリ起動に指定すると、
エミュレータ起動で処理が止まってアプリ起動を行うことができなかったため、単体で実行するようにしています。

### 4. FlutterアプリをRun: launch.json

最後に、Flutterアプリを起動するための設定を `launch.json` に登録します。

サーバーと同様に、アプリディレクトリを別ウィンドウで開いて `F5` を押すことで、アプリのローカルデバッグ環境が開始されます。

これで、サーバーサイドとアプリサイド両方のローカルデバッグ環境が立ち上がります。

:::message
できれば1コマンドでサーバーとアプリの両方のローカルデバッグ環境を起動したいのですが、
現状は2つのウィンドウを開いてそれぞれのデバッグ環境を起動する必要があります🧐
何か良い方法があれば教えてください🙇‍♂️
:::

## 終わりに

まだ探り探りかつ学びながらの実装ではありますが、アプリをFlutter(Dart)で、サーバーサイドをGoで並行開発する方法に確かなメリットを感じています。

これから複数のアプリを作ってみて、より良い方法があれば改善もしていきたいと思います。

ご覧いただきありがとうございました。

## 宣伝

Altive株式会社では、Flutterアプリの開発・運営を承っております。
お気軽にお問い合わせください🫡 
https://altive.co.jp/contact

---

Riverpod の実践入門本を公開中です📘
https://zenn.dev/riscait/books/flutter-riverpod-practical-introduction

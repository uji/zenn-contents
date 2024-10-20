---
title: "Go 1.24 から go.mod でのツール管理がより簡潔になるかもしれない"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go"]
published: true
---

こんにちは、ujiです。
Goでプログラムを書く際、Goで書かれたコマンドラインツールを使用する機会が少なからずあると思いますが、 皆さんはGoの開発で利用するコマンドラインツールのバージョンはどのように管理していますか？

Go公式WikiのModuleのページでは、`tools.go` ファイルを用意して、blank importで利用するツール群の依存関係をgo.modの管理対象にする方法が紹介されています。
https://go.dev/wiki/Modules#how-can-i-track-tool-dependencies-for-a-module

```:go.mod
//go:build tools

package tool

import _ "golang.org/x/tools/cmd/goimports"
```

自分の場合は、Go 1.16でgo installでのバージョン指定ができるようになってからは、
シンプルに、Makefileに `go install` コマンドを書くことが多いです。
（こちらのアプローチの場合、Dependabot, Renovate などのpackage自動アップデートツールを使おうとすると設定が少し煩雑になります）

```:Makefile
.PHONY: tools
tools:
	go install golang.org/x/tools/cmd/goimports@v0.1.5
	go install honnef.co/go/tools/cmd/staticcheck@v0.3.2
```

このような開発ツールの依存関係の管理をさらに簡単に行えるようにする提案が承諾され、Go 1.24のアップデートに入る可能性があります。

## 提案の背景

既存のアプローチだと、依存関係の管理は優れている一方で、ツール自体の追加、削除などの管理が簡素化されていない、というのが提案者の考えです。

例えば、gqlgen（GraphQLのコード生成ツール）を使う新しいプロジェクトをセットアップする手順は次のようになります。
（例はissueで挙げられていたものをそのまま使っています）

```shell:shell
# Go Moduleの初期化
mkdir example
cd example
go mod init example

# ツールの追加
printf '// +build tools\npackage tools\nimport _ "github.com/99designs/gqlgen"' | gofmt > tools.go
go mod tidy

# gqlgenの初期化コマンドを実行
go run github.com/99designs/gqlgen init
```

ツールを追加するためにここでは printf コマンドが使用されていますが、これはGoのツールチェイン外での操作になります。
この追加方法には、

- Unix 系のシステムでしか機能せず、Windows では動作しない
- tools.go ファイルがすでに存在している場合のハンドリング

など、開発者の考慮するべき事項がいくつか存在し、あまり親しみやすい方法とは言えません。

ツールの追加や更新、削除もGoのツールチェインに含め、ツール管理の体験を向上させるのが、この提案の目的になります。

## 提案の概要

issueでは、go.mod にツール用のdirectiveを追加し、Goコマンドからそれを編集できるようにする仕組みが提案されています。

### go.mod の `tool` directiveの追加

ツールを管理するためのdirective `tool` を追加します。

```:go.mod
go 1.24

tool (
    golang.org/x/tools/cmd/stringer
    ./cmd/migrate
)
```

### go get, go mod editのオプション追加

先述のtool directiveを更新するためにgo get, go mod editに新しいオプションを追加します。
toolを追加、更新したい場合は、
`go get -tool path/to/package`
`go mod edit -tool path/to/package`

逆に削除したい場合は、
`go get -tool path/to/package@none`
`go mod edit -droptool path/to/package`

などを利用します。
`go mod tidy` では依存が削除されることはありません。

### go tool コマンドでツールを実行できるようにする

例えば、go.mod に

```:go.mod
tool golang.org/x/tools/cmd/stringer
require golang.org/x/tools v0.9.0
```

上記のように記述がある場合、`go tool stringer` を実行すると、`go run golang.org/x/tools/cmd/stringer@v0.9.0` と同様に動作します。

go run とは違い、go tool はビルドされたバイナリを `$GOCACHE/tool/<current-module-path>/<TOOLNAME>` にキャッシュします。

---

これらの変更によって、ツールの管理がGoのツールチェインで完結できるようになり、意図しないバージョンのツールが利用される危険性も減ります。
また、ツール専用のキャッシュロジックも設けることで、複数プロジェクトの開発を並行して進める際のディスク利用もより効率的になります。

詳細は以下のDesign docにまとまっています。

https://go.googlesource.com/proposal/+/54d6775ff71ccbc00c276db2a4e4841d67011cf4/design/48429-go-tool-modules.md

実装についても、プロトタイプが既に公開されています。

https://go-review.googlesource.com/c/go/+/613095

## 最後に

Proposalは去年の8月時点で既にAcceptedになっています。
↓Acceptedのコメント

https://github.com/golang/go/issues/48429#issuecomment-1699638321

が、今回紹介した内容はあくまでも 2024-10-19 時点のもので、今後変更になる可能性があります。
スケジュールについてもまだ不透明な状況です。
（Design docには1.24で取り組むと記載がありますが、Go 1.24のマイルストーンやリリースノートのドラフトには入っていません）
https://github.com/golang/go/milestone/322
https://tip.golang.org/doc/go1.24

最新の情報をチェックしたい方はissueでの議論をウォッチして置けると良さそうです。

https://github.com/golang/go/issues/48429

この記事の内容についてご意見や改善点がありましたら、ぜひフィードバックをお寄せください。
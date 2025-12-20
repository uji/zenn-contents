---
title: "Go の test における flag パッケージ活用Tips"
emoji: "⛳️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "test"]
published: true
published_at: 2025-12-21 00:00
---

:::message
この記事は、[Go Advent Calendar 2025](https://qiita.com/advent-calendar/2025/go) の 21 日目の記事です。
:::

先日、[Findy さんのイベント](https://findy.connpass.com/event/375530/)にて testdata ディレクトリの活用についての発表をさせていただきました。

https://x.com/uji_rb/status/1996066089432969331

このイベントの中で、Golden Test の実現に標準の flag パッケージを活用できる話をしました。
testdata 利用の有無に限らずこのTipsは便利なので、知見の共有がしやすくなるように記事にしようと思います。

## 利用方法

使い方は3ステップです：

```go
// 1. パッケージをインポート
import "flag"

// 2. グローバル変数としてフラグを定義
var example = flag.Bool("example", false, "example flag.")

// 3. テスト内で値を参照（*exampleでポインタから値を取得）
func TestExample(t *testing.T) {
    if *example {
        // フラグ有効時の処理
    }
}
```

あとは `go test -update` と実行するだけ。
go test が内部で `flag.Parse()` を呼んでくれるので、追加のコードは不要です。

https://github.com/golang/go/blob/3f94f3d4b2f03a913de3f5a737bad793418e751f/src/testing/testing.go#L2351-L2354

[golang/go](https://github.com/golang/go) のコードでもこのTipsは広く使われているので、実際のコードと一緒に活用例を紹介します。

## Golden Test での活用 

先日の発表とも重複する内容ですが、Golden Testはよくある活用パターンです。
Golden Test はテスト対象の出力結果を事前に保存された "Golden" ファイルと比較するテストで、コードを変更した際、意図しない変更や回帰がないかを検証するために特に有用な手法になります。

以下のコードは[src/go/doc/doc_test.go](https://github.com/golang/go/blob/3f94f3d4b2f03a913de3f5a737bad793418e751f/src/go/doc/doc_test.go#L24) の例です。
`flag.Bool` で `update` フラグを定義し、有効な場合のみ testdata ディレクトリにあるGoldenファイルを更新するようにしています。

```go
var update = flag.Bool("update", false, "update golden (.out) files")
...
func test(t *testing.T, mode Mode) {
    ...
    for _, pkg := range pkgs {
        t.Run(pkg.Name, func(t *testing.T) {
            ...
            // Golden ファイルのupdate
            golden := filepath.Join(dataDir, fmt.Sprintf("%s.%d.golden", pkg.Name, mode))
            if *update {
                err := os.WriteFile(golden, got, 0644)
                if err != nil {
                    t.Fatal(err)
                }
            }
            // Golden ファイルの取得
            want, err := os.ReadFile(golden)
            if err != nil {
                t.Fatal(err)
            }
            // 検証
            ...
```

README ファイルを Golden ファイルとして自動生成させている場面もあったりします。

https://github.com/golang/go/blob/70c22e0ad7d89504ab26fb157864f61a79cd4d47/src/cmd/compile/script_test.go#L18

↑各種CLIのテストのために独自に実装されているスクリプト言語の README が扱われています。
README の更新漏れを検知できるのは良いですね。

## デバッグ用途での利用

デバッグ時のサポートを受けるためのフラグは各所で定義されています。
例えば src/cmd/compile/internal/ssa/debug_test.go では、デバッガの出力を詳細にプリントしたり、dryrun 実行を行ったり、Delve の代わりに GDB を使ったりなど様々です。

https://github.com/golang/go/blob/70c22e0ad7d89504ab26fb157864f61a79cd4d47/src/cmd/compile/internal/ssa/debug_test.go#L24-L30

ファイルシステムのテストでは、tmp ディレクトリに生成されたファイルを消さずに残すためのフラグなどもあり、デバッグに重宝しそうです。

https://github.com/golang/go/blob/70c22e0ad7d89504ab26fb157864f61a79cd4d47/src/cmd/go/go_test.go#L824-L844

## テストの冗長性の制御

Go の testing パッケージでは `-short` フラグの解析がデフォルトで備わっており、`testing.Short()` を使って条件分岐を書くことで、長時間かかるテストは定期的なテストではスキップするなどの制御が可能です。

更に時間をかけたテストも実行、など冗長性のパターンを増やしたい場面で flag パッケージが使われることがあります。

https://github.com/golang/go/blob/70c22e0ad7d89504ab26fb157864f61a79cd4d47/src/crypto/mlkem/mlkem_test.go#L162-L176

## feature flag としての利用

Go 1.26 から SIMD のサポートが実験的に始まる予定です。

↓ Proposal

https://github.com/golang/go/issues/73787

それに伴い、コンパイラ周辺のテストでも SIMD を考慮したコードが書かれているのですが、SIMD の有効フラグを使って処理を切り替えており、テストで利用する feature flag としての運用も見られます。

https://github.com/golang/go/blob/70c22e0ad7d89504ab26fb157864f61a79cd4d47/src/cmd/compile/internal/ssagen/intrinsics_test.go#L19-L20

## シンプルにスクリプトの実行に利用

src/unicode/letter_test.go では、線形探索と二分探索のカットオフポイントを探すスクリプトが実装されており、`go test -calibrate` で実行できるようにされていました。

https://github.com/golang/go/blob/70c22e0ad7d89504ab26fb157864f61a79cd4d47/src/unicode/letter_test.go#L442C1-L458

テストやデバッグ用途でのみ利用するスクリプトはこのように実装すると、意図しない用途での利用を防げたり、go build のビルド対象から除外できるなどの点で良さそうです。

## まとめ

今回紹介した例は、[golang/go](https://github.com/golang/go) のテストコードでの flag パッケージ利用箇所は以下から一覧して探しました。
(GitHub のコード検索クエリ `repo:golang/go "= flag." path:/src/**/*_test.go` で絞っています)

https://github.com/search?q=repo%3Agolang%2Fgo+%22%3D+flag.%22+path%3A%2Fsrc%2F**%2F*_test.go&type=code

flag パッケージで定義したフラグは CI の継続的なテストで常用するというよりは、開発者が手元でテストや周辺コードを調整・検証する際に便利な使い方が多いです。
テストの柔軟性や効率を高めたいときの手札として、紹介したTipsを覚えてもらえると嬉しいです。
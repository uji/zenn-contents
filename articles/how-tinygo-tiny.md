---
title: 'TinyGoはどのように "Tiny" を実現している？'
emoji: "🐉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Go", "tinygo", "llvm"]
published: true
published_at: 2025-10-12 15:00
---

:::message
この資料は TinyGo Conference 2025 in Japan の登壇資料です。
:::

## 自己紹介

- 名前: [uji](https://x.com/uji_rb)
- 神戸市在住
- NOT A HOTEL のソフトウェアエンジニア
- Gopher 7年生、TinyGo は 3年くらい前に初めて触りました
- KOBE.go, Kyoto.go 運営

![](https://storage.googleapis.com/zenn-user-upload/e31979c59c42-20241201.png)
*KOBE.go TinyGo Keebイベントの様子*

- 作ったもの

https://x.com/uji_rb/status/1964242879175327748

https://x.com/uji_rb/status/1975770765812527488

https://zenn.dev/uji/articles/wio-terminal-tinygo

## 得られること

- TinyGo がどのように "Tiny" しているのかをざっくりと理解できる
- TinyGo が生まれた背景、設計思想に触れ、GoやTinyGoのコンパイラやランタイムの仕組みに興味が持てる！

## そもそも TinyGo とは？

TinyGoは、Go言語をマイコン（マイクロコントローラ）やWebAssembly（WASM）のようなリソースが限られた小規模な環境で実行するために設計された、Go言語の代替コンパイラです。
TinyGoという言語がGoとは別にあるわけではありません。

![](https://storage.googleapis.com/zenn-user-upload/a9d0b5464587-20251008.png)

なので、TinyGoでのプログラミングでも、Go言語の記法さえ理解していればほぼ同じ感覚でコードを書くことができます。

サポートされている言語機能一覧はこちら
https://tinygo.org/docs/reference/lang-support/

例えば、最初に紹介した TinyGo Keeb のキーボードでは、
- 色を変化させる処理
- 光る場所を変化させる処理

この2つを goroutine で並行に動かしています。組み込みプログラミングで goroutine で簡単に並行処理を書けるのは感動ものです。
https://x.com/uji_rb/status/1964242879175327748

## Go コンパイラだとダメなの？

Goコンパイラで生成した実行ファイルは、ランタイム（プログラム実行時に必要な部品や環境）への依存が強く、サイズがそれなりに大きくなります。
これは、開発者が本質的なプログラミングに集中できるよう、様々な機能をリッチに備えているためです。
CPUコアやメモリなどのリソースが豊富にある環境での利用が前提となっている節があります。

マイコンプログラミングはその辺り、シビアに見る必要があります。
例えば、冒頭で紹介した TinyGo Keeb のキーボードでは `RP2040-Zero` というマイコンが使われているのですが、
搭載されているRAM(SRAM)が264KB、オンボードフラッシュメモリが2MBというスペックで、Goコンパイラで生成した実行ファイルを扱うには厳しさがあります。
<!-- フラッシュメモリ、から直接コードを実行します。また、定数グローバル変数は可能な限りフラッシュメモリに格納されるのが一般的です。 -->

https://www.waveshare.com/rp2040-zero.htm

### Hello Worldコードの実行ファイルサイズ比較

```go
package main

import "fmt"

func main() {
	fmt.Println("Hello, 世界")
}
```

Playground でも使われているHello Worldコードの実行ファイルを、最もシンプルなビルドコマンドで比較してみました。
(Apple M4, Go 1.25.2, TinyGo 1.39.0)

```shell
go build -o gobin .
tinygo build -o tinygobin .
ls -l
```

```
total 5288
-rw-r--r--@ 1 uji  staff       35 Oct  8 13:22 go.mod
-rwxr-xr-x@ 1 uji  staff  2387858 Oct  8 13:28 gobin
-rw-r--r--@ 1 uji  staff       74 Oct  8 13:23 main.go
-rwxr-xr-x@ 1 uji  staff   308016 Oct  8 13:28 tinygobin
```

- **go: 約2.4 MB**
- **tinygo: 約308 KB**

と、かなり差分が出ています。
マイコンでプログラムを動作させる際、フラッシュメモリにプログラムを書き込む必要があるので、
Hello World のプログラムですら、Goでコンパイルすると先ほど紹介した `RP2040-Zero` には載らないサイズになってしまいます。 

実行サイズ以外にも、マイコン固有のCPU命令セットや、OSのないベアメタル環境のサポートが不十分、といった課題もあったりします。

参考

https://tinygo.org/docs/concepts/faq/why-a-new-compiler/

公式ドキュメントには Rust を使うのではなく Go でコンパイラを作る理由を述べたページもあるので興味ある方は読んでみてください。

https://tinygo.org/docs/concepts/faq/why-go-instead-of-rust/

## 実行ファイルやメモリ消費量を小さくするための工夫

TinyGoでは、Goコンパイラで重要視されていたパフォーマンスをある程度犠牲にする一方で、実行ファイルやメモリ消費量削減に最適化したコンパイルを実現しています。
Goコンパイラとはどのような差異があるのか見てみましょう。

<!-- ここまで9分 -->

### コンパイルフロー

Go コンパイラのコンパイルフローは以下のようになっています。

```
1. 字句解析（文字列をトークンと呼ばれる最小単位のテキストに分割）
    ↓
2. 構文解析（トークンを抽象構文木に変換）
    ↓
3. 意味解析（型検査や名前解決を実行）
    ↓
4. SSA (静的単一代入形式 と呼ばれる中間表現) 変換
    ↓
5. ミドルエンド最適化（インライン展開・エスケープ最適化など）
```

Go ではこれら全てを独自に実装してコンパイラを実現しています。


一方、TinyGoのコンパイルは以下のような流れで行われます。

```
1. 字句解析
    ↓
2. 構文解析
    ↓
3. 意味解析
    ↓
4. SSA 変換
    ↓
5. LLVM IR (Intermediate Representation) への変換
    ↓
6. LLVM 最適化 Pass
```

SSAの変換まではGo公式で用意されているエコシステムを活用して実現しています。
（標準ライブラリの `go/parser` や `go/types`、`golang.org/x/tools/go/ssa` など）
最も重要な違いは、**LLVM**をバックエンドとして利用している点です。
SSA 変換後は LLVM のエコシステムを利用して実行ファイルの生成まで行っています。

#### LLVMを採用した理由

![](https://llvm.org/img/LLVMWyvernSmall.png)
*LLVMのロゴ。[公式](https://llvm.org/Logo.html)より引用*

LLVMは、コンパイラの機能を提供するツールチェーン、汎用コンパイラ基盤です。

https://github.com/llvm/llvm-project

https://aosabook.org/en/v1/llvm.html


特に以下の点でTinyGoの要件にマッチしています。

1. **多様なターゲット対応**: LLVMは多くのアーキテクチャ（ARM、RISC-V、WebAssemblyなど）をサポートしており、マイクロコントローラからWebAssemblyまで幅広いターゲットに対応できます。
2. **高度な最適化**: LLVM Passと呼ばれる LLVM IR に対する解析・最適化の機能を提供するツール群が非常に優秀で、デッドコード除去やインライン化などのあらゆる最適化ツールにより、実行ファイルサイズを大幅に削減できます。（[一覧](https://llvm.org/docs/Passes.html)）
3. **段階的な機能追加**: LLVM のツールはモジュールが細かく分かれており、必要最低限の機能を含めることができ、必要に応じて段階的に機能を追加できる。

https://tinygo.org/docs/concepts/compiler-internals/pipeline/

LLVMは汎用コンパイラ基盤として広く普及しており、実績があります。実は clang や rustc もLLVMへの依存があります。
この辺りを、独自に実装して実現するのは大変です。他言語でも実績を上げている最適化ツールを再利用することで、エコシステムの恩恵を受けられるのはとても強力です。

LLVM を使って実現されたコンパイルの短所の一つとしてコンパイル速度がありますが、TinyGo が想定するユースケースはコンパイルコードがそこまで大きくなるものではないので、問題になりづらいと思います。そういう意味でもLLVMの選択は理にかなっていそうです。
（zig でもバックエンドとして採用していましたが、コンパイル速度のボトルネックを解消するため独自コンパイラへの置き換えが完了しつつあります。）

https://github.com/ziglang/zig/issues/16270

https://mitchellh.com/writing/zig-builds-getting-faster


実は、今のTinyGoの前に Go の別の代替コンパイラ gccgo に依存してコンパイラを実現するプロジェクトもあったりしました。（最適化の余地があまりなく廃止となったそうです）

https://github.com/aykevl/tinygo-gccgo

### ランタイム機能の簡略化

Go特有のランタイム機能としてgoroutineやガベージコレクション、インターフェースなどがありますが、この辺りの実装も必要最低限なものにすることで、実行ファイルサイズを削減しています。

#### GC（ガベージコレクション）

メモリ上のごみを見つけて回収し、プログラムが再利用できるようにしてくれる仕組みです。
Go では開発者を本質的なプログラミングに集中させるため、ガベージコレクションを採用し、シンプルながらも大規模環境にも耐えうるものを実現しています。

最近大幅なアップデートがあったのですが、Nagamiさんの資料がとてもわかりやすかったです

https://x.com/uji_rb/status/1975206371999219964

TinyGo では、従来の Go ランタイムよりも古典的でシンプルなGCアルゴリズムがデフォルトで利用されます。（Mark Sweep 方式の保守的GC）

![](https://storage.googleapis.com/zenn-user-upload/b097e65c460e-20251011.gif)
*Mark Sweep GC のイメージ*

また、TinyGoはターゲットとするマイコンが豊富でそれぞれメモリ容量は幅があるため、いくつかのGCアルゴリズムをビルド時に選択できるようになっています。

- Conservative GC (デフォルト)
- Precise GC
- Leaking GC
- No GC
- Custom GC
- Boehm GC

`src/runtime/gc_*.go` で各種実装が見れるので興味あれば追ってみてください。

#### goroutine

Goの最も特徴的な機能の一つであるgoroutineですが、こちらもTinyGoでは大幅に簡略化されたスケジューラーで制御されています。

通常のGoのgoroutineは、マルチコアCPU環境で効率的にリソースを活用した並列・並行処理が実現できるように色々複雑な処理が実装されています。

> 例
> - **M:N スケジューラー**: 多数のgoroutine（G）を少数のOSスレッド（M）上で実行し、プロセッサ（P）を割り当てる仕組み
> - **ワークスティーリング**: 暇を持て余したスレッドが他のスレッドからタスクを「盗む」ことで負荷分散する仕組み
> - **プリエンプション**: 実行中のgoroutineを強制的に中断して他の優先度の高いgoroutineに切り替える仕組み

Go の goroutine 実現の仕組みは、さきさんの資料・発表がわかりやすいです。

https://zenn.dev/hsaki/books/golang-concurrency

https://gocon.jp/2021autumn/sessions/go-scheduling/

公式のドキュメントページもあります。

https://go.dev/src/runtime/HACKING

一方TinyGoは、マルチコア・マルチスレッドの環境でのパフォーマンスは優先事項にならないので、シンプルに**すべてのgoroutineが1つのスレッド上で順次実行される**ようになっています。（WASI 文脈では並列処理サポートが検討されているらしい）
Go ランタイムにあったようなリッチな最適化の機能はなく、goroutineが自発的に制御を手放すまで実行を継続（協調マルチタスク）するようになっています。

これにより、スケジューラー自体が占有するメモリは大幅に減ります。

#### インターフェース

通常のGoコンパイラでは、インターフェースメソッドの高速な呼び出しを実現するために、関数ポインタのリスト（method table）をコンパイル時に計算して実行ファイルに埋め込まれます。これにより、メソッド呼び出し時の探索コストを削減できます。

しかし、TinyGoでは事前計算せず、メソッド呼び出しが実際に使用される場所で動的に解決するアプローチをとっています。
これも実行速度よりもメモリ効率を重視するためのアプローチになります。

### グローバル変数の扱い

従来のGoはコード実行時に多くの初期化を行い、その中にはすべてのグローバル変数も含まれます。
TinyGoは これらの関数を可能な限りコンパイル時に解釈します。
こうすることでグローバル変数はフラッシュメモリ上に配置され、RAMを使わずにすみます。

## まとめ

色々と紹介してきましたが、TinyGoが「Tiny」を実現している仕組みは、以下のような技術的工夫によるものでした。

1. **LLVMバックエンドの活用**: 高度な最適化により実行ファイルサイズを大幅削減
2. **ランタイム機能の簡略化**: GC、goroutineスケジューラー、インターフェースの実装を最小限に
3. **コンパイル時最適化**: グローバル変数の事前計算などによりRAM使用量を削減

これらにより、マイコンのような制約環境でもGoの表現力を活用したプログラミングが可能になっています。
今回は重要な部分に絞って紹介しましたが、他にも「Tiny」に寄与する仕組みはあるので、興味ある方は是非ドキュメントやコードを見てみてください！
Heap Allocation の最適化の話なんかも面白いです！

https://tinygo.org/docs/concepts/compiler-internals/heap-allocation/

## 最後に

12月に神戸で Go Workshop Conference が開催されます！
Go/TinyGo Conference のワークショップが楽しかった方・参加できなかった方は是非来てください！

やさしい内容のものもありますので、Go初学者の方や書いたことがない方も楽しめる内容になるのではと思います（神戸のグルメ情報まとめてお待ちしております！）

https://gwc.gocon.jp/2025/

ワークショップ・ブースコンテンツの公募もやってますので是非…！

https://forms.gle/ruWwUXARbq3d2zfHA

https://forms.gle/vLawxpzAjiJUFiUQ6
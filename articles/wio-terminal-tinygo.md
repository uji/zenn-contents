---
title: "Wio Terminal+TinyGoでネームバッジを作ってみる"
emoji: "🎖️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["tinygo"]
published: true
published_at: 2024-12-03 00:00
---

この記事はTinyGo Advent Calendar 2024 3日目の記事です。

https://qiita.com/advent-calendar/2024/tinygo

TinyGo を使って Wio Terminal で動くネームバッジを作りました。
↓こんなやつです。

https://x.com/uji_rb/status/1817463672807313639

カンファレンスなどで首から下げて使えると便利かなと思ってます。（まだ使えてない）

## Wio Terminal とは？

Seeed 社が出しているのディスプレイ付きの開発ボードになります。
コンパクトな筐体に各種インターフェースが揃っており、マイコンプログラミングの遊びやプロトタイピングに適した商品になってます。

https://www.switch-science.com/products/6360

TinyGoのサポートもされています。
[sagoさん](https://x.com/sago35tk) が執筆されている技術書「基礎から学ぶ TinyGoの組込み開発」の題材にもなっているので、TinyGoでのマイコンプログラミングの入門にもバッチリなボードではないでしょうか。

https://sago35.hatenablog.com/entry/2022/11/04/230919

## コード

リポジトリはこちら

https://github.com/uji/ujiwio

と言っても、ほぼsagoさんが用意してくれているサンプルコードの写経です。

https://github.com/sago35/tinygo-examples/tree/cae8d6e2d136f6ceea970ca1b23c57f64a079156/wioterminal/wiobadge

Go標準にもあるembedが使えるので画像ファイルの扱いはとても簡単でした。すごい。
```go
//go:embed name.png
var name_png []byte

//go:embed qrcode_x.png
var qrcode_x_png []byte
```

他にもあらゆるコードが標準のGoと同じ書き味なので、シンタックス周りはあまり苦戦することなく実装ができて感動です。

sagoさんの作品例はWio Terminalのものはもちろん、他のボードのものも色々ありますので、TinyGoでマイコンプログラミングに挑戦される際は是非参考に見てください。

https://github.com/sago35/tinygo-examples

## 今後やりたいこと

イベントなどで使えるように、首から下げられるようにしたいですが、バッテリーの確保とストラップとの接続をどうしようか考え中です。

バッテリーはWio Terminal専用のGPIOにつなぐタイプのものや、

https://www.switch-science.com/products/6816

Grove端子で使える乾電池タイプのユニットがあるのでどちらかを買おうかなと考えています。

https://www.switch-science.com/products/7638

ストラップとの接続ですが、紐を通せるような機構が無さそうで、あんまり良い案が思いついておらずです。
良いやり方あれば教えて下さい🙏

---

あとはバッジの機能も何かおもしろいものを増やしたいですね。
[koebiten](https://github.com/sago35/koebiten)(TinyGoで使える[Ebitengine](https://ebitengine.org)) で作ったゲームが動いたりしたら面白そうです。

https://x.com/sago35tk/status/1858453817357980069

## 最後に

TinyGoへの関心や興味は、[TinyGo Keeb Tour](https://tinygo-keeb.connpass.com/) というTinyGoで自作キーボードを作るハンズオンイベントがきっかけで強くなりました。

https://tinygo-keeb.connpass.com/

自分が運営している[KOBE.go](https://kobego.connpass.com)でもsagoさんに講師をしていただきハンズオンイベントを開催したのですが、初心者の方でも楽しみながらTinyGoのマイコンプログラミングに入門してもらえる内容になりました。
sagoさんはじめ、TinyGo Keeb Tourを支えてくださっている皆さんにBig感謝です

![](https://storage.googleapis.com/zenn-user-upload/e31979c59c42-20241201.png)
*イベントの様子*

今年は神戸、東京、仙台、福岡の4都市で開催されましたが、来年も全国各地で開催されるので、もし興味のある方は参加してみることをおすすめします！

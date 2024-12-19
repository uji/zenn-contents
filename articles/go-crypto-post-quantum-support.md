---
title: "Goのポスト量子暗号（耐量子計算機暗号）サポート方針"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Go", "security", "暗号", "耐量子計算機暗号"]
published: true
---

この記事はGo Advent Calendar 2024 20日目の記事です。

https://qiita.com/advent-calendar/2024/go

来年2月にリリース予定のGo 1.24で、新しい crypto パッケージ [crypto/mlkem](https://pkg.go.dev/crypto/mlkem@master) がサポートされる予定です。

https://tip.golang.org/doc/go1.24#crypto-mlkem

cryptoに新しいパッケージが追加されるのはGo 1.20の [crypto/ecdh](https://go.dev/doc/go1.20#crypto_ecdh) 以来になります。([proposal](https://github.com/golang/go/issues/52221))

crypto/mlkem の提案の背景を追っていくと、Goのポスト量子暗号（耐量子計算機暗号とも呼ばれる）サポートのロードマップを見つけたので、その内容を紹介します。

https://github.com/golang/go/issues/64537

## 概要

- ポスト量子暗号は量子コンピュータによる攻撃に対しても安全性を保つことを目的とした暗号アルゴリズムの総称
- 暗号業界全体でポスト量子暗号への移行が推進されており、Go のセキュリティチームは、できるだけ早くポスト量子鍵交換をGo標準で実現することが重要だと考えている
- ML-KEM (FIPS 203) は、最近標準化され、業界全体に導入されており、Go 1.23 で[#67061](https://github.com/golang/go/issues/67061) の一部としてすでに実装されている
- Go 1.24から、crypto/mlkem として公開パッケージでもML-KEMがサポートされるようになる

## ポスト量子暗号とは

ポスト量子暗号（PQC: Post-Quantum Cryptography）とは、量子コンピュータによる攻撃に対しても安全性を保つことを目的とした暗号アルゴリズムの総称です。（量子コンピュータ時代の後の暗号技術ということで「ポスト」がつく）

現在広く使用されている暗号、特に安全性が計算困難性に依存している公開鍵暗号方式（例えばRSAや楕円曲線暗号）は、量子コンピュータの登場により解読されるリスクがあります。
2021年の研究によれば、RSA暗号の2048ビット整数は2000万量子ビットの量子コンピュータならおよそ8時間で解読可能だとのことです。([ref](https://www.seleqt.net/gadget/quantum-computer-break-2048-bit-encryption-8-hours))
量子コンピュータがこれらの暗号を解読できるようになるまで実用化するのは、予測では最短で2030年と言われていますが、2030年までに耐量子計算機対策をすれば良い、というわけではありません。

量子コンピュータによる脅威には、**HNDL攻撃(Harvest Now, Decrypt Later)を考慮し、量子コンピューティングの実用化より十分前に対策しておく必要があります。**
これは、既存の暗号技術が安全とされている間に暗号化されたデータを収集しておき、将来的に量子コンピュータなどの進歩した技術でデータを解読しようとする攻撃手法です。

この脅威に対処するため、業界全体でなるべく早くポスト量子暗号へ移行することが推奨されています。

https://developers-jp.googleblog.com/2024/09/post-quantum-cryptography-standards.html

https://qiita.com/yoshidaintellilink/items/e3fa5e54c5832c9eaab4

https://wolfssl.jp/wolfblog/2024/07/25/dilithium-vs-falcon/

https://www.redhat.com/ja/blog/post-quantum-cryptography-introduction

## Goのポスト量子暗号の対応方針

アメリカ国立標準技術研究所（NIST）での標準化や、業界全体でのポスト量子暗号への移行の始まりに伴い、Goのポスト量子暗号の対応は、こちらのissueで議論が開始されています。

https://github.com/golang/go/issues/64537

基本的に、Goはサポートする暗号化アルゴリズムの選択については非常に保守的です。
これは、メンテナンスの負担を小さく維持するためですが、安定した（将来的に大幅に変更される可能性が低い）アルゴリズムや、すでに幅広いエコシステムで積極的に使用されているアルゴリズムのサポートに集中するためでもあります。
アルゴリズムは数多くありますが、まだ実験段階であったり、業界で広く採用されていないなど、標準ライブラリに含めるための基準を満たしていないものについては、サードパーティのパッケージとして実装される方が理にかなっている、というのがGoチームの考えです。

どのポスト量子暗号アルゴリズムをサポートしたいかを考えるとき、最初のハードルはそのアルゴリズムが実際に標準ライブラリ自体で使用されるかどうかで、少なくとも最初は、自分たちが使用しないアルゴリズムのサポートを導入したくないということも言及されています。

### 対応優先度

HNDL攻撃(issueでは `collect-now-decrypt-later` と呼ばれています) を防ぐために、量子鍵交換の移行が最優先事項として考えられています。
逆に、署名などのHNDL攻撃に対して脆弱ではないアルゴリズムについては優先順位が低くする事も書かれています([ref](https://github.com/golang/go/issues/64537#issuecomment-2445056004))

### 最初のアクション

実はGo 1.23で既にポスト量子暗号への移行は適用され始めています。
ML-KEM と呼ばれる、最近標準化され業界全体で導入が進んでいるアルゴリズムが、実験的にcrypto/tlsに組み込まれています。

https://github.com/golang/go/issues/67061

従来の楕円曲線Diffie-Hellman鍵交換（X25519）と、耐量子計算機暗号であるML-KEM-768を組み合わせたハイブリッド鍵交換メカニズムを `X25519Kyber768Draft00` として実験的に実装され、デフォルトで有効になりました。（GODEBUG環境変数で元に戻すことが可能）

### ML-KEM とは

ML-KEM (Module-Lattice Key Encapsulation Mechanism) はNISTが2024年8月11日に標準化したポスト量子暗号アルゴリズムです。
商務長官によって連邦情報処理基準(FIPS)の承認を受け「FIPS 203」の番号が与えられています。

https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.203.pdf

詳細はここでは省略しますが、数学的に難解な「格子問題」を基盤としたアルゴリズムにすることで、量子コンピュータによる攻撃に対しても高い耐性を持つアルゴリズムになっています。

### Go 1.24 でのアップデート

先述の crypto/tls に実験的に導入されていたML-KEMが永続的なものになる予定です。

https://github.com/golang/go/issues/69985

また、冒頭で話したように、crypto/mlkem として公開パッケージでもML-KEMがサポートされるようになります。

https://github.com/golang/go/issues/70122

コードはこちら

https://pkg.go.dev/crypto/mlkem@master

今回のリリースでは、ML-KEM-768とML-KEM-1024の実装の提供が予定されています。
この2つは

- crypto/tlsなど、Go内部で利用する場面があること
- 商用国家安全保障アルゴリズムスイート 2.0（CNSA 2.0）では、ML-KEM-1024の使用が求められていること

以上から実装が重要視されています。

## 最後に

今後も継続的にポスト量子暗号への移行は進められていきそうですが、議論を見ると、
- 署名はHNDL攻撃に対して脆弱ではないためSLH-DSAやML-DSA（署名のポスト量子暗号アルゴリズム）の対応は優先度を下げる
- ML-KEM-512については、現時点での導入例がないため、少なくとも当面は除外する

など、「本当に今対応が必要なものかどうか」に対する言及が多く、その慎重さにGoらしさを感じました。

今回の記事ではポスト量子暗号のサポートについて紹介しましたが、
来月1/18(土)に福岡で開催される[Gopher's Gathering](https://connpass.com/event/329963/)というイベントで、Go標準の暗号ライブラリ全体のメンテナンス戦略についてお話する予定なので、もし興味あれば是非話を聞きに来てください！

https://connpass.com/event/329963/

## 参考文献

https://words.filippo.io/dispatches/mlkem768/

https://zenn.dev/anko/articles/ctf-crypto-lattice

https://zenn.dev/ankoma/articles/f165d06efb1468
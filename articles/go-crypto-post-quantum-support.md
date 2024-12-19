---
title: "Goのポスト量子暗号（耐量子コンピュータ暗号）サポート方針"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Go", "security", "暗号", "耐量子コンピュータ暗号"]
published: false
---

この記事はGo Advent Calendar 2024 20日目の記事です。

https://qiita.com/advent-calendar/2024/go

来年2月にリリース予定のGo 1.24で、新しい crypto パッケージ [crypto/mlkem](https://pkg.go.dev/crypto/mlkem@master) がサポートされる予定です。

https://tip.golang.org/doc/go1.24#crypto-mlkem

cryptoに新しいパッケージが追加されるのはGo 1.20の [crypto/ecdh](https://go.dev/doc/go1.20#crypto_ecdh) 以来になります。([proposal](https://github.com/golang/go/issues/52221))

crypto/mlkem の提案の背景を追っていくと、Goのポスト量子暗号（耐量子コンピュータ暗号とも呼ばれる）サポートのロードマップを見つけたので、その内容を紹介します。

https://github.com/golang/go/issues/64537

## 概要

- ポスト量子暗号は量子コンピュータによる攻撃に対しても安全性を保つことを目的とした暗号アルゴリズムの総称
- 暗号業界全体でポスト量子暗号への移行が推進されており、Go のセキュリティチームは、できるだけ早くポスト量子鍵交換をGo標準で実現することが重要だと考えている
- ML-KEM (FIPS 203、旧称 Kyber) は、最近標準化され、業界全体に導入されており、Go 1.23 で[#67061](https://github.com/golang/go/issues/67061) の一部としてすでに実装されている
- Go 1.24から、ポスト量子暗号がサポートされ始める。新しい crypto パッケージ crypto/mlkem としてがサポートされる

## ポスト量子暗号とは

ポスト量子暗号（PQC: Post-Quantum Cryptography）とは、量子コンピュータによる攻撃に対しても安全性を保つことを目的とした暗号アルゴリズムの総称です。

現在広く使用されている暗号、特に安全性が計算困難性に依存している公開鍵暗号方式（例えばRSAや楕円曲線暗号）は、量子コンピュータの登場により解読されるリスクがあります。
量子コンピュータがこれらの暗号を解読できるようになるまで実用化するのは、予測では最短で2030年と言われていますが、2030年までに耐量子計算機対策をすれば良い、というわけではありません。
量子コンピュータによる脅威には、**HNDL攻撃(Harvest Now, Decrypt Later)を考慮し、量子コンピューティングの実用化より十分前に対策しておく必要があります。**
これは、既存の暗号技術が安全とされている間に暗号化されたデータを収集し、将来的に量子コンピュータなどの進歩した技術でデータを解読しようとする攻撃手法です。

この脅威に対処するため、業界全体でなるべく早くポスト量子暗号へ移行することが推奨されています。

### 参考になる文献

https://developers-jp.googleblog.com/2024/09/post-quantum-cryptography-standards.html

https://qiita.com/yoshidaintellilink/items/e3fa5e54c5832c9eaab4

https://wolfssl.jp/wolfblog/2024/07/25/dilithium-vs-falcon/

https://www.redhat.com/ja/blog/post-quantum-cryptography-introduction

## Goのポスト量子暗号の対応方針

NISTでの標準化や、業界全体でのポスト量子暗号への移行の始まりに伴い、Goのポスト量子暗号の対応は、こちらのissueでロードマップが議論されています。

https://github.com/golang/go/issues/64537

### 対応優先度

HNDL攻撃(issueでは `collect-now-decrypt-later` と呼ばれています) を防ぐために、量子鍵交換の移行が最優先事項として考えられています。
逆に、署名などのHNDL攻撃に対して脆弱ではないアルゴリズムについては優先順位が低くする事も書かれています([ref](https://github.com/golang/go/issues/64537#issuecomment-2445056004))

https://github.com/golang/go/issues/64537#issuecomment-1921211107

### 最初のアクション

実はGo 1.23で既にポスト量子暗号への移行は適用され始めています。
ML-KEM (FIPS 203、旧称 Kyber) と呼ばれる、最近標準化され業界全体で導入が進んでいるアルゴリズムが、実験的にcrypto/tlsに組み込まれています。

https://github.com/golang/go/issues/67061

実験的なポスト量子鍵交換メカニズム `X25519Kyber768Draft00` が実装され、デフォルトで有効になりました。（GODEBUG環境変数で元に戻すことが可能）

### Go 1.24 でのアップデート

先述の crypto/tls に実験的に導入されていたML-KEMが永続的なものになる予定です。

https://github.com/golang/go/issues/69985

また、crypto/mlkem として公開パッケージでもML-KEMがサポートされるようになります。

https://github.com/golang/go/issues/70122

コードはこちら

https://pkg.go.dev/crypto/mlkem@master

今回のリリースでは、ML-KEM-768とML-KEM-1024の実装の提供が予定されています。
特に、CNSA 2.0（Commercial National Security Algorithm Suite 2.0）では、ML-KEM-1024の使用が求められているため、これらの実装が重要視されています。
なお、ML-KEM-512については、現時点での導入例がないため、少なくとも当面は除外されています。この提案は、Go 1.24のマイルストーンに含まれています。

--- 

今後も継続的にポスト量子暗号への移行は進められていきそうですが、議論を見ると、本当に今対応が必要なものかどうか、に対する議論が多く、その慎重さにGoらしさを感じました。

## 最後に

今回の記事ではGoのポスト量子暗号のサポートについて紹介しましたが、
来月1/18(土)に福岡で開催される[Gopher's Gathering](https://connpass.com/event/329963/)というイベントで、Go標準の暗号ライブラリ全体のメンテナンス戦略についてお話する予定なので、もし興味あれば是非話を聞きに来てください！

https://connpass.com/event/329963/

## 参考文献

https://words.filippo.io/dispatches/mlkem768/

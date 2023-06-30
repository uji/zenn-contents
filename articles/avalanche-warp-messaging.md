---
title: "Avalancheのクロスチェーンプロトコル、Avalanche Warp Messagingの仕組み"
emoji: "📮"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Avalanche", "ブロックチェーン"]
published: false
publication_name: "moneyforward"
---

社内の勉強会でAvalanche Warp Messagingの仕組みについて話したので、その内容をこちらにDumpします。

## Avalancheざっくり説明

- Avalancheは、高性能、高い拡張性、カスタマイズ性を併せ持つセキュアなプラットフォーム
  - これを実現するため、プラットフォーム上のブロックチェーン同士の高い相互運用性(インターオペラビリティ)を実現する仕組みが備わっている
  - イーサリアム・キラーのプラットフォームのうちの一つと言われている
- Avalancheは送金や取引の実施とプラットフォームの運営が別のチェーンに分別されている
  - プライマリネットワークは3つのチェーンによって構成されている

![](https://storage.googleapis.com/zenn-user-upload/19aefffc6930-20230630.png =500x)
*[公式ドキュメント](https://docs.avax.network/learn/avalanche/avalanche-platform)より引用*

- あらゆるニーズに対応するため、AvalancheにはSubnetと呼ばれる仕組みが存在する

## AvalancheのSubnetとは

![](https://storage.googleapis.com/zenn-user-upload/d6196979fddc-20230630.png =500x)
*Subnetとブロックチェーンの関係*

- SubnetはAvalancheバリデーターのグループであり、ブロックチェーンではない
- Subnetがブロックチェーンをホストする関係にある
- EthereumでいうConsensus Layerの話
  - Avalancheプラットフォーム上のブロックチェーンはSubnet、P-ChainをConsensus Layerの
- 1つのSubnetで複数のブロックチェーンをホストすることも可能

### プライマリネットワーク(P-Chain)との関係
![](https://storage.googleapis.com/zenn-user-upload/0f1ea66d0b87-20230630.png =500x)
*P-ChainとSubnetとバリデーターの関係*

- サブネットやバリデーターは、プライマリネットワークのP-Chainと呼ばれるプラットフォームの運営に特化したチェーンによって管理されている
- サブネット内のすべてのバリデーターは、Avalancheのプライマリネットワークのバリデーターにもなる必要がある
  - 2000 AVAXをステーキングすることでバリデーターになることが可能
- 各サブネットは他のサブネットから分離されているため、あるサブネットでの使用量が増加しても、別のサブネットのパフォーマンスに影響を与えることはない
- Subnetを作成することで、Avalancheのバリデーター資源を活用しつつ、さらに制限のないブロックチェーン構築環境が実現できる
  - バリデータ要件、実行ロジック、ガス代体系、インセンティブ設計などを独自にカスタムすることができる
- そこまで詳細なカスタム要件がない場合、Subnetを構築せずにP-Chainに独自のチェーンをホストさせることも可能

## Avalanche Warp Messaging（AWM）とは

- AWMは、Subnet同士の相互運用するための通信規格
- 2022年12月23日にリリースされたBanff 5(Avalanche Go v1.9.5)により、AWMを使った通信が実現できるようになった

https://twitter.com/avax/status/1605988995917299720?s=20

- これにより、ユーザーは自分自身のVMを構築し、他のサブネットと相互運用することが可能になる。
- また、他のサブネット上の契約にメッセージを送ることでdAppの機能を複合化することも可能。
- ブリッジに依存することなく通信できるため、ハッキングのリスクが低く安全

### 仕組み

参考: [@AvaxDevelopersによる解説スレッド](https://twitter.com/AvaxDevelopers/status/1668357327328796672)

## 他のエコシステムとの比較

## まとめ

## 参考文献

- 公式ホワイトペーパー 

https://www.avalabs.org/whitepapers

- Avalanche-Japaneseによるホワイトペーパーの概要

https://medium.com/ava-labs-jp/%EF%BC%93%E5%88%86%E3%81%A7%E8%AA%AD%E3%82%81%E3%82%8Bavalanche%E3%83%97%E3%83%A9%E3%83%83%E3%83%88%E3%83%95%E3%82%A9%E3%83%BC%E3%83%A0%E3%83%9B%E3%83%AF%E3%82%A4%E3%83%88%E3%83%9A%E3%83%BC%E3%83%91%E3%83%BC%E3%81%AE%E6%A6%82%E8%A6%81-36b275460fe8


- 公式ドキュメント What is Subnet? 

https://docs.avax.network/learn/avalanche/subnets-overview

- OKCoinJapan Youtubeチャンネル 「「誰も気づかない」Avalancheの魅力とは？」

https://www.youtube.com/watch?v=LJixjLex6HY

- coinbaseによる解説記事

https://www.coinbase.com/ja/cloud/discover/dev-foundations/intro-to-avalanche-subnets
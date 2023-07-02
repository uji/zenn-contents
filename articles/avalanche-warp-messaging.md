---
title: "AvalancheのSubnet間の通信プロトコル Avalanche Warp Messaging(AWM) の仕組み"
emoji: "📮"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Avalanche", "ブロックチェーン"]
published: true
published_at: 2023-07-03 20:00
publication_name: "moneyforward"
---

社内の勉強会で、AWMの仕組みについて扱ったので、そこでの学びをこちらにDumpします。
記事の内容に不備等あれば是非フィードバックいただきたいです。

## Avalancheざっくり説明

- Avalancheは、高性能、高い拡張性、カスタマイズ性を併せ持つセキュアなプラットフォーム
- 複数のブロックチェーンを、お互いが任意のデータを送受信できる高い相互運用性(インターオペラビリティ)を持った状態で動作させることで、これを実現しようとしている
- Avalancheは送金や取引の実施とプラットフォームの運営が別のチェーンに分別されている
  - プライマリネットワークは3つのチェーンによって構成されている
  - 本記事の文脈で重要となるのはP-Chain。P-Chainはプラットフォームの運営に特化したチェーン

![](https://storage.googleapis.com/zenn-user-upload/19aefffc6930-20230630.png =500x)
*[公式ドキュメント](https://docs.avax.network/learn/avalanche/avalanche-platform)より引用*
## AvalancheのSubnetとは

SubnetはAvalancheのプラットフォーム上に簡単かつ柔軟にブロックチェーンを構築するための仕組み

![](https://storage.googleapis.com/zenn-user-upload/d6196979fddc-20230630.png =500x)
*Subnetとブロックチェーンの関係*

- SubnetはAvalancheバリデーターのグループであり、ブロックチェーンではない
- Subnetがブロックチェーンをホストし、コンセンサスレイヤーの役割を担う
- 1つのSubnetで複数のブロックチェーンをホストすることも可能
![](https://storage.googleapis.com/zenn-user-upload/0f1ea66d0b87-20230630.png =500x)
*P-ChainとSubnetとバリデーターの関係*

- Subnetやバリデーターは、プライマリネットワークのP-Chainと呼ばれる、プラットフォームの運営に特化したチェーンによって管理されている
- Subnet内のすべてのバリデーターは、Avalancheのプライマリネットワークのバリデーターにもなる必要がある
  - 2000 AVAXをステーキングすることでバリデーターになることが可能
- 各Subnetは他のSubnetから分離されているため、あるSubnetでの使用量が増加しても、別のSubnetのパフォーマンスに影響を与えることはない

Subnetを作成することで、Avalancheのバリデーター資源を活用しつつ、自由にルールを設定したブロックチェーンを構築することができるようになる
- バリデーター要件、実行ロジック、ガス代体系、VM、インセンティブ設計まで独自にカスタムすることができる
## Avalanche Warp Messaging（AWM）とは

- AWMは、Subnet同士を相互運用するための通信規格
- 2022年12月23日にリリースされたBanff 5(Avalanche Go v1.9.5)により、AWMを使った通信が実現できるようになった

https://twitter.com/avax/status/1605988995917299720?s=20

### 仕組み

1. バリデーターは署名に利用するキーペアを持っており、公開鍵がP-Chainに登録されている
![](https://storage.googleapis.com/zenn-user-upload/222cdd8c8354-20230630.png =500x)
2. 送信するメッセージに対して行われるSubnetのバリデーターのBLS署名から集約署名(BLS Multi-Sigunature)を生成し送信する
![](https://storage.googleapis.com/zenn-user-upload/d879ca7f8b73-20230701.png =500x)
3. メッセージを受け取ったSubnet Bは、送信元であるSubnet Aのバリデーターの公開鍵を取得する
![](https://storage.googleapis.com/zenn-user-upload/09749bc259de-20230701.png =500x)
4. 得られた公開鍵をもとに署名を検証し、問題がなければトランザクションを発行する
![](https://storage.googleapis.com/zenn-user-upload/8640ae2d7707-20230701.png =500x)

- AvaLabsの図
![](https://storage.googleapis.com/zenn-user-upload/71497f836a77-20230630.png =500x)
*[@AvaxDevelopersによる解説スレッド](https://twitter.com/AvaxDevelopers/status/1668357356122693637?s=20) より引用*

https://twitter.com/AvaxDevelopers/status/1668357327328796672

- BLS署名、BLS署名の集約署名の技術を活用
  - これらはEthereum PoSのコンセンサスレイヤーでも利用されている
- 通信の形式に細かい指定はない
  - 公式が提供しているAWMの利用に対応したVMのリファレンス実装 [XSVM](https://github.com/ava-labs/xsvm) を見ると、通信用のAPIを独自に定義している
  - 必須となるバリデーターの合意の割合なども、Subnetごとに設定できる
```:リファレンス実装 XSVM のメッセージ送信APIの使われ方(XSVMのREADME参照)
<< POST
{
  "jsonrpc": "2.0",
  "method": "xsvm.message",
  "params":{
    "txID":<cb58 encoded>
  },
  "id": 1
}
>>> {"message":<json>, "signature":<bytes>}
```

- P-Chainを利用した検証により、ブリッジやオラクル、その他第三者が提供するサービスに依存することなく通信できる
- 類似プロトコルとしてよく比較されるPolkadot XCMは、より厳格なフォーマットを提供している
  -  Avalancheは柔軟性を重視する設計思想から、シンプルな規約のみを提供する意思決定をしていると考えられる
- Cosmosもプラットフォームネイティブな通信規格としてCosmos IBCを提供しているが、Cosmos IBCには、データを送受信するブロックチェーン以外への依存を限りなくなくそうとする設計思想があり、相互運用性を実現する仕組みも大きく異なる

## まとめ

- Avalancheには、Subnetと呼ばれる、複数のブロックチェーンを高い相互運用性を持って動作させる仕組みが存在する
- Subnetやバリデーターは、プライマリネットワークのP-Chainと呼ばれる、プラットフォームの運営に特化したチェーンによって管理されている
- Subnetを作成することで、Avalancheのバリデーター資源を活用しつつ、自由にルールを設定したブロックチェーンを構築することができるようになる
- AWMはSubnet同士の相互運用を実現する通信規格で、BLS署名の集約署名をP-Chainを介して検証することで、ブリッジやオラクル、その他第三者が提供するサービスに依存することなく通信できる
## 参考文献

- 公式ホワイトペーパー 

https://www.avalabs.org/whitepapers

- Avalanche-Japaneseによるホワイトペーパーの概要

https://medium.com/ava-labs-jp/%EF%BC%93%E5%88%86%E3%81%A7%E8%AA%AD%E3%82%81%E3%82%8Bavalanche%E3%83%97%E3%83%A9%E3%83%83%E3%83%88%E3%83%95%E3%82%A9%E3%83%BC%E3%83%A0%E3%83%9B%E3%83%AF%E3%82%A4%E3%83%88%E3%83%9A%E3%83%BC%E3%83%91%E3%83%BC%E3%81%AE%E6%A6%82%E8%A6%81-36b275460fe8

- 公式ドキュメント What is Subnet? 

https://docs.avax.network/learn/avalanche/subnets-overview

- 公式Twitter Spaceのまとめスレッド

https://twitter.com/Arcade__Max/status/1606032144207212544?s=20

- OKCoinJapan Youtubeチャンネル 「「誰も気づかない」Avalancheの魅力とは？」

https://www.youtube.com/watch?v=LJixjLex6HY

- coinbaseによる解説記事

https://www.coinbase.com/ja/cloud/discover/dev-foundations/intro-to-avalanche-subnets

- 公式によるAWMの利用に対応したVMのリファレンス実装 XSVM

https://github.com/ava-labs/xsvm

- Skyland Ventures 松本 頌平さんのnote記事
インターオペラビリティとCosmos IBCのいろは Vol.1 【UnUniFi 木村優さん】

https://note.com/sholana/n/n09687b639fc5
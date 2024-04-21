---
title: "Code-Hex/Synchroにファジングテストを実装して得た学び"
emoji: "🔧"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "test", "fuzzing"]
published: false
publication_name: "notahotel"
---

Goのタイムゾーン型安全なTimeパッケージCode-Hex/Synchroをご存じでしょうか。
Genericsを使って安全にタイムゾーンを扱えるTimeパッケージです。

```go
	utcNow := synchro.Now[tz.UTC]()
	jstNow := synchro.Now[tz.AsiaTokyo]()
	fmt.Println(utcNow)
	fmt.Println(jstNow)
	// Output:
	// 2023-09-02 14:00:00 +0000 UTC
	// 2023-09-02 23:00:00 +0900 JST
```

↓開発者Code-Hexさんによる、Go Conference mini 2023 Winter IN KYOTOでの発表資料

https://x.com/codehex/status/1730785158494896341

先日このパッケージにファジングテストを実装したのですが、実装する上で多くの学びがあったのでこの記事に残します。

https://github.com/Code-Hex/synchro/pull/33

## ファジングテストとは

テスト手法の一つで、ファズと呼ばれる大量のランダムデータを入力することで、対象がクラッシュしないかどうかを確認するテストです。
人間が見落としがちなエッジケースに到達できるため、脆弱性の発見に特に役立ちます。
Goでは1.18から標準機能としてサポートされ始めました。

https://go.dev/doc/security/fuzz/

## テスト対象

今回テストした対象は `synchro.ParseISO` です。
この関数は、ISO 8601に完全に準拠したパースをサポートしており、独自実装がなされています。
この実装に脆弱性が含まれないことをテストするのがファジングテスト実装の目的になります。

https://github.com/Code-Hex/synchro/blob/v0.5.2/synchro.go#L96-L112

Synchroのインスパイヤ元である[chrono](https://github.com/chronotope/chrono/blob/main/fuzz/fuzz_targets/fuzz_reader.rs)でも同様にファジングテストが実装されています。

## 学び
### Go標準のtime.ParseはRFCを完全に準拠しているわけではない

Fuzzingで得た学びというよりはsynchroを触って得た学びなのですが、time.ParseはRFC完全準拠ではありません。
RFCで許可されているすべての時間フォーマットを受け入れるわけではなく、正式に定義されていない時間フォーマットも受け入れます。
これはドキュメントにも説明があります。([godoc](https://pkg.go.dev/time#Parse:~:text=RFC3339%2C%20RFC822%2C%20RFC822Z%2C%20RFC1123%2C%20and%20RFC1123Z%20are%20useful%20for%20formatting%3B%20when%20used%20with%20time.Parse%20they%20do%20not%20accept%20all%20the%20time%20formats%20permitted%20by%20the%20RFCs%20and%20they%20do%20accept%20time%20formats%20not%20formally%20defined.
))

例えば `0000-01-01T0:00:00+00:00`(時間が一桁になってる)はRFC 3339/ISO 8601としてはアウトなんですが、Goのtime.Parseだとパースできてしまいます。
意図されていなかった動作ですが後方互換生維持のためにそのままにされているようです。

参考Issue

https://github.com/golang/go/issues/20555

### ファジングテストにも分類があり、目的が異なる

偏にファジングテストといってもいくつかの視点で分類が存在し、達成したい目的やテスト対象の性質によって使い分ける必要があります。

- 0からファズを生成するか、テスト実装者が用意したシードデータ（コーパスシード）を変更することで生成するか
- 入力構造を認識するか否か
- テスト対象の内部構造を認識するか否か
- テスト対象の仕様を認識するか否か

### テスト対象の仕様を認識するか否かで変化するテストの性質

ファジングテストはもともとブラックボックステスト手法として用いられてきましたが、テスト対象の構造や仕様を認識したホワイトボックステストとして利用することで別の目的を達成することも可能です。
（一部のみ認識するグレーボックステストとしてのファジングテストも存在する）

#### テスト対象の仕様を認識しないテスト（ブラックボックステスト）
- テスト対象のインターフェースのみがわかっている
- ファズをテスト対象に入力し、クラッシュ(予期せぬエラーやPanic、Data race)が発生したら失敗とみなす
- 脆弱性や堅牢性のテストで使われる
- 仕様に関する問題は検知できない

![](https://storage.googleapis.com/zenn-user-upload/3c94be7ffd10-20240427.png)

#### テスト対象の仕様を認識したテスト（ホワイトボックステスト）
- テスト対象のインターフェースに加えてプロパティ（特定の入力が与えられると出力として期待される特性）がわかっている
- ファズを入力した際に期待される出力結果も併せて生成する
- 出力結果の生成は、テスト対象の実装とは違ったロジックで出力結果を算出する関数（プロパティ関数）を用意し、利用する
- 入力値の何が問題だったかを人間にとってわかりやすい形式にしてくれるプロセス Shrink が存在する
- **リファクタリングや複雑なシナリオテスト**で使われる
- コミュニティや文脈によってはプロパティベーステスト(Property based testing)とも呼称される

![](https://storage.googleapis.com/zenn-user-upload/9dddb81261b7-20240427.png)

SynchroはISO 8601完全準拠という新しいプロパティを持ったGoライブラリなので、今回は脆弱性や堅牢性のテストのみを目的としたブラックボックステストとしてのファジングテストを実装しました。
もし、Synchroがtimeパッケージの仕様を引き継いだものであれば、仕様の差異もテストすることができるホワイトボックステストとしてのファジングテストを実装する方針が執れたかもしれません。

### Fuzzingの継続的インテグレーションの考え方

ファジングテストはタイムアウト(Go Fuzzing の場合 `fuzztime`)を設定しないとテストが失敗するまで無限に実行されるため、CIで実行する場合は注意が必要です。

- テストの実行頻度
- タイムアウト時間
- コーパスシードの設定
- 入力データの構造の制限

などを適切に設定し、いかにして十分なカバレッジを確保するか考える必要があります。

Go Fuzzingの場合、コマンドラインの出力にある **"new interesting"** の部分からコードカバレッジの拡大状況を確認できます。
これは、以前に生成されたどの入力によってもカバーされなかった新しいコードを実行させた入力を意味します。
得られたnew interestingはコーパスとして保持され、後続のファズの生成に利用されます。
new interestingカウントは通常、新しいコードパスが発見されるにつれて最初は急速に増加し、その後、コーパスがテスト対象のコードをより包括的にカバーするようになるにつれて減少します。

```
~ go test -fuzz FuzzFoo
fuzz: elapsed: 0s, gathering baseline coverage: 0/192 completed
fuzz: elapsed: 0s, gathering baseline coverage: 192/192 completed, now fuzzing with 8 workers
fuzz: elapsed: 3s, execs: 325017 (108336/sec), new interesting: 11 (total: 202)
fuzz: elapsed: 6s, execs: 680218 (118402/sec), new interesting: 12 (total: 203)
fuzz: elapsed: 9s, execs: 1039901 (119895/sec), new interesting: 19 (total: 210)
fuzz: elapsed: 12s, execs: 1386684 (115594/sec), new interesting: 21 (total: 212)
PASS
ok      foo 12.692s
```

以下は、今回実装したテストをGitHub Actions上で実行した際のコーパス(コマンドライン出力のtotal)の増加のグラフです。

![](https://storage.googleapis.com/zenn-user-upload/a08c73109c3c-20240502.png)
*Pull Requestより引用 横軸は秒数*

この結果をもとに、タイムアウトはnew interestingの増加が緩やかになる300秒に設定し、コミット毎に実行するようにしています。
コーパスシードの設定や、入力データ構造の制限はしていないので、これらを設定できるとより効率的にカバレッジを確保できそうです。

### OSS-Fuzz

今回、ファジングテストの継続実行にGitHub Actionsを利用しましたが、OSS-Fuzzと呼ばれるサービスを使う選択肢もあります。

https://google.github.io/oss-fuzz/

OSS-FuzzはGoogleが提供する無料のサービスで、OSSプロジェクトはOSS-Fuzzの管理するリポジトリに設定をPull Requestすることで、サービスによる継続的なファジングテストの実行サポートを受けることができます。
2016年の開始以来、OSS-Fuzzは1,000を超えるOSSプロジェクトにおいて、10,000以上の脆弱性と36,000以上のバグの特定と修正を支援してきたそうです。
皆さんご存じの有名OSSも多数利用しています。([利用プロジェクト一覧](https://github.com/google/oss-fuzz/tree/master/projects))

自分はまだOSS-Fuzzを利用することで得られる恩恵が正確に把握できていないのですが、CIツールの最大実行時間(GitHub Actionsは6時間)だと十分にカバレッジを確保できないような大きなテスト対象を扱うプロジェクトなどに良さそうです。

(LLMを使ったFuzzingフレームワークの研究なども進められてて、面白いなーと思いました)
https://github.com/google/oss-fuzz-gen

## 最後に

実装したファジングテストは結構な回数を実行したのですが、synchro.ParseISOがクラッシュすることはありませんでした。
Code-Hexさんの努力により、もともと高いテストカバレッジが確保されていたので、利用するにあたっての安全性への懸念はありませんでしたが、ファジングテストの結果が継続的に得られるようになったことでより安心して利用できそうです。
皆さんもぜひSynchroを使いましょう！

今回の実装を経て、Fuzzingに関する基本的な理解を得られましたが、コーパスシードの設定の勘所や、構造を制限する実装上の工夫、OSS-Fuzzの利用など、より効率的にカバレッジを確保するためのノウハウは自分の知らないことがたくさんありそうです。
機会があればこのあたりもキャッチアップして記事にできればと思います。

この記事やコントリビュートの内容で不備等あれば是非フィードバックいただきたいです。

### 参考文献

@ymotongpoo さんのGo1.18リリースパーティの登壇資料
https://docs.google.com/presentation/d/e/2PACX-1vQ2PX-s-As01o_fvGi9qdx9ZCpQS6RePDWw6rkznN-lI3z8bc4JJ601HLzb1fujo4uf0wSK0Wzl_oc1/pub?pli=1&resourcekey=0-R72bI85HvGnbMfYmAP_77g&slide=id.g405a9dc47b_0_0

https://future-architect.github.io/articles/20220214a/

https://www.techtarget.com/searchsecurity/definition/fuzz-testing

https://speakerdeck.com/sivchari/dive-into-testing-package-part-of-fuzzing-test

https://who3411.hatenablog.com/entry/2018/12/21/210754

https://speakerdeck.com/mururu/fuzzing-for-network-client-in-go

https://goyoki.hatenablog.com/entry/2023/12/29/003739

https://adalogics.com/blog/fuzzing-100-open-source-projects-with-oss-fuzz
---
title: "Goの別パッケージのテストはデフォルトではt.Parallel()の有無に限らず並列実行される"
emoji: "🐕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "test"]
published: false
publication_name: "notahotel"
---

Goでテストを書く際、 t.Parallel()を使ってテストを並列実行することはよくあります。しかし、別パッケージのテストは、t.Parallel()を使わなくても、デフォルトで並列実行されることをご存知でしょうか？（CPUコア数が2以上の場合に限ります）

この記事では、Goのテスト実行時の並列性について解説し、これに起因する問題とその解決方法について紹介します。特に、複数のパッケージでテストを実行する際に、どのような挙動がデフォルトで発生するのか、またそれをどのように制御できるのかに焦点を当てています。

タイトルは、初学者の方にも検索で見つけてもらいやすいよう、事象そのものをタイトルにしてみました。Goのテストについて学び始めた方がつまづくポイントに答える内容を意識しています。

## Goの複数パッケージの並列実行について

Goのtestingパッケージを使用したテストの実行は、t.Parallel()を使用しない限り逐次的に行われます。
ただし、この逐次実行は、同じパッケージ内のテストに限られます。

複数のパッケージに対してテストを実行する場合、パッケージごとにテストが並列で実行されます。(`./...`とすべてのパッケージを指定した場合も同様)
たとえば、aパッケージとbパッケージがある場合、それぞれのパッケージ内ではテストが逐次実行されますが、aパッケージとbパッケージのテストは並列で実行されます。

Goでは、go testコマンドを使って複数のパッケージを指定してテストを実行する場合、-pフラグ（ビルドの際にも使われるフラグ）によって、並列に実行されるパッケージ数を指定できます。
go help buildコマンドで-pフラグの説明を確認すると、次のように記載されています。

```
-p n
	the number of programs, such as build commands or
	test binaries, that can be run in parallel.
	The default is GOMAXPROCS, normally the number of CPUs available.

  訳
	ビルド・コマンドやテスト・バイナリなど、並列に実行できるプログラムの数を指定します。
	デフォルトはGOMAXPROCSで、通常は使用可能なCPUの数です。
```

つまり、テストに関しても、-pフラグで指定した値までのテストバイナリを並列に実行します。-pフラグを指定しない場合、デフォルトでは利用可能なCPUの数がその値となります。各プロセスは、自動的にどのパッケージのテストを実行するか割り振られ、各プロセス内では1つのパッケージのテストが逐次的に実行されます。たとえば、-p=1と指定すると、1つのプロセスのみでテストが実行されるため、すべてのパッケージのテストが順次実行されることになります。

-p とは別に -parallel というフラグも存在しますが、これは、同じパッケージ内で実行されるテスト関数の並列実行を制御するものです。

- **-p はパッケージ単位** の並列度を制御し、複数のパッケージのテストを並列で実行するかどうかを決める。
- **-parallel は各パッケージ内のテスト関数** の並列度を制御し、各パッケージ内でのテスト関数の実行が並列に行われる数を決める。

実際に、-pフラグで1より大きな値を指定して複数のパッケージをテストしながら、別のターミナルでpsコマンドを実行すると、パッケージごとに生成されたテスト用バイナリが並列で実行されているのが確認できます。

ちなみに、-pフラグを処理しているGo本体のコードは以下の部分です。

https://github.com/golang/go/blob/go1.23.1/src/cmd/go/internal/work/exec.go#L195-L230

sync.WaitGroupを使用して、-pで指定された数(cfg.BuildP)のgoroutineを制御しています。

このような並列実行の仕様は、共有リソースを扱うテストにおいて、思わぬ問題を引き起こすことがあります。
この並列実行によって実際にどのような問題が発生するか、自分が遭遇した問題の例を挙げて解説します。

## 遭遇した問題

NOT A HOTEL の Web APIサーバーでは、Cloud Spanner のあるテーブルの最新レコードのデータを参照するロジックがあり、これが複数箇所で再利用されていました。
このロジックを含む実装のテストが複数のパッケージで存在する場合、前述のとおりt.Parallel()の有無に限らず並列実行がなされてしまいます。
それぞれのテストで同じテーブルに対してレコードがインサートされるため、テストが並列に実行されてしまうと互いに干渉し合ってしまいます。

### この場合に取れる選択肢

Goのテストでパッケージ間の並列実行によってテスト結果に干渉が生じる場合、いくつかの対策を考えることができます。それぞれの選択肢について解説します。

#### 1. 並列実行を止める（go test -p 1）

最も簡単な方法は、go testコマンドに-p 1オプションを追加して、テストの並列実行を止めることです。これにより、すべてのテストが逐次実行され、テスト間の干渉を避けることができます。

しかし、並列実行を停止することでテストの実行時間が問題になる場合があります。特に大規模なプロジェクトでは、テスト時間の増加が開発速度に悪影響を与える可能性があります。
テスト数が少ない場合には有効ですが、テスト数が多い場合は別のアプローチを検討した方がよいでしょう。

#### 2. モックやスタブを使ったテストの分離

テスト間の干渉を防ぐために、共有リソース（例えば、データベース）を直接操作する代わりに、モックやスタブを使って依存する外部リソースをシミュレートする方法があります。
これにより、各テストが共有リソースに依存しなくなるため、並列実行による干渉を完全に防ぐことができますが、過度なモックやスタブの利用は、テストの複雑化や、忠実度が下がることによる信頼性低下のリスクを伴います。

参考
https://testing.googleblog.com/2013/05/testing-on-toilet-dont-overuse-mocks.html

#### 3. テストごとにトランザクションを使ってデータをロールバックする

テストの開始時にトランザクションを開始し、テスト終了時にそのトランザクションをロールバックすることで、データベースの状態を変更しないようにする方法です。
テスト間でデータが共有されないため、干渉を防ぐことができますが、複数のトランザクションを扱う処理や、データベースの外部に影響を及ぼす処理（例: 通知の送信など）をテストするのは困難です。

#### 4. テスト環境ごとに一時的なデータベースやテーブルを作成する

テストごとに一時的なデータベースやテーブルを作成し、テストが終了したら削除するというアプローチです。
各テストが共有のデータベース、テーブルを使うことはなくなるため、テストの干渉が起こる可能性がなくなりますが、
データベースのセットアップと管理が複雑になります。 データベースやテーブルの作成と削除による、実行時間やリソースのオーバーヘッドが問題になる場合があります。
この方法は、クラウド環境やコンテナ環境で一時的なリソースを簡単に作成・削除できる場合に特に有効です。

#### 5. ロックファイルを使った排他制御

最後に紹介するのはテスト実行中に排他制御を行うために、ロックファイルを使用する方法です。
この方法では、共有のデータベース、テーブルへのアクセスを制御するために、テストの開始時にロックを取得し、終了時にロックを解除します。Goのsyscallパッケージを利用して、ファイルベースのロックを実装することができます。
実装が若干複雑で、複数の実行環境で1つのDBを共有する場合など、環境によってはうまく機能しないことがあるのがデメリットですが、少ないオーバーヘッドでテストの並列度をできるだけ維持しながら、共有リソースへのアクセスを制御することが可能です。

以下は自分が実際に実装したテストヘルパーです。一番下のパブリックな関数を各テストの冒頭で呼び出すことで排他制御することが可能です。

```go
package testutil

import (
	"errors"
	"os"
	"syscall"
	"testing"
)

type lockType int16

const (
	lock   lockType = syscall.LOCK_EX // 排他ロック
	unlock lockType = syscall.LOCK_UN
)

// ファイルロックの実装はGo本体の internal package を参考にしている https://cs.opensource.google/go/go/+/refs/tags/go1.23.0:src/cmd/go/internal/lockedfile/internal/filelock/filelock_unix.go
func lockFile(t *testing.T, file *os.File, lockType lockType) {
	t.Helper()

	fd := file.Fd()
	var err error
	for {
		// ロックファイルを排他ロックする
		//nolint:gosec // reason: ほとんどの環境でファイルディスクリプターは小さな数値（通常は 1024 未満）になるためオーバーフローは考慮しない
		err = syscall.Flock(int(fd), int(lockType))
		if !errors.Is(err, syscall.EINTR) {
			break
		}
	}
	if err != nil {
		t.Fatalf("failed to lock file: %v", err)
	}
}

func lockFileAtPathWithCleanup(t *testing.T, filePath string) {
	t.Helper()

	file, err := os.OpenFile(filePath, os.O_CREATE|os.O_RDONLY, 0o440)
	if err != nil {
		t.Fatal(err)
	}

	lockFile(t, file, lock)

	t.Cleanup(func() {
		lockFile(t, file, unlock)
		file.Close()
	})
}

const xxxLockFilePath = "/tmp/xxx-lock-file"

// lockXxx は、指定されたパスに基づいて一時ファイルを作成し、排他制御を行う
// 共有リソースに対して書き込みが発生するテストの冒頭で使用すること
// サブテストが存在する場合は、親のテストでは呼ばず、サブテストの冒頭で呼び出すこと(-parallel の設定によってはタイムアウトになる可能性があるため)
// 別パッケージのテストは t.Parallel() の有無に限らず並列実行されてしまうため、その場合の並列実行を防ぐために使用する
//
// NOTE: 一時ファイルによる排他制御のため、CIなどで複数環境で1つのDBを共有する場合は対応できない
func LockXxx(t *testing.T) {
	lockFileAtPathWithCleanup(t, xxxLockFilePath)
}

```

他にも色々あると思いますが、きりがないのでこのあたりにしておきます。

### 最終的に選んだのは…

以下の理由から、最終的にロックファイルを使った方法を使って問題を解決しました。(上記で紹介した実装例とほぼ同じコードを使っています。)

- テストコードに加える変更が小さく収まり、対応方針の変更に柔軟に対応しやすい
- 少ないオーバーヘッドでテストの並列度をできるだけ維持しながら、共有リソースへのアクセスを制御できる
- 複数のトランザクションを扱う処理や、データベースの外部に影響を及ぼす処理のテストに対応したい
- CIの環境がシンプルな構成で、ロックファイルの運用が問題にならなかった

これは、あくまでも自分が遭遇した場面だと最適だと考えられた、というだけであって、プロダクトによって選ぶべき方法は変わってくると思います。
プロジェクトの規模や利用しているデータベース、求められるテストの忠実度や実行速度など、多様な要素を考慮して最適な方法を選ぶことが重要です。

## 最後に

この記事では、Goのテストにおける並列実行の仕様と、それによって引き起こされる問題、そしてそれらの問題に対処するためのいくつかの選択肢について解説しました。
特に、複数のパッケージをテストする際に発生するデータベースの干渉問題は、初学者やGoでのテストを始めたばかりの方がつまずきやすいポイントです。

テストの並列実行は、テスト時間の短縮や効率的なリソース利用といった利点がある一方で、データの競合やテストの不安定化といった課題も伴います。本記事で紹介した方法を使って、自分のプロジェクトに適した解決策を見つけていただければ幸いです。

もしこの記事が、Goのテストに関する問題を解決する一助となれば嬉しいです。また、この記事の内容についてご意見や改善点がありましたら、ぜひフィードバックをお寄せください。

### 参考文献

https://engineering.mercari.com/blog/entry/how_to_use_t_parallel/

https://tech.kanmu.co.jp/entry/2023/06/02/172458

https://tech.mfkessai.co.jp/2019/11/parallel-go-test/
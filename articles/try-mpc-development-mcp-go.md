---
title: "GoのModel Context Protocol (MCP)の開発フレームワークmcp-goを使ってみる"
emoji: "✨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "mcp", "AI", "cline"]
published: true
---

# Model Context Protocol (MCP) 概要

- MCPは、Claudeの開発元であるAnthropicによりLLMとローカル環境を接続するための標準プロトコル
- この規格で実装されたMCPサーバーを使うことで、LLMにローカルファイルの読み取りやAPIへのアクセスなどの機能を拡張できる
- JSON-RPC の仕様が規格化されているだけなので、様々なプログラミング言語で実装可能
- Claude DesktopやCursor、Clineなどでは既にMCPサーバーの利用がサポートされている

https://modelcontextprotocol.io/introduction

仕様（Draft）はこちら

https://spec.modelcontextprotocol.io/specification/draft/

# GoのMCP開発ライブラリ

Python, TypeScript, Java, Kotlin は公式からSDKが提供されていますが、(2025-03-21時点)
Goでライブラリを使ってMCPサーバーを実装する場合は、自前で実装するか、サードパーティライブラリに頼ることになります。

現在開発が活発そうなライブラリは以下2つ

- https://github.com/mark3labs/mcp-go
- https://github.com/metoro-io/mcp-golang

importedはmcp-goが68で一番多い (2025-03-21時点)ので今回こちらを試してみています。

https://pkg.go.dev/search?q=mcp

# Example を動かしてみる

https://github.com/mark3labs/mcp-go?tab=readme-ov-file#quickstart にすぐに試せるコードがあるのでそちらを動かしてみます。

operation(add, subtract, multiply, divide) と、計算に使うx, yを受け取って四則演算を行うシンプルなMCPサーバーの実装です。
除算のみ未対応の機能としてエラーを返すようにしてみています。

```go
package main

import (
	"context"
	"fmt"

	"github.com/mark3labs/mcp-go/mcp"
	"github.com/mark3labs/mcp-go/server"
)

func main() {
	// MCPサーバーインスタンスの作成
	s := server.NewMCPServer(
		"Calculator Demo",
		"1.0.0",
		server.WithResourceCapabilities(true, true), // Resource の機能で使われるオプションなのでToolの公開のみであれば不要そう
		server.WithLogging(),
	)

	// 四則計算ツールのインターフェース登録
	calculatorTool := mcp.NewTool("calculate",
		mcp.WithDescription("Perform basic arithmetic operations"),
		mcp.WithString("operation",
			mcp.Required(),
			mcp.Description("The operation to perform (add, subtract, multiply, divide)"),
			mcp.Enum("add", "subtract", "multiply", "divide"),
		),
		mcp.WithNumber("x",
			mcp.Required(),
			mcp.Description("First number"),
		),
		mcp.WithNumber("y",
			mcp.Required(),
			mcp.Description("Second number"),
		),
	)

	// 四則計算ツールを実装
	s.AddTool(calculatorTool, func(ctx context.Context, request mcp.CallToolRequest) (*mcp.CallToolResult, error) {
		op := request.Params.Arguments["operation"].(string)
		x := request.Params.Arguments["x"].(float64)
		y := request.Params.Arguments["y"].(float64)

		var result float64
		switch op {
		case "add":
			result = x + y
		case "subtract":
			result = x - y
		case "multiply":
			result = x * y
		case "divide":
			return mcp.NewToolResultError("未対応の機能です"), nil
		}

		return mcp.NewToolResultText(fmt.Sprintf("%.2f", result)), nil
	})

	// サーバー起動
	if err := server.ServeStdio(s); err != nil {
		fmt.Printf("Server error: %v\n", err)
	}
}

```

サーバー、ツール、どちらのオプションもGoでよくあるFunctional Option Patternで設定できるのですんなり使えるインターフェースになっている印象です。（ToolのDescriptionはOptionで良いのか…？など気になりはありますが）

## Cline で設定

Claude Desktopで試すのが一番簡単そうだったんですが、公式アプリはWindows, Macのサポートのみのため、自分が利用しているArch Linuxでは使えませんでした。今回はClineで動作させてみます。

ちなみに、DebianベースのOSであればこちらが使えるかもしれません。

https://github.com/aaddrick/claude-desktop-debian

Clineの利用画面の上部にあるMCP Serversの設定画面を開き、

![](https://storage.googleapis.com/zenn-user-upload/d8141d1b89d6-20250321.png)

Installedを選択。

![](https://storage.googleapis.com/zenn-user-upload/73b4a61292e7-20250321.png)

Configure MCP Servers から、設定ファイルを開くことができます。

![](https://storage.googleapis.com/zenn-user-upload/aea91b6ac303-20250321.png)

設定ファイルに実装したMCPサーバーの実行コマンドを書いたら設定完了です。
画像では、`go build` の成果物を実行コマンドとして設定してますが、`go run` コマンドの設定でも機能すると思います。
（画像には別で設定したMCP Brave Search の設定も含まれています。）

![](https://storage.googleapis.com/zenn-user-upload/a7b88ff80c72-20250321.png)

自分の環境だと設定ファイルを変更してすぐはうまく接続が確立されなかったのですが、VSCodeのReload Windowで解消しました。

## 動かしてみる

「100+200の計算をツールを用いて行ってください」とメッセージを投げると、

![](https://storage.googleapis.com/zenn-user-upload/5cdff02cb5d8-20250321.png)

MCPサーバーへのリクエストが提案されます。
Approveすると、

![](https://storage.googleapis.com/zenn-user-upload/6f3b5553f124-20250321.png)

何故か謝られてますが、MCPサーバーを使った計算タスクがこなせました。

除算を依頼すると、MCPサーバーが対応していないためエラーになりますが、echoを使った代替案を提案してくれました。
![](https://storage.googleapis.com/zenn-user-upload/00dbddd777c9-20250321.png)

# まとめ

Goでもサードパーティライブラリを使うことで簡単にMCPサーバーを実装することができました。
Goのライブラリ資源を再利用したい場合などは、Goで実装する意義があるかもしれません。
（例えばGo標準の暗号ライブラリの機能や、DockerなどGo製ツールの操作を提供するMCPは使いどころがありそう）

今回は、mcp-goを使ってToolを提供する実装を動かしてみましたが、mcp-goはResource（サーバーからLLMにデータとコンテンツを公開する仕組み）や、Prompt（LLMが効率的にMCPを使えるよう、再利用可能なプロンプトテンプレートとワークフローを提供する仕組み）などの機能もサポートしており、より高機能なMCPサーバーの実装もこなせそうです。

ただ、MCP自体現状は仕様がシンプルなので、ライブラリを使わずに net/http で実装してしまうでも事足りるのかもしれません。

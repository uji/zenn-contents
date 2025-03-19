---
title: "GoのModel Context Protocol (MCP)の開発フレームワークmcp-goを使ってみる"
emoji: "✨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["go", "mcp", "AI"]
published: false
---

# Model Context Protocol (MCP) 概要

- MCPは、Claudeの開発元であるAnthropicによりLLMとローカル環境を接続するための標準プロトコル
- この規格で実装されたMCPサーバーを使うことで、LLMにローカルファイルの読み取りやAPIへのアクセスなどの機能を拡張できる
- JSON-RPC で仕様化されているだけなので、様々なプログラミング言語で実装可能
- Claude DesktopやClineなどでは既にMCPサーバーの利用がサポートされている

https://modelcontextprotocol.io/introduction

仕様（Draft）はこちら

https://spec.modelcontextprotocol.io/specification/draft/

# GoのMCP開発ライブラリ

今開発が活発そうなのは以下2つ

- https://github.com/mark3labs/mcp-go
- https://github.com/metoro-io/mcp-golang

importedはmcp-goが68で一番多い (2025-03-20時点)

https://pkg.go.dev/search?q=mcp

# デモ
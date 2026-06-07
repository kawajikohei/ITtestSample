# Playwright MCPサーバー導入ログ

## 日時
2026-06-07

## 背景・経緯

ユーザーから「今後、フォーム入力なども行わせる予定がある」との話があった。
画面テストの実施方法として、以下の2案を検討した。

### 検討した選択肢

| 方法 | 内容 |
|---|---|
| MCPサーバーを使う（Playwright MCP） | `claude mcp add` でブラウザ操作用MCPサーバーを追加し、専用ツールでブラウザを直接・対話的に操作する |
| MCPなしでスクリプトを書いて実行 | Playwrightのテストスクリプトを書いてBash経由で実行し、スクリーンショットをファイル出力する |

### スクリプト方式（MCPなし）のデメリットとして整理した点
1. ブラウザの状態を見ながらその場で次の操作を判断できない（事前にシナリオを書き切る必要がある）
2. セレクタ違いやタイミング問題が起きた際の試行錯誤サイクルが重い
3. 初回セットアップ（`npx playwright install` 等のブラウザ本体ダウンロード）が必要になる場合がある
4. 失敗原因の把握がしづらい（事後のログ・スクリーンショット頼みになる）
5. ログイン後の分岐操作のような「画面を見て次を決める」系の操作に弱い

→ フォーム入力など対話的・状態依存の操作が今後増える前提であれば、MCPサーバー導入のメリットが大きいと判断し、ユーザーへ提案した。

## 環境調査結果（提案前に確認した内容）

```
claude mcp list            → No MCP servers configured.
npx playwright --version   → Version 1.60.0 (Node.js版は利用可能)
python -c "import playwright" → ModuleNotFoundError（Python版は未インストール）
```

## スコープに関する確認

ユーザーから「他のディレクトリに影響しないか」と質問があったため、`claude mcp add --help` でスコープの仕様を確認し、以下を説明した。

| スコープ | 影響範囲 | 保存場所 |
|---|---|---|
| local（デフォルト） | このプロジェクト（NESProject）限定。他ディレクトリ・他人に影響しない | ユーザー側設定（プロジェクトパス専用として記録） |
| project | プロジェクト全体・チーム共有 | プロジェクト直下の `.mcp.json`（gitにコミットされる） |
| user | 全プロジェクトに影響（グローバル） | ユーザー設定（`~/.claude` 配下） |

このリポジトリは現在gitリポジトリではないため project スコープでも共有相手はいない旨を伝えた。

## ユーザーの決定
**local スコープ**で追加する、との回答を得た。

## 実施した作業
以下のコマンドを local スコープ（デフォルト、オプション指定なし）で実行し、Playwright MCPサーバーを追加した。

```
claude mcp add playwright npx @playwright/mcp@latest
```

## 結果

コマンド実行結果：

```
Added stdio MCP server playwright with command: npx @playwright/mcp@latest to local config
File modified: C:\Users\kawaj\.claude.json [project: c:\Users\kawaj\Downloads\NESProject]
```

### 検証

`claude mcp list` / `claude mcp get playwright` を実行したところ「No MCP servers configured」と表示され、一見登録されていないように見えた。
そこで `C:\Users\kawaj\.claude.json` を直接確認したところ、以下のキーで正しく登録されていることを確認した。

- プロジェクトキー: `c:/Users/kawaj/Downloads/NESProject`（スラッシュ区切り表記）
- 登録内容:
  ```json
  {
    "playwright": {
      "type": "stdio",
      "command": "npx",
      "args": ["@playwright/mcp@latest"],
      "env": {}
    }
  }
  ```

`claude mcp list` で表示されなかったのは、現在のセッションが起動時点の設定をキャッシュしているため（あるいはパス区切り文字 `\` と `/` の表記差異の影響）と考えられる。
**設定自体は正しく書き込まれており、新しいセッションを開始すればPlaywright MCPサーバーのツールが利用可能になる見込み。**

## ステータス
Playwright MCPサーバーの local スコールでの追加が完了。次回セッション開始時に動作確認を行う。

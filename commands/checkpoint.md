---
description: セッション内チェックポイントを保存（compact前に推奨）
allowed-tools: Read, Write, Bash, Glob
---

セッション中の進捗をチェックポイントとして外部ファイルに保存してください。`/compact` 前に実行することで、圧縮で失われる詳細情報を退避できます。

## 1. プロジェクトの特定

```bash
git rev-parse --show-toplevel 2>/dev/null || pwd
```

このパスを `$PROJECT_ROOT` とする。

**プロジェクトスラグの生成**:

```bash
echo $HOME
```

`$PROJECT_ROOT` のパスからホームディレクトリ（上記の結果）を除去し、`/` を `-` に置換したものを `$SLUG` とする。
例: ホームが `/Users/alice` で `$PROJECT_ROOT` が `/Users/alice/dev/my-project` → `dev-my-project`

## 2. セッションディレクトリの準備

```bash
mkdir -p ~/.claude/sessions/$SLUG
```

## 3. 変更情報の収集

git リポジトリの場合、以下を実行:

```bash
git status --short
git diff --stat
git diff --staged --stat
```

非 git リポジトリの場合はスキップし、会話履歴のみで分析する。

## 3.5. セッション種別の判定

ステップ3の結果と会話内容から、セッションの種別を判定する:

- **コード変更あり**: git diff/status に差分がある、またはソースファイルの作成・編集を行った
- **コード変更なし**: リサーチ、設計議論、Vault管理、設定変更など

この判定に基づいて、ステップ5のテンプレートで含めるセクションを決定する。

## 4. 会話履歴の分析

**会話の全履歴を振り返り**、以下を把握する:

- セッションの目的（ユーザーが最初に依頼した内容）
- 完了したタスク
- 進行中のタスク（現在の状態を含む）
- 未着手のタスク
- 重要な判断とその理由
- 変更したファイルとその説明

## 5. チェックポイントファイルの生成

タイムスタンプを取得:

```bash
date "+%Y%m%d-%H%M"
```

保存先: `~/.claude/sessions/$SLUG/checkpoint-{YYYYMMDD-HHMM}.md`

引数でフェーズ名が指定されている場合はそれを見出しに使用し、なければ会話内容から自動要約する。

### テンプレート

```markdown
# Checkpoint: {フェーズ名 or 自動要約}

- **Project**: {プロジェクト名}
- **Directory**: {$PROJECT_ROOT}
- **Timestamp**: {YYYY-MM-DD HH:MM}

## Objective

セッションの目的を1-2文で記述。

## Completed

- 完了タスク1
- 完了タスク2

## In Progress

- 進行中タスク — 現在の状態の説明

## Pending

- 未着手タスク1
- 未着手タスク2

## Key Decisions

- **判断内容**: 理由の説明（このセクションが最も重要 — 次回の自分が「なぜ？」と思わないように）

## Changed Files（コード変更ありの場合）

- `path/to/file` — 変更内容の説明

## Key Artifacts（コード変更なしの場合、Changed Files の代わりに使用）

- 作成・参照したファイル、ノート、URL、成果物の一覧と説明
  （例: Obsidian ノート、設計ドキュメント、調査結果 URL、設定ファイルなど）
```

> **機密情報の除外**: API キー、パスワード、トークン、接続文字列、.env ファイルの内容等の機密情報は含めないこと。Related Files / Changed Files に .env, credentials.json 等の機密ファイルを列挙しない。

### サイズ制約

- **目標**: 2KB
- **上限**: 4KB
- 簡潔さを優先し、冗長な説明は避ける。箇条書きを活用する。

## 6. 完了報告

以下を表示する:

- 保存したファイルのパス
- チェックポイントの要約（3行以内）
- 「`/compact` を実行してコンテクストウィンドウを解放できます」という提案

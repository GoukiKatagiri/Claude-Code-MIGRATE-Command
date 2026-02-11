---
description: コンテクスト長の回避のため新セッションへ移行（checkpoint→handoff→resume を一括実行）
allowed-tools: Read, Write, Bash, Glob
---

コンテクストウィンドウの飽和を回避するため、現在のセッションのコンテクストを外部化し、新しいセッションに移行してください。checkpoint → handoff → 新セッション起動 → 自動 resume を一括で実行します。

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

## 3. タイムスタンプの取得

```bash
date "+%Y%m%d-%H%M"
```

以降、この値を `$TS` とする。

## 4. Checkpoint の作成

会話の全履歴を振り返り、以下の内容でチェックポイントファイルを生成する。

保存先: `~/.claude/sessions/$SLUG/checkpoint-$TS.md`

```markdown
# Checkpoint: {フェーズ名 or 自動要約}

- **Project**: {プロジェクト名}
- **Directory**: {$PROJECT_ROOT}
- **Timestamp**: {YYYY-MM-DD HH:MM}

## Objective

セッションの目的を1-2文で記述。

## Completed

- 完了タスク一覧

## In Progress

- 進行中タスク — 現在の状態

## Pending

- 未着手タスク一覧

## Key Decisions

- **判断内容**: 理由の説明

## Changed Files（コード変更ありの場合）

- `path/to/file` — 変更内容の説明

## Key Artifacts（コード変更なしの場合、Changed Files の代わりに使用）

- 作成・参照したファイル、ノート、URL、成果物の一覧と説明
```

> **適応ルール**: git diff/status が空かつソースファイルの編集がない場合は「コード変更なし」と判定し、Changed Files の代わりに Key Artifacts を使用する。

サイズ目標: 2KB、上限: 4KB

## 5. Handoff の作成

チェックポイントと会話全履歴を基に、引き継ぎドキュメントを生成する。

git リポジトリの場合、追加情報を収集:

```bash
git status --short
git diff --stat
git log --oneline -10
```

保存先: `~/.claude/sessions/$SLUG/handoff-$TS.md`

```markdown
# Handoff: {セッション概要}

- **Project**: {プロジェクト名}
- **Directory**: {$PROJECT_ROOT}
- **Timestamp**: {YYYY-MM-DD HH:MM}

## Context

プロジェクトと目的の説明（2-3文）。

## What Was Done

- 完了した作業一覧

## Current State

作業対象の現在の状態。コード変更ありならビルド/テスト/デプロイの可否、なしなら調査・設計等の進捗。

## Open Issues

- 未解決の問題と原因

## Next Steps

1. **[高]** 最優先タスク
2. **[中]** 次のタスク
3. **[低]** 余裕があれば

## Key Decisions & Rationale

- **判断内容**: 理由

## Architecture Notes（コード変更ありの場合）/ Background Knowledge（コード変更なしの場合）

コード変更あり: 構造情報、依存関係、設計パターン。なし: 前提知識・調査結果の要約。該当なければ省略可。

## Related Files

次のステップで必ず参照すべきリソース（上位5件）: ソースコード、Obsidian ノート、設定ファイル、参考URLなど。

1. `path/to/file1` — 理由
2. `path/to/file2` — 理由
```

> **適応ルール**: Checkpoint と同様、git diff/status が空かつソースファイルの編集がない場合は「コード変更なし」と判定。Current State は進捗状況を、Architecture Notes は Background Knowledge に差し替える。

サイズ目標: 3KB、上限: 6KB

## 6. MEMORY.md にマーカーを追記

**MEMORY.md パスの特定**:

```bash
echo "$(pwd | tr '/' '-')"
```

結果を `$PROJ_ID` とする。MEMORY.md のパス: `~/.claude/projects/$PROJ_ID/memory/MEMORY.md`

このファイルを Read し、末尾に以下を追記する:

```markdown

## Pending Resume

前セッションの引き継ぎ: `~/.claude/sessions/$SLUG/handoff-$TS.md`
このファイルを Read してコンテクスト復元を行い、完了後このセクションを削除すること。
```

このマーカーは `/resume` 実行時に自動削除される安全策。

## 7. 新 Terminal タブの起動

> **注意**: このステップは macOS の Terminal.app を前提としています。iTerm2 や他のターミナルを使用している場合は、AppleScript 部分を適宜調整してください。Linux / Windows (WSL) の場合はステップ8のフォールバック手順を参照してください。

`$HANDOFF_PATH` に handoff ファイルのフルパスを設定し、以下を実行:

```bash
osascript -e "tell application \"Terminal\"" \
  -e "activate" \
  -e "do script \"cd '$PROJECT_ROOT' && claude '/resume $HANDOFF_PATH'\"" \
  -e "end tell"
```

## 8. 完了報告

以下を表示する:

```
## セッション移行完了

- **Checkpoint**: ~/.claude/sessions/$SLUG/checkpoint-$TS.md
- **Handoff**: ~/.claude/sessions/$SLUG/handoff-$TS.md
- **MEMORY.md**: Pending Resume マーカーを追記済み
- **新セッション**: Terminal の新タブで起動中

新セッションで `/resume` が自動実行され、コンテクストが復元されます。
このタブは `/exit` で終了してください。
```

### フォールバック

新セッションが自動起動しない場合（macOS 以外の環境、または osascript でエラーが発生した場合）:

1. 新しいターミナルタブ/ウィンドウを手動で開く
2. `cd $PROJECT_ROOT && claude '/resume'` を実行する
3. MEMORY.md の Pending Resume マーカーにより、Claude は handoff の存在を自動認識する

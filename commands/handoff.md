---
description: セッション終了時の引き継ぎドキュメント生成（/resume で読み込み）
allowed-tools: Read, Write, Bash, Glob
---

セッション終了時に、次セッションの Claude が作業を引き継げる包括的なドキュメントを生成してください。

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

## 3. 既存チェックポイントの確認

```bash
ls -t ~/.claude/sessions/$SLUG/checkpoint-*.md 2>/dev/null | head -1
```

最新のチェックポイントが存在すれば Read し、引き継ぎドキュメントの補完情報として活用する。

## 4. 変更情報の収集

git リポジトリの場合:

```bash
git status --short
git diff --stat
git log --oneline -10
```

非 git リポジトリの場合はスキップ。

## 5. 会話履歴の総合分析

**会話の全履歴を振り返り**、以下を整理する:

- プロジェクトの背景と目的
- 完了した全作業
- 作業対象の現在の状態（コード変更ありの場合はビルド・テスト・デプロイの可否、なしの場合は調査・設計・ドキュメント等の進捗状況）
- 未解決の問題とその原因
- 次にやるべきこと（優先度順）
- 重要な判断とその理由
- 次セッションで知るべき構造情報または前提知識
- 次のステップで必ず参照すべきリソース

## 6. 引き継ぎファイルの生成

タイムスタンプを取得:

```bash
date "+%Y%m%d-%H%M"
```

保存先: `~/.claude/sessions/$SLUG/handoff-{YYYYMMDD-HHMM}.md`

### テンプレート

```markdown
# Handoff: {セッション概要}

- **Project**: {プロジェクト名}
- **Directory**: {$PROJECT_ROOT}
- **Timestamp**: {YYYY-MM-DD HH:MM}

## Context

プロジェクトと目的の説明を2-3文で。次セッションの Claude が初見でも理解できるように。

## What Was Done

- 完了した作業1
- 完了した作業2

## Current State

作業対象の現在の状態。コード変更ありの場合はビルド/テスト/デプロイの可否。コード変更なしの場合は調査・設計・ドキュメント等の進捗状況。

## Open Issues

- **問題**: 原因と影響の説明

## Next Steps

1. **[高]** 最優先でやるべきこと
2. **[中]** 次にやるべきこと
3. **[低]** 余裕があればやること

（このセクションが最も重要 — 次セッションのアクションプランになる）

## Key Decisions & Rationale

- **判断内容**: 理由の説明。代替案があった場合はなぜ選ばなかったかも。

## Architecture Notes（コード変更ありの場合）/ Background Knowledge（コード変更なしの場合）

コード変更ありの場合: 次セッションで知るべき構造情報。依存関係、設計パターン、注意点など。
コード変更なしの場合: 次セッションが知るべき前提知識・調査結果の要約。該当なければ省略可。

## Related Files

次のステップで必ず参照すべきリソース（上位5件）: ソースコード、Obsidian ノート、設定ファイル、参考URLなど。

1. `path/to/file1` — 理由
2. `path/to/file2` — 理由
```

### サイズ制約

- **目標**: 3KB
- **上限**: 6KB
- 情報密度を高く保ち、冗長な説明は避ける。

## 7. 完了報告

以下を表示する:

- 保存したファイルのパス
- 引き継ぎドキュメントの要約（5行以内）
- 「`/devlog` で変更記録も残せます」という提案
- 新セッションでは `/resume` で引き継ぎを読み込めること

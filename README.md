# Claude Code Session Commands

Claude Code のセッション間でコンテクストを引き継ぐための4つのカスタムコマンド。auto-compact による受動的な圧縮ではなく、意図的に構造化された情報を次セッションに渡します。

| コマンド | 用途 |
|---|---|
| `/checkpoint` | セッション内チェックポイントを保存（`/compact` 前に推奨） |
| `/handoff` | セッション終了時の引き継ぎドキュメント生成 |
| `/resume` | 前セッションの引き継ぎを読み込みコンテクスト復元 |
| `/migrate` | handoff → 新セッション起動 → resume を一括実行 |

## ワークフロー

```
セッション内:  作業 → /checkpoint → /compact → 作業続行
セッション終了: /handoff → 終了
次セッション:   /resume → コンテクスト復元 → 作業続行
ワンコマンド:   /migrate → handoff + 新セッション起動 + resume を一括実行
```

## 導入方法

### A. Claude Code で導入（推奨）

Claude Code に以下のプロンプトを貼るだけで導入できます:

```
このリポジトリのセッション管理コマンドを自分の環境に導入してください:
https://github.com/GoukiKatagiri/Claude-Code-MIGRATE-Command

1. commands/ 内の4ファイルを ~/.claude/commands/ にコピー
2. ~/CLAUDE.md にコマンド一覧を追記（重複がなければ）
3. 導入結果を報告
```

### B. 手動導入

```bash
# 1. リポジトリをクローン
gh repo clone GoukiKatagiri/Claude-Code-MIGRATE-Command /tmp/Claude-Code-MIGRATE-Command

# 2. コマンドファイルをコピー
mkdir -p ~/.claude/commands
cp /tmp/Claude-Code-MIGRATE-Command/commands/*.md ~/.claude/commands/

# 3. クリーンアップ
rm -rf /tmp/Claude-Code-MIGRATE-Command
```

#### CLAUDE.md への追記（任意）

`~/CLAUDE.md` にコマンド一覧を追記しておくと、Claude がコマンドの存在を認識しやすくなります:

```markdown
# セッション管理コマンド
| コマンド | 用途 |
|---|---|
| `/checkpoint` | セッション内チェックポイントを保存（compact前に推奨） |
| `/handoff` | セッション終了時の引き継ぎドキュメント生成 |
| `/resume` | 前セッションの引き継ぎを読み込み |
| `/migrate` | 新セッションへ移行（handoff→resume を一括） |
```

## 各コマンドの説明

### `/checkpoint` — セッション内チェックポイント

セッション中の進捗をスナップショットとして外部ファイルに保存します。`/compact` でコンテクストウィンドウを圧縮する前に実行することで、圧縮で失われる詳細情報を退避できます。

**保存される情報:**
- セッションの目的と完了/進行中/未着手タスク
- 重要な判断とその理由
- 変更ファイル一覧（コード変更あり）またはキーアーティファクト一覧（コード変更なし）

**使いどころ:**
- `/compact` の直前
- 複雑なタスクの区切りで中間セーブしたいとき

### `/handoff` — セッション終了時の引き継ぎ

セッション終了時に、次セッションの Claude が作業を引き継げる包括的なドキュメントを生成します。checkpoint より詳細で、次のアクションプラン（Next Steps）に重点を置いています。

**テンプレート構成:**
- Context（プロジェクト概要）
- What Was Done（完了した作業）
- Current State（現在の状態）
- Open Issues（未解決の問題）
- Next Steps（優先度付きアクションプラン）
- Key Decisions & Rationale（判断と理由）
- Architecture Notes / Background Knowledge
- Related Files（次セッションで参照すべきファイル）

### `/resume` — 引き継ぎ読み込み

前セッションの handoff / checkpoint を読み込み、コンテクストを復元します。

**引き継ぎファイルの解決優先順位:**
1. 引数でパスが指定されている場合 → そのファイルを使用
2. MEMORY.md に `Pending Resume` マーカーがある場合 → 記載された handoff パスを使用
3. 自動検出 → `~/.claude/sessions/{slug}/` から最新の handoff（なければ checkpoint）を検索

Related Files はパス一覧として表示され、必要時に都度 Read する lazy loading 方式です。

### `/migrate` — ワンコマンドで新セッションに移行

handoff → 新セッション起動 → 自動 resume を一括実行します。コンテクストウィンドウが飽和してきたとき、1コマンドで新セッションに移行できます。

**実行される処理:**
1. Handoff ファイルの作成
2. MEMORY.md に Pending Resume マーカーを追記
3. 新しい Terminal タブで `claude '/resume'` を起動

> **Note:** 新セッションの自動起動は macOS の Terminal.app を前提としています。iTerm2 や他のターミナル、Linux / Windows (WSL) の場合は手動で新ターミナルを開き `claude '/resume'` を実行してください。MEMORY.md の Pending Resume マーカーにより、handoff ファイルは自動検出されます。

## 仕組み

### 保存先

すべてのセッションファイルは `~/.claude/sessions/{project-slug}/` に保存されます。

プロジェクトスラグはプロジェクトルートのパスから自動生成されます:
```
$PROJECT_ROOT からホームディレクトリを除去し、/ を - に置換
例: ~/dev/my-project → dev-my-project
```

### 引き継がれるもの

- セッションの目的と達成状況
- 重要な判断とその理由（**最も重要** — 次回の自分が「なぜ？」と思わないように）
- 次にやるべきこと（優先度付き）
- 関連ファイルのパスと役割
- アーキテクチャ情報や前提知識

### 失われるもの

- 会話の逐語的な内容（要約に圧縮される）
- 試行錯誤の詳細プロセス
- ツール実行の生の出力

### コンテクスト消費量

- Checkpoint: 目標 2KB / 上限 4KB
- Handoff: 目標 3KB / 上限 6KB
- `/resume` で復元時の消費: handoff 本体の ~750 tokens のみ（Related Files は lazy loading）

## 設計思想

Claude Code の auto-compact はコンテクストウィンドウが溢れたときに自動で圧縮を行いますが、**何を残し何を捨てるかの判断は Claude 任せ**です。結果として、重要な判断理由や作業の文脈が失われることがあります。

このコマンドセットは **意図的な構造化引き継ぎ** を行います:
- 「なぜその判断をしたか」を明示的に記録
- 次のアクションプランを優先度付きで整理
- 参照すべきファイルを事前に特定

また、セッションの種別（コード変更あり/なし）を自動判定し、テンプレートを適応させます。コーディング作業では Changed Files と Architecture Notes を、リサーチや設計作業では Key Artifacts と Background Knowledge を記録します。

## License

MIT

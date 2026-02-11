---
description: 前セッションの引き継ぎを読み込みコンテクストを復元
allowed-tools: Read, Write, Bash, Glob
---

前セッションの引き継ぎドキュメントを読み込み、コンテクストを復元してください。

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

## 2. 引き継ぎファイルの決定

以下の優先順位で引き継ぎファイルを決定する:

### A. 引数でファイルパスが指定されている場合

`$ARGUMENTS` に含まれるパスをそのまま使用する。

### B. MEMORY.md に Pending Resume マーカーがある場合

**MEMORY.md パスの特定**:

```bash
echo "$(pwd | tr '/' '-')"
```

結果を `$PROJ_ID` とする。MEMORY.md のパス: `~/.claude/projects/$PROJ_ID/memory/MEMORY.md`

このファイルを Read し、`## Pending Resume` セクションが存在するか確認する。存在する場合、そこに記載された handoff パスを使用する。

### C. 自動検出

以下の順で最新ファイルを検索:

```bash
ls -t ~/.claude/sessions/$SLUG/handoff-*.md 2>/dev/null | head -1
```

handoff が見つからない場合:

```bash
ls -t ~/.claude/sessions/$SLUG/checkpoint-*.md 2>/dev/null | head -1
```

いずれも見つからない場合は「引き継ぎファイルが見つかりません」と報告して終了。

## 3. 引き継ぎファイルの読み込み

決定したファイルを Read して全内容を取得する。

## 4. Related Files の先行読み込み

引き継ぎファイルに `## Related Files` セクションがある場合、リストされたファイルの**上位3件**を Read する。ファイルが存在しない場合はスキップする。

## 5. MEMORY.md マーカーのクリーンアップ

ステップ2Bで特定した MEMORY.md パス（`~/.claude/projects/$PROJ_ID/memory/MEMORY.md`）を Read し、`## Pending Resume` セクションが存在する場合は Edit で該当セクション全体（見出し行から次の `##` 見出し行の直前、またはファイル末尾まで）を削除する。

削除対象: `## Pending Resume` の行から、その内容行すべて（次の `##` 見出しの直前まで、または末尾の空行まで）。

## 6. 復元内容の表示

以下のフォーマットで要約を表示する:

```
## セッション復元完了

**前回のセッション**: {Handoff/Checkpoint のタイトル}
**プロジェクト**: {プロジェクト名}

### 前回の成果
- {完了した作業の要約}

### 今回やるべきこと
1. {Next Steps から優先度順}

### 読み込んだ関連ファイル
- {先行 Read したファイル一覧}

---
コンテクストが復元されました。作業を続行できます。
```

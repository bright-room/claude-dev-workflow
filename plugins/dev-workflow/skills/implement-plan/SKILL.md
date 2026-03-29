---
name: implement-plan
description: Issue 上の実装プランまたはローカル Markdown を元にコードを実装する。Issue 番号またはファイルパスを指定する。
argument-hint: "<issue-number> | <markdown-file-path> [--branch <branch-name>]"
---

# Implement Plan Skill

実装プランに基づいてコードを実装する。入力ソースは以下の2つ:

- **Issue 番号**: Issue コメント上の実装プラン（`<!-- claude:plan -->` マーカー）を読み取って実装する
- **Markdown ファイルパス**: ローカルの Markdown ファイルを読み込んで実装する

**このスキルはプランに基づく新規実装のみを行う。** PR のレビュー指摘への対応は `/fix-review` スキルを使用すること。

## 前提条件

- `gh` CLI が認証済みであること

## 引数

```
$ARGUMENTS = <issue-number> | <markdown-file-path> [--branch <branch-name>]
```

### 入力モードの判定

| 引数パターン | モード | 動作 |
|-------------|--------|------|
| 数値のみ（例: `42`） | **Issue プラン実装** | Issue コメントからプランを読み取り実装 |
| ファイルパス [+ `--branch`]（例: `plan.md`） | **Markdown 実装** | ローカル Markdown を読み込んで実装 |

引数なしの場合はエラーとする。

## 手順

### 1. 入力ソースの取得

#### モード A: Issue プラン実装

Issue コメントから `<!-- claude:plan -->` マーカー付きのプランを抽出する。

```bash
# Issue のコメント一覧からプランコメントを取得
gh api 'repos/{owner}/{repo}/issues/<issue-number>/comments' \
  --jq '.[] | select(.body | contains("<!-- claude:plan -->"))'
```

- プランコメントが見つからない場合はエラーメッセージを出力して終了する
- 複数のプランコメントが見つかった場合は、最新（最後に投稿された）コメントを採用する
- プランの Phase / Step の構成、対象ファイル、変更内容を把握する

#### モード B: Markdown 実装

指定された Markdown ファイルを読み込み、内容を深く理解する。

- **実装プランの場合**: Phase / Step の構成、対象ファイル、変更内容を把握する
- **その他の Markdown**: 記述された要件・仕様を把握する

### 2. プロジェクト固有情報の確認

`.claude/skills/references/project-context.md` の「実装ガイド」セクションを読み込み、以下を把握する:

- ビルド・フォーマットコマンドと注意事項
- 言語固有の実装規約
- テストの配置ルール
- 実装順序（モノレポの場合）
- CI に委ねてよい項目

### 3. ブランチの準備

#### モード A（Issue プラン実装）: 新規ブランチを作成

1. `main` ブランチの最新を取得し、そこから新規ブランチを作成する
2. ブランチ名: `feat/<issue-number>-<概要のケバブケース>`

#### モード B（Markdown 実装）: 引数に依存

| 引数 | 動作 |
|------|------|
| `--branch` なし | Markdown の内容からブランチ名を自動生成し、`main` から新規ブランチを作成 |
| `--branch <existing-branch>` | 指定されたブランチにチェックアウトし、最新を pull する |

ブランチ名の自動生成ルール:
- 実装プランの場合: `feat/<issue-number>-<概要のケバブケース>`
- その他: `feat/<概要のケバブケース>`

### 4. コードの実装

入力ソースの内容に基づいてコードを実装する。

#### 実装時の注意事項

- 既存のコードベースのパターン・命名規則に従うこと
- project-context.md に記載された言語固有の実装規約に従うこと
- 実装プランがある場合は Phase / Step の順序に従って段階的に実装すること
- 各ステップの実装後、ビルドが通ることを確認すること

#### 実装の進め方

project-context.md の「実装順序」に従って実装する。記載がない場合は以下のデフォルト順序で進める:

1. **プロダクトコードの実装**: 新規ファイルの作成、既存ファイルの修正
2. **テストコードの実装**: project-context.md の「テスト配置ルール」に従って配置する
3. **ドキュメントの更新**: README、CLAUDE.md、`.claude/rules/*.md` など（構成変更がある場合）

### 5. ビルドとフォーマットの確認

project-context.md の「ビルド・フォーマットコマンド」セクションに記載されたコマンドを実行する。

- フォーマッターは必ずコミット前に実行すること
- ビルドやテストが失敗した場合は原因を特定し修正すること。修正後に再度実行し、成功するまで繰り返す
- CI に委ねてよい項目として記載されたテストはスキップしてよい

### 6. コミット

変更内容をコミットする。

- コミットメッセージは変更内容を適切に要約すること
- Issue 番号をコミットメッセージに含めること
  - 例: `feat: add new feature (#42)`
- 複数の論理的なまとまりがある場合は、適切にコミットを分割すること
- Co-Authored-By には実行時のモデル情報を使用すること

```bash
git add <files>
git commit -m "$(cat <<'EOF'
<commit message>

Co-Authored-By: <実行中のモデル名> <noreply@anthropic.com>
EOF
)"
```

### 7. Push と PR 作成

#### モード A（Issue プラン実装）: Push して PR 作成

1. リモートに Push する

```bash
git push -u origin <branch-name>
```

2. PR を作成する。Issue を紐づけるため **PR 本文** に `Closes #<issue-number>` を記載する。

```bash
gh pr create --title "<PR title>" --body "$(cat <<'EOF'
## Summary
<変更内容の箇条書き>

Closes #<issue-number>

## Test plan
<テスト方針のチェックリスト>

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

- PR タイトルは 70 文字以内に収めること

3. Issue に Kind/Type ラベルが付与されている場合、同じラベルを PR にも付与する。project-context.md の「Issue/PR ラベルの prefix」を参照して正しい prefix を使用すること。

```bash
# prefix は project-context.md に従って "Kind: " または "Type: " を使用する
LABELS=$(gh issue view <issue-number> --json labels --jq '[.labels[].name | select(startswith("<prefix>"))] | join(",")')
if [ -n "$LABELS" ]; then
  gh pr edit --add-label "$LABELS"
fi
```

- PR の URL をユーザーに返すこと

#### モード B（Markdown 実装）: 引数に依存

**`--branch` なしの場合（新規実装）:**
- Push して PR を作成する（モード A と同様のフロー）

**`--branch` ありの場合（既存ブランチでの修正）:**
- Push のみ行い、完了をユーザーに報告する

## 注意事項

- 入力ソースの内容を正確に理解し、過不足のない実装を行うこと
- 推測ではなく、実際のコードを読んで確認した事実に基づいて実装すること
- 実装中に不明点や判断が必要な事項があればユーザーに確認すること
- ビルドが通らない状態でコミット・Push しないこと
- project-context.md に記載されたフォーマッターを忘れずに実行すること

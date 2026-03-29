# dev-workflow

GitHub Issue を起点にした開発ワークフローを Claude Code のスキルとして提供するプラグイン。
プラン作成 → 実装 → レビュー → 棚卸しまでを一貫して扱える。

## Skills

| スキル | コマンド例 | 概要 |
|--------|-----------|------|
| **setup-project-context** | `/setup-project-context` | リポジトリを自動調査し、project-context.md を生成する |
| **create-plan** | `/create-plan 42` | Issue の内容をもとにコードベースを調査し、実装プランを Issue コメントに投稿する |
| **update-plan** | `/update-plan 42` | Issue コメントや会話のフィードバックを反映してプランを更新する |
| **implement-plan** | `/implement-plan 42` | Issue 上のプランに従いコードを実装し、PR を作成する |
| **fix-review** | `/fix-review 99` | PR の未解決レビューコメントに対応してコードを修正する |
| **review** | `/review 99` | PR の差分をレビューし、インラインコメントを投稿する |
| **triage** | `/triage` | Issue の一括棚卸し（クローズ・ラベル付与・マイルストーン整理・レポート出力） |

## ワークフロー

```
/setup-project-context ── 初回セットアップ（project-context.md 生成）
  │
  ▼
Issue 作成
  │
  ▼
/create-plan #N ──── プランを Issue コメントに投稿
  │
  ├─ フィードバックがあれば ──▶ /update-plan #N
  │
  ▼
/implement-plan #N ─ ブランチ作成 → 実装 → PR 作成
  │
  ▼
/review <PR> ─────── レビューコメント投稿
  │
  ├─ 指摘があれば ──▶ /fix-review <PR> で修正
  │
  ▼
マージ
  │
  ▼
/triage ──────────── 定期的にまとめて棚卸し
```

## 各スキルの詳細

### setup-project-context

```
/setup-project-context
```

- リポジトリのコードベースを自動調査し、`.claude/skills/references/project-context.md` を生成
- 言語・ビルドツール・テスト構成・ディレクトリ構成などを検出して、テンプレートの各セクションを埋める
- 他のスキル（create-plan, implement-plan, review 等）が参照する project-context.md の初期セットアップに使用

### create-plan

```
/create-plan <issue-number>
```

- Issue の Description を読み取り、コードベースを調査して実装プランを作成
- `<!-- claude:plan -->` マーカー付きで Issue コメントに投稿
- プランには概要・設計判断・影響範囲・ファイル構成・実装ステップ・テスト戦略を含む
- GitHub API に失敗した場合は `.claude/outputs/plans/` にローカル出力

### update-plan

```
/update-plan <issue-number>
```

- 既存プランに対する Issue コメントのフィードバックと会話コンテキストを収集
- フィードバックを分類（直接修正 / 確認事項の回答 / 方針変更）して適用
- 変更履歴セクションを追加してバージョン管理

### implement-plan

2 つの入力モードに対応:

```
/implement-plan <issue-number>                    # Issue プランから実装 → PR 作成
/implement-plan <path.md> [--branch <branch>]     # ローカル Markdown から実装
```

- プランの Phase / Step 順に実装
- フォーマッタ実行・ビルド確認後にコミット・プッシュ
- Issue モードでは `Closes #N` 付きで PR を自動作成

### fix-review

```
/fix-review <pr-number>
```

- PR の未解決レビューコメント（インラインコメント）を GraphQL で取得
- 各指摘に対応するコード修正を実施
- 曖昧な指摘や設計判断が必要なものはスキップし、対応結果を報告
- フォーマッタ実行・ビルド確認後にコミット・プッシュ

### review

```
/review <pr-number>                   # PR 差分レビュー（デフォルト）
/review <pr-number> --category code   # カテゴリ指定
/review <pr-number> --full            # 全カテゴリでレビュー
/review --full-codebase               # コードベース全体レビュー（ローカル出力）
```

**レビューカテゴリ:**

| カテゴリ | 対象 |
|---------|------|
| architecture | 設計パターン・モジュール分割・依存関係 |
| code | ロジック・エッジケース・命名・規約 |
| test | カバレッジ・テストケース網羅性 |
| security | 認証認可・バリデーション・機密情報 |
| docs | API ドキュメント・README |
| build | ビルド設定・依存関係・CI |

**重要度:** Critical / High / Medium / Low の 4 段階で指摘

### triage

```
/triage
```

5 ステップを順に実行:

1. **クローズ判定** — 実装済み Issue を理由付きでクローズ
2. **ラベル付与** — Kind / Priority ラベルが未設定の Issue にラベルを付与
3. **Issue 作成** — 直近クローズしたマイルストーンのプラン「今後の展望」から新 Issue を作成
4. **マイルストーン整理** — Kind + Priority に基づいてマイルストーンを平準化
5. **レポート出力** — 棚卸し結果を GitHub Discussions に投稿

## セットアップ

### 前提条件

- Claude Code がインストール済み
- `gh` CLI が認証済み

### プロジェクト固有の設定

各リポジトリに `.claude/skills/references/project-context.md` を配置することで、スキルの動作をプロジェクトに合わせてカスタマイズできる。

`/setup-project-context` を実行すると、リポジトリを自動調査してテンプレートを埋めた `project-context.md` を生成できる。手動で作成する場合は `skills/references/project-context-template.md` を参照。

主な設定項目:

- **コード調査ガイド** — create-plan がコードベースを調査する際の手順
- **実装ガイド** — implement-plan / fix-review が従うビルド・フォーマット・命名規約
- **レビューガイド** — review のカテゴリ別チェックポイントやファイルパスマッピング
- **ラベル・ワークフロー規約** — triage が使用するラベル体系

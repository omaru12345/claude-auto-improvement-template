# claude-auto-improvement-template

**Claude Code × GitHub Actions による自律改善ループのテンプレートです。**

毎週月曜の朝9時に Claude AI が自動でサイトを分析 → 課題を選定 → 実装 → PR作成します。あなたはPRを確認してマージするだけ。

> 🇺🇸 [English README](README.md)

---

## 仕組み

```
毎週月曜 9:00 JST
    │
    ├─ Google Search Console で SEO データ取得
    ├─ AdSense API でページ別収益取得（任意）
    ├─ Web 検索で競合調査
    │
    ├─ 重要インサイトあり？
    │   Yes → GitHub Issue を新規作成
    │   No  → 既存の未対応 Issue を選択
    │
    └─ 実装 → テスト → PR 作成

PR レビュー提出
    └─ Copilot のコメントに自動対応 → push → 返信
```

---

## セットアップ

### 1. このリポジトリをテンプレートとして使用

「Use this template」ボタンから自分のリポジトリを作成してください。

### 2. GitHub Secrets の登録

```bash
# 必須
gh secret set ANTHROPIC_API_KEY --body "sk-ant-..."

# GSC 用（任意）— Google Cloud のサービスアカウント JSON
gh secret set GOOGLE_APPLICATION_CREDENTIALS_JSON --body "$(cat service-account.json)"

# AdSense 用（任意）
gh secret set ADSENSE_CLIENT_ID     --body "..."
gh secret set ADSENSE_CLIENT_SECRET --body "..."
gh secret set ADSENSE_REFRESH_TOKEN --body "..."
gh secret set ADSENSE_PUBLISHER_ID  --body "pub-..."
```

AdSense と GSC は**未設定でも動きます**。その場合は競合 Web 調査のみで改善ループが動きます。

### 3. `.claude/skills/improvement-loop.md` をカスタマイズ

以下の部分を自分のプロジェクトに合わせて編集してください：

- テスト実行コマンド（デフォルト: `ruff check` + `pytest` / `tsc` + `npm build`）
- 競合調査のキーワード
- インサイト判定の基準値

### 4. 動作確認

GitHub → Actions →「Claude - Autonomous Improvement Loop」→「Run workflow」で即時テスト実行できます。

---

## ファイル構成

```
.github/workflows/
├── claude-improvement-loop.yml    # スケジュール実行（月曜 9:00 JST）
└── claude-review-responder.yml    # レビュー自動対応

.claude/skills/
└── improvement-loop.md            # Claude への指示書（思考手順）
```

---

## ⚠️ よくあるハマりポイント

| 症状 | 原因 | 対処 |
|---|---|---|
| ワークフローが途中で止まる（承認待ち） | `--dangerously-skip-permissions` 未設定 | 全 Claude ワークフローの `claude_args` に追加 |
| review-responder が `App token exchange failed: 401` で失敗 | Copilot がレビューを投稿すると `github.actor` が外部ボットになり Anthropic の OIDC 交換が拒否される | `claude-code-action@v1` ではなく Claude CLI 直接呼び出しを使う（本テンプレートでは対応済み） |
| Bot 作成 PR に Copilot がアサインされない | Bot actor は `pull_request` イベントを発火しない | `claude-improvement-loop.yml` 末尾の Copilot アサインステップで対応済み |
| Bot 作成 PR に Claude 自動レビューが通らない | Bot actor はデフォルトでブロック | `allowed_bots: "claude[bot]"` を `claude-review.yml` に追加 |
| Copilot が `id-token: write` 削除を提案 | Copilot の誤指摘 | 無視する（削除すると `claude-code-action@v1` の OIDC 認証が壊れる） |

---

## コスト目安

| 項目 | 目安 |
|---|---|
| Anthropic API | 1回あたり約 $5 |
| 月4回（月曜のみ） | 約 $20/月 |
| GitHub Actions | 無料枠内 |

**オフにする方法**：GitHub Actions タブ → ワークフロー →「Disable workflow」

月・金の週2回にする場合は `claude-improvement-loop.yml` の金曜 cron のコメントを外してください（約 $40/月）。

---

## 詳細解説記事

- [Part 1: 概要・設計編](#) — Qiita
- [Part 2: 実装全公開編](#) — Qiita
- [Part 3: GSC・AdSense API 認証設定編](#) — Qiita

---

## License

MIT

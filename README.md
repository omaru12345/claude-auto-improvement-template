# claude-auto-improvement-template

**Claude Code × GitHub Actions による自律改善ループのテンプレートです。**

毎週月・金の朝9時に Claude AI が自動でサイトを分析 → 課題を選定 → 実装 → PR作成します。あなたはPRを確認してマージするだけ。

---

## 仕組み

```
毎週月・金 9:00 JST
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

# GSC 用（任意）
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
- インサイント判定の基準

### 4. 動作確認

GitHub → Actions → 「Claude - 自律改善ループ」→「Run workflow」で即時テスト実行できます。

---

## ファイル構成

```
.github/workflows/
├── claude-improvement-loop.yml    # スケジュール実行（月・金 9:00 JST）
└── claude-review-responder.yml    # レビュー自動対応

.claude/skills/
└── improvement-loop.md            # Claude への指示書（思考手順）
```

---

## コスト目安

| 項目 | 目安 |
|---|---|
| Anthropic API | 1回あたり数ドル |
| 月8回（月・金 × 4週）| 月10〜30ドル前後 |
| GitHub Actions | 無料枠内 |

**オフにする方法**：GitHub Actions タブ → ワークフロー →「Disable workflow」

---

## 詳細解説記事

- [Part 1: 概要・設計編](#) — Qiita
- [Part 2: 実装全公開編](#) — Qiita
- [Part 3: GSC・AdSense API 認証設定編](#) — Qiita

---

## License

MIT

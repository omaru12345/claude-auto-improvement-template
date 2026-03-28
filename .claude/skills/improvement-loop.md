---
name: improvement-loop
description: AI完結の自律改善ループ。GSC/AdSense/競合調査→インサイト判定→課題選択→実装→PR作成まで一通り実施する。月曜・金曜 9:00 JST に GitHub Actions から呼ばれる。
type: workflow
---

# 自律改善ループ 実施手順

## 0. 初期確認

```bash
# main ブランチ最新化
git checkout main && git pull origin main

# 既存のオープン PR を確認（コンフリクト回避）
gh pr list --state open --json number,title,headRefName,files
```

オープン PR がある場合、それらが触っているファイルには**絶対に手を出さない**。

---

## 1. データ収集

### 1-1. GSC（Google Search Console）

```bash
# 認証セットアップ
if [ -n "$GOOGLE_APPLICATION_CREDENTIALS_JSON" ]; then
  printf '%s' "$GOOGLE_APPLICATION_CREDENTIALS_JSON" > /tmp/gac.json
  chmod 600 /tmp/gac.json
  export GOOGLE_APPLICATION_CREDENTIALS=/tmp/gac.json
fi

# GSCデータ取得スクリプトを実行
pip install -q google-auth google-api-python-client

python3 - <<'EOF'
import json, sys, os
sys.path.insert(0, 'agent')
from services.gsc_service import GSCService

try:
    svc = GSCService()
    top_queries = svc.get_top_queries(days=28, row_limit=20)
    low_ctr    = svc.get_low_ctr_pages(days=28, ctr_threshold=0.03, min_impressions=10)
    print("=== Top Queries ===")
    print(json.dumps(top_queries, ensure_ascii=False, indent=2))
    print("=== Low CTR Pages ===")
    print(json.dumps(low_ctr, ensure_ascii=False, indent=2))
except Exception as e:
    print(f"[GSC SKIP] {e}", file=sys.stderr)
EOF
```

### 1-2. AdSense（認証情報がある場合のみ）

```bash
# 認証情報が無ければスキップ
if [ -z "$ADSENSE_REFRESH_TOKEN" ]; then
  echo "[AdSense SKIP] 認証情報なし"
else
  # アクセストークン取得
  TOKEN_RESPONSE=$(curl -s -X POST https://oauth2.googleapis.com/token \
    -d "client_id=${ADSENSE_CLIENT_ID}" \
    -d "client_secret=${ADSENSE_CLIENT_SECRET}" \
    -d "refresh_token=${ADSENSE_REFRESH_TOKEN}" \
    -d "grant_type=refresh_token")
  ACCESS_TOKEN=$(echo "$TOKEN_RESPONSE" | python3 -c "import sys,json; print(json.load(sys.stdin).get('access_token',''))" 2>/dev/null)
  if [ -z "$ACCESS_TOKEN" ]; then
    echo "[AdSense SKIP] トークン取得失敗: $TOKEN_RESPONSE"
    exit 0
  fi

  # AdSense アカウント一覧取得（pub-ID → account name に変換）
  ACCOUNT_NAME="accounts/${ADSENSE_PUBLISHER_ID}"

  # 過去28日のページ別レポート（収益・RPM・クリック率）
  TODAY=$(date +%Y-%m-%d)
  START=$(date -d "28 days ago" +%Y-%m-%d 2>/dev/null || date -v-28d +%Y-%m-%d)
  curl -s "https://adsense.googleapis.com/v2/${ACCOUNT_NAME}/reports:generate?dateRange=CUSTOM&startDate.year=${START:0:4}&startDate.month=${START:5:2}&startDate.day=${START:8:2}&endDate.year=${TODAY:0:4}&endDate.month=${TODAY:5:2}&endDate.day=${TODAY:8:2}&dimensions=URL_CHANNEL&metrics=PAGE_VIEWS&metrics=CLICKS&metrics=COST_PER_CLICK&metrics=AD_REQUESTS_CTR&metrics=ESTIMATED_EARNINGS&orderBy=-ESTIMATED_EARNINGS&limit=20" \
    -H "Authorization: Bearer ${ACCESS_TOKEN}" \
    | python3 -m json.tool
fi
```

### 1-3. 競合調査（Web検索）

以下のクエリで Web 検索を実施し、競合サービスの新機能・人気コンテンツを調査する：

- `"小売 DX AIエージェント SNS 事例共有" site:*.jp OR site:*.com 2024 OR 2025`
- `"飲食 AI 自動化 業務改善 事例" 人気 ランキング 2025`
- `retail tech AI case sharing platform features 2025`
- `食品業界 IT導入事例 コミュニティ サービス 比較`

**調査ポイント:**
- 競合が持っているがこちらにない機能（フィルタ・評価・比較表など）
- UIパターンで多く採用されているもの
- ユーザーが求めているコンテンツカテゴリ

---

## 2. インサイト判定

収集したデータを以下の基準で評価し、**重要インサイトの有無**を判定する。

### 重要インサイトと判定する基準（1つ以上該当すれば作成）

| データソース | 基準 | 改善方向 |
|---|---|---|
| GSC | 表示100回以上かつCTR < 1% のページ | タイトル・メタディスクリプション改善 |
| GSC | 順位4〜10位（2ページ目）のクエリ | 構造化データ・コンテンツ拡充 |
| AdSense | 特定ページのRPMが全体平均の半分以下 | 広告レイアウト・コンテンツ品質改善 |
| 競合調査 | 競合が提供しているが自サイトにない主要機能 | 機能追加 |
| 競合調査 | 新トレンドキーワードが複数サービスで確認 | コンテンツ・エージェント対応 |

### 重要インサイトなしと判定する場合

ISSUES.md を読み込み、`[未対応]` の課題の中から以下の条件で1つ選ぶ：

```bash
cat ISSUES.md | grep -A 5 "\[未対応\]"
```

選定条件：
- **オープン PR のファイルと重複しない**
- フロントエンドとバックエンドを交互に選ぶ（直前のPRと別レイヤー）
- 複雑度が低い（1〜2時間で完了できそうなもの）

---

## 3. 課題作成（重要インサイトあり の場合）

`.claude/skills/issue-parent-child.md` に従い、GitHub Issue と ISSUES.md を更新する。

```bash
# ラベル確認・作成
gh label create "auto-improvement" --color "0075ca" --description "自律改善ループによる自動PR" 2>/dev/null || true

# Issue 作成
gh issue create \
  --title "[自律改善] {タイトル}" \
  --body "{インサイトの根拠・改善内容・期待効果を記載}" \
  --label "auto-improvement"
```

---

## 4. 実装

### 4-1. ブランチ作成

```bash
DATE=$(date +%Y%m%d)
BRANCH="feature/auto-improvement-${DATE}"
git checkout -b "${BRANCH}"
```

### 4-2. コード変更

選択した課題に基づき実装する。**必ず守ること：**

- CLAUDE.md のルールを遵守（any型禁止・App Router使用・DDDレイヤー・型安全性）
- 変更は最小限にとどめる（課題スコープ外を触らない）
- オープン PR が触っているファイルには絶対に手を出さない

### 4-3. テスト実行

```bash
# Backend が変更された場合
cd backend
source venv/bin/activate 2>/dev/null || pip install -q -r requirements.txt
ruff check app/ && pytest tests/ -v
cd ..

# Frontend が変更された場合
cd frontend
npm ci --silent
npx tsc --noEmit && npm run build
cd ..
```

テストが通らない場合は修正してから次のステップへ進む。

---

## 5. PR 作成

```bash
# コミット
git add -A
git commit -m "feat: {変更内容の要約}

自律改善ループによる自動実装。
インサイト: {根拠を1行で}

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>"

# プッシュ
git push origin "${BRANCH}"

# PR 作成（auto-improvement ラベル付き）
gh pr create \
  --title "feat: {変更内容の要約}" \
  --label "auto-improvement" \
  --body "$(cat <<'EOF'
## 変更概要

{変更内容を箇条書きで}

## インサイト根拠

| ソース | 指標 | 値 |
|---|---|---|
| {GSC/AdSense/競合調査} | {指標名} | {値} |

## 関連 Issue
Closes #{Issue番号 or なし}

## 備考

自律改善ループ（月曜・金曜 9:00 JST）による自動実装。
EOF
)"
```

---

## 6. 完了報告

実施内容を標準出力にまとめて出力する：

```
=== 改善ループ完了 ===
実施日時: {日時}
インサイト: {あり/なし}
対応課題: {課題タイトル}
PR: {PR URL}
変更ファイル: {ファイル一覧}
```

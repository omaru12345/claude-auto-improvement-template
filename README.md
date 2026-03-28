# claude-auto-improvement-template

**A template for autonomous improvement loops using Claude Code × GitHub Actions.**

Every Monday at 9:00 AM JST, Claude AI automatically analyzes your site → selects issues → implements fixes → creates PRs. You just review and merge.

> 🇯🇵 [日本語版 README はこちら](README.ja.md)

---

## How it works

```
Every Monday 9:00 AM JST
    │
    ├─ Fetch SEO data from Google Search Console
    ├─ Fetch page-level revenue from AdSense API (optional)
    ├─ Web search for competitor research
    │
    ├─ Any high/medium priority insights?
    │   Yes → Create new GitHub Issues
    │   No  → Pick from existing open Issues
    │
    └─ Implement → Test → Create PR

PR review submitted
    └─ Auto-respond to Copilot comments → push fixes → reply
```

---

## Setup

### 1. Use this repository as a template

Click the **"Use this template"** button to create your own repository.

### 2. Register GitHub Secrets

```bash
# Required
gh secret set ANTHROPIC_API_KEY --body "sk-ant-..."

# For GSC (optional) — service account JSON from Google Cloud
gh secret set GOOGLE_APPLICATION_CREDENTIALS_JSON --body "$(cat service-account.json)"

# For AdSense (optional)
gh secret set ADSENSE_CLIENT_ID     --body "..."
gh secret set ADSENSE_CLIENT_SECRET --body "..."
gh secret set ADSENSE_REFRESH_TOKEN --body "..."
gh secret set ADSENSE_PUBLISHER_ID  --body "pub-..."
```

AdSense and GSC are **optional** — without them, the loop runs using competitor web research only.

### 3. Customize `.claude/skills/improvement-loop.md`

Edit the following to match your project:

- Test commands (default: `ruff check` + `pytest` / `tsc` + `npm build`)
- Competitor research keywords
- Insight priority thresholds

### 4. Verify

Go to **GitHub → Actions → "Claude - Autonomous Improvement Loop" → "Run workflow"** to trigger a test run immediately.

---

## File structure

```
.github/workflows/
├── claude-improvement-loop.yml    # Scheduled run (Monday 9:00 AM JST)
└── claude-review-responder.yml    # Auto-respond to PR reviews

.claude/skills/
└── improvement-loop.md            # Claude's instruction manual (reasoning steps)
```

---

## ⚠️ Common pitfalls

| Symptom | Cause | Fix |
|---|---|---|
| Workflow hangs waiting for approval | `--dangerously-skip-permissions` not set | Add to `claude_args` in all Claude workflows |
| `App token exchange failed: 401` on review responder | When Copilot submits a review, `github.actor` becomes an external bot — Anthropic's OIDC exchange rejects it | Use Claude CLI directly instead of `claude-code-action@v1` (already done in this template) |
| Copilot not auto-assigned to bot-created PRs | Bot actor doesn't fire `pull_request` events | Copilot assign step is included at the end of `claude-improvement-loop.yml` |
| Bot-created PRs blocked from Claude review | Bot actors are blocked by default | Add `allowed_bots: "claude[bot]"` to `claude-review.yml` |
| Copilot suggests removing `id-token: write` | Incorrect suggestion | Ignore it — removing it breaks `claude-code-action@v1`'s OIDC auth |

---

## Cost estimate

| Item | Estimate |
|---|---|
| Anthropic API | ~$5 per run |
| Monthly (4 Mondays) | ~$20/month |
| GitHub Actions | Within free tier |

**To disable**: GitHub Actions tab → workflow → "Disable workflow"

To run on both Monday and Friday, uncomment the Friday cron in `claude-improvement-loop.yml` (~$40/month).

---

## Articles

- [Part 1: Overview & Design](#) — Qiita
- [Part 2: Full Implementation](#) — Qiita
- [Part 3: GSC & AdSense API Auth Setup](#) — Qiita

---

## License

MIT

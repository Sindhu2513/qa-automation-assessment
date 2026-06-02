# QA & Automation Developer — Take-Home Assessment

**Candidate:** Paladi Sindhu  
**Submission Date:** 01 June 2026  
**Role:** Automation & QA Developer

---

## Repository Structure

```
qa-automation-assessment/
├── docs/
│   ├── Task1_QA_Report_Paladi_Sindhu.pdf        # Task 1 — QA bug report
│   └── Task2_README_Paladi_Sindhu.pdf           # Task 2 + Bonus — workflow documentation
├── n8n-workflows/
│   └── Task2_Workflow_Paladi_Sindhu_json.json   # Both workflows in one exported file
├── README.md                                    # This file
└── LOOM_VIDEO.md                                # Loom walkthrough link
```

> The n8n export contains two independent workflow chains in a single JSON file — the CryptoAlert digest (Task 2) and the Uptime Monitor (Bonus Task). Both can be imported together via **Workflows → Import from file** in any n8n instance.

---

## Task 1 — QA Bug Report

**App tested:** [https://demo.realworld.io](https://demo.realworld.io)  
**Total issues found:** 12 — 2 Critical · 4 High · 4 Medium · 2 Low  
**Full report:** `docs/Task1_QA_Report_Paladi_Sindhu.pdf`

I tested the app manually across six main user flows and seven quality dimensions. Testing was entirely exploratory — no scanners, just DevTools, the Network tab, and edge-case inputs.

**Flows covered:** Sign Up · Login · Create Article · Edit Article · Delete Article · Logout  
**Dimensions checked:** Security · Form Validation · Session Management · Accessibility · Responsive Design · Performance · Error Handling

### Key Issues Found

| ID | Severity | Title |
|----|----------|-------|
| BUG-001 | Critical | No rate limiting on login — unlimited brute-force attempts possible |
| BUG-002 | Critical | JWT stored in localStorage — readable by any JavaScript (XSS risk) |
| BUG-012 | High | Logout does not invalidate the JWT on the server side |
| BUG-003 | High | Sign-up accepts passwords as short as 1 character |
| BUG-006 | High | No CSRF protection on state-changing API endpoints |
| BUG-004 | High | Article publishes successfully with an empty body field |

The root-cause analysis in the report focuses on BUG-001 (no rate limiting) — I picked it because it is the most immediately exploitable issue and the fix is well-defined. The full table with steps to reproduce, expected vs actual, and remediation priority for all 12 issues is in the PDF.

---

## Task 2 — n8n CryptoAlert Workflow

**File:** `n8n-workflows/Task2_Workflow_Paladi_Sindhu_json.json`  
**Trigger:** Schedule (every 1 hour)  
**APIs used:** CoinGecko `/coins/markets` + `/coins/{id}` (no API key needed)  
**Output:** Telegram message — bullish alert or normal digest depending on 24h price change

### What the workflow does

```
Schedule (1 hr)
  → CoinGecko: fetch top 10 coins by market cap
  → Code node: sort and keep only top 5, build summary string
  → Filter: pass only rank #1 coin to enrichment step
  → CoinGecko: fetch full detail for #1 coin (ATH, 24h high/low, description)
  → Code node: flatten nested market_data into clean fields
  → IF node: 24h change > 5%?
      TRUE  → Build Bullish Message → Telegram (rocket emoji alert)
      FALSE → Build Normal Message  → Telegram (quiet digest)
  → Error Trigger (separate) → Format Error → Telegram (failure notification)
```

**Why 5% as the threshold?** A 5% single-day move in a top-10 coin is a meaningful signal — 1–2% is just noise. I would expose this as a workflow-level variable in production so it can be tuned without touching code.

**Why CoinGecko?** No API key required on the free tier (~50 req/min), covers both required API calls from the same base URL, and the data is clean enough to use directly without transformation gymnastics.

**Why Telegram?** Free, no OAuth flow, instant delivery, works on mobile. The bot token and chat ID are stored as a named Telegram credential in n8n — nothing is hard-coded in the workflow JSON.

### Quick setup

1. Import `Task2_Workflow_Paladi_Sindhu_json.json` in n8n — both workflows load together
2. Go to **Settings → Credentials → New → Telegram** and add your bot token + chat ID
3. Activate the workflow — it fires automatically every hour
4. CoinGecko needs no credential setup

---

## Bonus Task — Uptime Monitor

**File:** same JSON as above (second workflow chain)  
**Trigger:** Schedule (every 5 minutes) + separate daily trigger at 08:00 UTC  
**Monitors:** [https://demo.realworld.io](https://demo.realworld.io) — the same app from Task 1  
**Output:** Telegram downtime alert + Google Sheets log row per check + daily summary digest

### What the workflow does

```
Schedule (5 min)
  → Init context (URL, timestamp)
  → HTTP Attempt 1 → Evaluate (status code, response time)
      IF non-200 → Wait 10s → HTTP Attempt 2 → Evaluate
          IF non-200 → Wait 15s → HTTP Attempt 3 → Evaluate
              → Decide Alert (is_down = true only if all 3 attempts failed)
              → IF Down → Telegram Downtime Alert
              → Google Sheets: append log row (always)

Schedule (08:00 UTC daily)
  → Read last 24h from Google Sheets
  → Build summary (uptime %, total checks, incidents, avg response time)
  → Telegram: send daily brief
```

Three attempts with progressive waits (10s then 15s) mean a single dropped packet never triggers a false alert. The daily summary at 08:00 UTC gives the team a morning-brief view without needing to open the sheet.

### Google Sheets setup

1. Create a sheet named **UptimeLog**
2. Row 1 headers: `Date | Time_UTC | URL | Status_Code | Is_Up | Attempts_Used | Response_Time_ms | Notes`
3. Connect a Google Sheets OAuth2 credential in n8n and paste your Sheet ID

---

## Error Handling — Both Workflows

Every HTTP Request node has `continueOnFail: true` so a timeout or bad response is passed through as a failed item rather than crashing the run. On top of that, an Error Trigger node catches anything that slips through and sends a formatted Telegram message with the error name, message, and offending node. Failures are never silent.

---

## Submission Checklist

- [x] Task 1 — QA report PDF with 12 bugs, full table, and root-cause analysis
- [x] Task 2 — n8n workflow JSON (CryptoAlert) with error handling and no hard-coded credentials
- [x] Bonus — Uptime Monitor workflow with retry logic, Sheets logging, and daily summary
- [x] README with setup instructions and design rationale
- [x] Loom video walkthrough (see `LOOM_VIDEO.md`)

---

*Submitted by Paladi Sindhu — Automation & QA Developer Assessment*

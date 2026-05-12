# AI Outreach Machine

Production-grade automated B2B cold outreach system built on n8n. Finds leads, researches them, writes personalized emails in 9 languages, sends with proper rate limits, classifies replies via Claude AI, and alerts hot leads to Telegram — all without human intervention.

**Status:** ✅ Production. 7 workflows running 24/7. Processed 469+ emails in pilot.

---

## What It Does

A complete cold outreach pipeline that replaces 15-25 hours of manual sales work per week:

1. **Scout** — finds prospects online (real estate agencies, law firms, dental clinics, etc.)
2. **Researcher** — visits each website, identifies pain points using Claude
3. **Copywriter** — generates personalized emails in the prospect's language
4. **Sender** — sends 3-5 per hour with A/B subject testing, respects Gmail limits
5. **Tracker** — reads IMAP, classifies replies via Claude (positive/negative/out_of_office/etc.), Telegram alerts on hot leads
6. **Notifier** — weekly reports every Monday 9am with conversion metrics
7. **Follow-up** — sends second email 5 days later to non-responders, picks strategy via Claude (bump/value-add/soft-close)

---

## Architecture

```
┌────────────────────────────────────────────────────────────────────────┐
│  OUTBOUND PIPELINE (daily)                                             │
│                                                                        │
│  Scout → Researcher → Copywriter → Sender                              │
│   ↓        ↓             ↓           ↓                                 │
│  Leads  Outreach    Outreach     Outreach                              │
│  sheet  (drafts)    (ready)      (sent)                                │
└────────────────────────────────────────────────────────────────────────┘
                                ↓
┌────────────────────────────────────────────────────────────────────────┐
│  INBOUND PIPELINE (every 15 minutes)                                   │
│                                                                        │
│  Gmail IMAP → Tracker → Claude classify → Telegram alert (hot leads)   │
│                  ↓                                                     │
│             Replies sheet + Suppression list                           │
└────────────────────────────────────────────────────────────────────────┘
                                ↓
┌────────────────────────────────────────────────────────────────────────┐
│  SUPPORT WORKFLOWS                                                     │
│                                                                        │
│  Notifier — every Monday 9:00 → weekly Telegram report                 │
│  Follow-up — every day 14:00 → second email to non-responders          │
└────────────────────────────────────────────────────────────────────────┘
```

**Single source of truth:** one Google Sheet with 4 tabs (Leads, Outreach, Replies, Suppression, Reports) shared across all 7 workflows.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Orchestration | [n8n](https://n8n.io/) (self-hosted, PM2 supervised) |
| AI generation/classification | Claude Sonnet 4.5 via Anthropic API |
| Data store | Google Sheets (Service Account) |
| Email send | Gmail SMTP |
| Email receive | Gmail IMAP (push, not poll) |
| Notifications | Telegram Bot API (3 bots — alerts/reports/follow-up) |
| Host | Ubuntu 24 VPS |

**Why this stack:**
- **n8n** instead of code: faster iteration, visual debugging, easy handoff to non-devs
- **Sheets** instead of database: client-readable, no separate UI needed, version history built-in
- **Claude** for both writing and classification: same model handles both, no fine-tuning needed
- **Telegram** instead of email/Slack alerts: founder gets hot leads on phone in seconds

---

## Workflows

### `worker-1-scout.json` — Lead Discovery

Scrapes target companies from public directories. Outputs to `Leads` sheet with: `company_name, website, email, city, country, language, source, score, status`.

Robust parsing handles malformed HTML, deduplicates by domain, scores leads by signals (has chat widget, has contact form, page speed).

**Schedule:** Daily 09:00.

### `worker-2-researcher.json` — Company Research

Visits each lead's website, extracts pain signals (no chat, no contact form, slow load, Realworks template, etc.), enriches the row with `pain` and `notes` columns.

Uses Claude Sonnet to read the site HTML and identify 1-2 specific issues for personalization.

**Schedule:** Daily 11:00.

### `worker-3-copywriter.json` — Email Generation

Generates personalized subject A/B and body for each lead. Claude reads `pain` + company info + language code, produces email in target language (Dutch, Polish, Czech, German, English, etc.).

Deduplicates against existing Outreach rows so no lead gets two drafts.

**Schedule:** Daily 13:00.

### `worker-4-sender.json` — Email Delivery

Sends drafts via Gmail SMTP. Slow trickle: max 3-5 emails per hour to avoid spam flags. Picks subject A or B randomly per email for A/B testing. Marks status to `sent` and stores variant + timestamp for analysis.

**Schedule:** Daily 15:00, runs ~3-4 hours.

### `worker-5-tracker.json` — Reply Handler (the most complex one)

IMAP IDLE on inbox → on new email → filter (whitelist check, hard blacklist for noreply/mailer-daemon) → match against Outreach by `from_email` → dedupe by `imap_uid` → loop:

1. Build Claude classification prompt with email body
2. Wait 2s (rate limit)
3. Call Claude → returns category (positive/negative/out_of_office/neutral/referral/unsubscribe), confidence, reason, key_quote
4. Append to Replies sheet
5. Update Outreach status to `replied`
6. If category is `positive` or `referral` → format HTML Telegram alert with company info, city flag, AI analysis, key quote, action line → send
7. If category is `unsubscribe` → add to Suppression sheet

17 nodes. Took 5 iterations (v2.5 → v2.9) to handle edge cases: SplitInBatches loop closure, strict IF type validation breaking on undefined fields, data loss after Google Sheets update node.

**Trigger:** IMAP push (real-time, <60s reaction).

### `worker-6-notifier.json` — Weekly Reports

Every Monday 9am: reads all Outreach + Replies, filters to last 7 days, aggregates 17 metrics (sent, replied, conversion%, breakdown by category, top cities with flags, top domains, best subject by reply rate, average response hours, hot leads list). Formats HTML Telegram message + appends row to Reports sheet for historical trend.

**Schedule:** Monday 09:00 weekly.

### `worker-7-followup.json` — Follow-up Sender

Daily at 14:00: finds Outreach rows where `status=sent` + `generated_at > 5 days ago` + no reply in Replies + not in Suppression + no prior followup. For each, asks Claude to pick a strategy (bump = soft reminder, value-add = add a stat, soft-close = polite cycle close) and generate body in original language. Sends with `Re: <original subject>`, updates row with `followup_at` + `followup_status`.

**Schedule:** Daily 14:00.

---

## Pilot Metrics

10-day pilot on Dutch real estate agencies (April 2026):

| Metric | Result |
|---|---|
| Leads discovered | 1,156 |
| Emails sent | 469 |
| Replies received | 9 |
| Hot leads (positive) | 1 |
| Operator hours | 0 |
| Reply rate | 1.9% (during NL public holiday week) |

Reply rate during regular weeks typically 2-3× higher per industry benchmarks. Pilot deliberately ran across King's Day holiday for realistic worst-case.

---

## Setup

Detailed steps in [`SETUP.md`](SETUP.md). Quick version:

1. **VPS with n8n** running. Tested on Ubuntu 24 with n8n under PM2.
2. **Google Sheet** with 5 tabs: `Leads`, `Outreach`, `Replies`, `Suppression`, `Reports`. Column schemas in [`SETUP.md`](SETUP.md).
3. **Service Account** for Google Sheets, JSON credential imported to n8n.
4. **Anthropic API key** added as n8n HTTP Header Auth credential.
5. **Gmail account** with App Password, SMTP + IMAP credentials in n8n.
6. **3 Telegram bots** via @BotFather (alerts, reports, follow-up), tokens as Telegram API credentials.
7. **Import 7 workflow JSONs** from `workflows/`. Replace credential placeholders. Activate triggers.

---

## What's Inside Each Workflow

Each JSON is heavily commented through node names. Read them in this order to understand the system:

1. `worker-1-scout.json` — start here, simplest
2. `worker-3-copywriter.json` — see how Claude integration works
3. `worker-4-sender.json` — rate limiting pattern with SplitInBatches + Wait
4. `worker-5-tracker.json` — most complex, real production patterns (loop closure, dedup, multi-branch routing)
5. `worker-6-notifier.json` — pure data aggregation example
6. `worker-7-followup.json` — Claude strategy selection + email threading

Each Code node uses `$('Node Name').item.json.field` for cross-node data access instead of `$json` — important pattern when nodes mutate the data flow (e.g., Google Sheets Update returns only updated fields).

---

## Lessons Learned

A few things that took multiple iterations to get right:

**1. n8n IF nodes break on `undefined` with strict type validation.** Set `typeValidation: 'loose'` for any condition that might compare against missing fields. Otherwise everything routes to false branch silently.

**2. SplitInBatches loops need explicit closure.** Every branch downstream must connect back to the SplitInBatches `loop` output, or items dangle and the workflow stalls.

**3. Google Sheets Update returns only updated columns.** If you `update` a row and downstream nodes need other columns from that row, read them via `$('Previous Node').item.json.field` rather than `$json`.

**4. Gmail SMTP rate matters more than total volume.** Sending 100 emails in 1 hour gets flagged. Sending 100 emails over 20 hours doesn't. Wait 12s between emails in production.

**5. IMAP `Mark as read` action is mandatory if polling.** Otherwise the next poll re-processes everything in INBOX. Combined with dedup by `imap_uid` in matching logic — bulletproof against duplicates.

**6. Claude returns JSON wrapped in markdown sometimes.** Parse with `string.match(/```json\s*([\s\S]*?)\s*```/)` fallback, then `.indexOf('{')` and `.lastIndexOf('}')` for clean extraction.

---

## File Structure

```
.
├── README.md                      # this file
├── SETUP.md                       # detailed setup instructions
├── LICENSE                        # MIT
├── .gitignore
└── workflows/
    ├── worker-1-scout.json
    ├── worker-2-researcher.json
    ├── worker-3-copywriter.json
    ├── worker-4-sender.json
    ├── worker-5-tracker.json
    ├── worker-6-notifier.json
    └── worker-7-followup.json
```

---

## License

MIT — fork, modify, deploy commercially. Attribution appreciated but not required.

---

## Contact

If you want to use this for your business or hire me to set it up for your niche — open an issue or reach out.

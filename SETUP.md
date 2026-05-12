# Setup Guide

Step-by-step instructions to deploy the AI Outreach Machine from scratch.

Estimated time: **2-4 hours** for first-time setup, ~30 minutes if you already have n8n running.

---

## Prerequisites

- A VPS or local machine running n8n (self-hosted, version 1.50+)
- A Google account with access to Google Sheets
- A Gmail account dedicated to outreach (separate from personal)
- An Anthropic API account with credits
- A Telegram account

---

## Step 1: Prepare the Google Sheet

Create a new Google Sheet titled `AI Outreach Machine`. Add these 5 tabs with exact column names:

### Tab 1: `Leads` (14 columns)

```
company_name | website | email | city | country | language | source | source_url | date_added | score | pain | size | status | notes
```

### Tab 2: `Outreach` (16 columns)

```
website | email | company_name | city | country | language | score | subject_a | subject_b | preheader | body | generated_at | date_generated | status | error | message_id
```

For follow-up support add 3 more columns at the end:

```
... | followup_at | followup_status | followup_message_id
```

### Tab 3: `Replies` (11 columns)

```
received_at | from_email | from_name | outreach_website | reply_subject | reply_body | category | confidence | reason | key_quote | imap_uid
```

### Tab 4: `Suppression` (3 columns)

```
email | reason | added_at
```

Add one placeholder row to avoid empty-sheet edge case:

```
dummy_unsubscribe@example.com | placeholder | 2026-01-01T00:00:00Z
```

### Tab 5: `Reports` (17 columns)

```
report_date | period_start | period_end | sent_count | replied_count | conversion_rate | positive | negative | out_of_office | neutral | referral | unsubscribe | top_city | top_domain | best_subject | avg_response_hours | hot_leads_json
```

**Copy the Sheet ID** from the URL (`docs.google.com/spreadsheets/d/{THIS_PART}/edit`). You'll need it for credential setup.

---

## Step 2: Create Google Service Account

1. Go to [Google Cloud Console](https://console.cloud.google.com/) → create new project (or use existing)
2. Enable **Google Sheets API** and **Google Drive API**
3. Create Service Account → grant role `Editor`
4. Generate JSON key, download it
5. Share your Google Sheet with the service account email (gives it edit access)

---

## Step 3: Set Up Anthropic API

1. Go to [console.anthropic.com](https://console.anthropic.com/)
2. Add billing, get an API key
3. Save the key — you'll add it as HTTP Header Auth in n8n

Pricing for this system: roughly **$0.001-0.003 per email generated**. 1000 emails ≈ $1-3.

---

## Step 4: Gmail Setup

You need **one dedicated Gmail account** for outreach (separate from personal).

1. Create new Gmail or use existing
2. Enable 2-Factor Authentication
3. Go to [Google Account → Security → App Passwords](https://myaccount.google.com/apppasswords)
4. Generate an App Password for "Mail" — this is what you'll use for SMTP/IMAP in n8n (NOT your Google password)
5. Save the 16-character app password

---

## Step 5: Create 3 Telegram Bots

You need three separate bots so different notifications don't mix.

For each bot:

1. Open Telegram, find `@BotFather`
2. Send `/newbot`
3. Set name and username (must end with `_bot`)
4. Copy the token

Create these three:

| Bot Name | Purpose | Used By |
|---|---|---|
| `Outreach Alerts` | Hot lead notifications | Worker 5 — Tracker |
| `Outreach Reports` | Weekly summaries | Worker 6 — Notifier |
| `Outreach Followup` | Follow-up daily summaries | Worker 7 — Follow-up |

**Important:** After creating each bot, find it in Telegram and press `/start`. Otherwise it can't send you messages.

Get your Telegram chat ID:

1. Send a message to `@userinfobot`
2. It replies with your numeric chat ID. Save it.

---

## Step 6: Configure n8n Credentials

In your n8n instance, go to **Credentials** → **Create new** for each:

### Credential 1: Google Sheets (Service Account)

- Type: `Google Service Account API`
- Name: `Google Sheets account` (any name)
- Service Account Email: from JSON file (`client_email`)
- Private Key: from JSON file (`private_key`, paste entire string including `-----BEGIN PRIVATE KEY-----`)

### Credential 2: Anthropic API

- Type: `HTTP Header Auth`
- Name: `Anthropic API`
- Header Name: `x-api-key`
- Header Value: your Anthropic API key

### Credential 3: Gmail SMTP (outbound)

- Type: `SMTP`
- Name: `Gmail SMTP Outreach`
- Host: `smtp.gmail.com`
- Port: `465`
- Secure: `True`
- User: your outreach Gmail address
- Password: the **App Password** from Step 4

### Credential 4: Gmail IMAP (inbound)

- Type: `IMAP`
- Name: `Gmail IMAP Outreach`
- Host: `imap.gmail.com`
- Port: `993`
- Secure: `True`
- User: same Gmail address
- Password: same App Password

### Credentials 5-7: Telegram Bots

For each of the 3 bots, create a separate credential:

- Type: `Telegram API`
- Name: e.g. `Telegram Outreach Bot`, `Telegram Reports Bot`, `Telegram Followup Bot`
- Access Token: the bot token from BotFather

---

## Step 7: Import Workflows

For each JSON in `workflows/`:

1. n8n → **Workflows** → **Import from File**
2. Select the JSON
3. After import, click on the workflow to open it

You'll see all credential references show as broken (red icons). This is expected — placeholders need to be replaced.

For each node with a credential:

1. Click the node
2. In the **Credentials** dropdown, select your actual credential from Step 6
3. Save

Also update these hardcoded values in nodes:

- Google Sheets nodes → **Document** field → replace `YOUR_GOOGLE_SHEET_ID` with your actual sheet ID
- Telegram nodes → **Chat ID** field → replace `YOUR_TELEGRAM_CHAT_ID` with your chat ID
- Gmail SMTP nodes → **From Email** → set to your outreach Gmail
- Code nodes referencing emails → search for `your-outreach-email@gmail.com` placeholder and replace

---

## Step 8: Adapt Workflows to Your Niche

The workflows are pre-tuned for **real estate agencies in Europe**. To use for a different niche:

### Worker 1 — Scout
Update the source URLs and parsing logic in Code nodes to match your target directories.

### Worker 2 — Researcher
The pain signals are real-estate specific (chat widget, contact form). Adjust the Claude prompt in `Analyze Site` node to match what matters for your niche.

### Worker 3 — Copywriter
Most important change. Open the `Build Copy Prompt` node and rewrite the system prompt for your industry. Keep the structure (hook → pain → solution → CTA) but change vocabulary.

### Worker 5 — Tracker
The classification categories work for any B2B niche, no changes needed.

### Worker 6 — Notifier
Reporting works for any niche, no changes needed.

### Worker 7 — Follow-up
Follow-up strategies (bump/value-add/soft-close) work universally. Just verify the language list matches your target market.

---

## Step 9: Test Each Workflow Before Activating

**Do not activate all workflows immediately.** Test in this order:

### Test 1: Scout
- Add a few seed URLs to test parsing
- Run manually via Execute Workflow
- Verify Leads sheet fills correctly

### Test 2: Researcher
- Use leads from Step 1
- Run manually
- Verify `pain` and `notes` columns populate

### Test 3: Copywriter
- Run manually
- Check Outreach sheet — subjects and bodies should be in correct languages
- **Read 3-5 generated emails as if you were the recipient.** If they sound robotic or weird, refine the prompt.

### Test 4: Sender (CAREFUL!)
- Add ONE row to Outreach with **your own personal email** as the test recipient
- Run manually, just for that one row
- Verify email arrives, looks correct
- Only then add real leads

### Test 5: Tracker
- Reply to the test email from another account
- Wait for IMAP poll (max 60 seconds)
- Verify Replies sheet gets new row + Telegram alert (if positive)

### Test 6: Notifier
- Run manually any day
- Verify Telegram report + Reports sheet entry

### Test 7: Follow-up
- For first run, edit Code node `Filter Candidates` and add `return candidates.slice(0, 3);` at end
- Run manually, verify 3 follow-ups sent correctly
- Remove the slice, then activate

---

## Step 10: Activate Workflows

In each workflow, top right corner: switch **Active** toggle to ON.

After activation, all triggers run on their schedules:

| Workflow | Schedule |
|---|---|
| Scout | Daily 09:00 |
| Researcher | Daily 11:00 |
| Copywriter | Daily 13:00 |
| Sender | Daily 15:00 |
| Tracker | IMAP push (real-time) |
| Notifier | Monday 09:00 |
| Follow-up | Daily 14:00 |

Adjust timezones in workflow settings (default Europe/Moscow — change to your TZ).

---

## Step 11: Monitoring

For the first 7 days:

- Check Telegram daily for unexpected alerts
- Open n8n Executions tab — look for any red (failed) runs
- Check Sender's `Sent` folder in Gmail — verify emails look good
- Monitor reply rate — if 0% after 3 days, something's wrong with deliverability

After 7 days the system runs itself. You only need to:

- Reply to hot lead Telegram alerts within 1 hour
- Read Monday weekly report
- Add new prospects to Scout's source list as needed

---

## Troubleshooting

### "Workflow can't find credential"
You skipped credential rebinding after import. Click each node, re-select credential from dropdown.

### "Google Sheets returns 0 rows"
Service Account doesn't have access. Re-share the sheet with the service account email.

### "Telegram error 400: chat not found"
You didn't press `/start` on the bot. Open Telegram → find bot → press Start.

### "Claude returns invalid JSON"
The Parse Claude Response nodes have fallback handling. If errors persist, increase `max_tokens` in the request body.

### "Gmail bounces or blocks sending"
Slow down. Increase Wait node delays. Verify you used App Password, not regular Gmail password. Consider warming up the inbox (start with 5 emails/day, scale up over 2 weeks).

### "Follow-up sends to wrong language"
The Code node reads `language` column from Outreach. Verify Scout fills it correctly. Default fallback is `en`.

---

## Cost Estimate

Monthly running costs for a single client:

| Item | Cost |
|---|---|
| VPS (basic) | $5-15 |
| Anthropic API (1500 emails/mo) | $5-15 |
| Domain (optional, for branded sending) | $1 |
| **Total** | **~$10-30/month** |

For most B2B niches, one closed deal pays for 12+ months of infrastructure.

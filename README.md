# AI Email Auto-Reply Assistant

![n8n](https://img.shields.io/badge/n8n-workflow-EA4B71?style=flat-square&logo=n8n)
![Claude AI](https://img.shields.io/badge/Claude-AI-orange?style=flat-square)
![Gmail](https://img.shields.io/badge/Gmail-API-red?style=flat-square&logo=gmail)
![Google Sheets](https://img.shields.io/badge/Google_Sheets-logging-34A853?style=flat-square&logo=googlesheets)
![Slack](https://img.shields.io/badge/Slack-alerts-4A154B?style=flat-square&logo=slack)
![License](https://img.shields.io/badge/license-MIT-blue?style=flat-square)

> An n8n automation workflow that watches your Gmail inbox, triages every message with AI, generates context-aware draft replies, routes urgent emails to Slack, and logs everything to Google Sheets — fully automated, zero code to deploy.

---

## Overview

Most inboxes are full of noise. This workflow separates signal from noise using a two-stage AI pipeline — a cheap filter to skip junk, then a full triage to classify, prioritize, and draft a reply — all without a single line of application code.

**Built by [Haseeb Zahid](https://github.com/haseeb-developer) as a portfolio project** demonstrating real-world AI automation, prompt engineering, and multi-service API orchestration.

---

## How It Works

```
 ┌─────────────────────────────────────────────────────────┐
 │                     New Gmail Email                     │
 └────────────────────────┬────────────────────────────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │  Stage 1: Filter      │  cheap 10-token LLM call
              │  PROCESS or IGNORE?   │  skips newsletters, CI/CD,
              └──────────┬────────────┘  no-reply senders
                         │
              ┌──────────┴──────────┐
           IGNORE                PROCESS
              │                     │
              ▼                     ▼
           [skip]       ┌───────────────────────┐
                        │  Stage 2: AI Triage   │  classifies category,
                        │  category · priority  │  priority, intent,
                        │  intent · summary     │  shouldReply flag
                        └──────────┬────────────┘
                                   │
                       ┌───────────┴────────────┐
                  shouldReply?               not replying
                       │                        │
                       ▼                        ▼
           ┌────────────────────┐     ┌──────────────────┐
           │  Generate Reply    │     │   Is Urgent?     │
           │  (persona-locked,  │     └────────┬─────────┘
           │   120-word cap)    │         ┌────┴────┐
           └─────────┬──────────┘      Urgent     Low/Med
                     │                    │          │
                     ▼                    ▼          ▼
           ┌──────────────────┐  ┌──────────────┐  [skip]
           │  Gmail Draft     │  │  Slack Alert │
           │  (human reviews) │  │  #email-alerts│
           └─────────┬────────┘  └──────┬───────┘
                     │                  │
                     └────────┬─────────┘
                              ▼
                  ┌─────────────────────┐
                  │  Google Sheets Log  │  one row per email
                  │  (full audit trail) │
                  └─────────────────────┘
```

---

## Features

| Feature | Details |
|---|---|
| **Smart filtering** | Skips newsletters, no-reply senders, CI/CD notifications before spending tokens on triage |
| **AI triage** | Classifies each email by category (Job / Client / Support / Meeting / Spam / Personal / Newsletter) and priority (Urgent → Low) |
| **Context-aware replies** | Drafts professional, persona-locked replies tailored to email type — word-capped at 120 |
| **Draft, not Send** | Replies are saved as Gmail Drafts; a human reviews before anything goes out |
| **Urgent routing** | Anything flagged Urgent that gets no reply triggers a Slack alert so nothing slips through |
| **Full audit log** | Every processed email appended to Google Sheets with date, sender, category, priority, summary, intent, and reply status |
| **Error observability** | Dedicated error handler posts a Slack notification with the failing node name and message on any workflow failure |
| **Model-agnostic** | Runs on Claude (Anthropic) or any OpenAI-compatible API — swap `LLM_API_URL`, `LLM_API_KEY`, `LLM_MODEL` |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Workflow engine | [n8n](https://n8n.io/) — self-hosted (Docker / npm) or n8n Cloud |
| Email | Gmail API via Google OAuth2 |
| AI / LLM | Claude (Anthropic) — primary; any OpenAI-compatible endpoint supported |
| Logging | Google Sheets API via Google OAuth2 |
| Alerts | Slack API (OAuth2) |
| Configuration | Environment variables — no secrets in `workflow.json` |

---

## Repository Structure

```
AI-Email-Assistant/
├── workflow.json               ← Import directly into n8n
├── config/
│   └── .env.example            ← Copy to .env and fill in values
├── prompts/
│   ├── filter_prompt.txt       ← Stage 1: PROCESS/IGNORE (10-token call)
│   ├── analyze_prompt.txt      ← Stage 2: triage — category, priority, intent
│   ├── reply_prompt.txt        ← Reply generation — persona-locked, 120-word cap
│   ├── slack_alert_prompt.txt  ← Urgent alert format
│   └── weekly_digest_prompt.txt← Future: scheduled weekly summary
├── docs/
│   └── workflow-notes.md       ← Architecture decisions and known limitations
├── screenshots/                ← Add your n8n canvas screenshots here
└── README.md
```

---

## Quick Start

### Prerequisites

- n8n instance (local or cloud)
- Google Cloud project with Gmail API + Sheets API enabled
- Slack app with `chat:write` permission
- Anthropic API key (or any OpenAI-compatible API key)

### 1. Run n8n

```bash
# Docker (recommended)
docker run -it --rm \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n

# Or with npm
npx n8n
```

Open `http://localhost:5678`

### 2. Import the Workflow

1. In n8n: **Workflows → Import from File**
2. Select `workflow.json`

### 3. Add Credentials

In n8n **Settings → Credentials**, create:

| Credential | Type |
|---|---|
| Gmail | Google OAuth2 API |
| Google Sheets | Google OAuth2 API |
| Slack | Slack OAuth2 API |
| LLM Provider | HTTP Header Auth (`Authorization: Bearer YOUR_KEY`) |

### 4. Set Environment Variables

```bash
cp config/.env.example .env
```

Edit `.env` with your values:

```env
LLM_API_URL=https://api.anthropic.com/v1
LLM_API_KEY=sk-ant-...
LLM_MODEL=claude-sonnet-5
GOOGLE_SHEET_ID=your_spreadsheet_id_here
GOOGLE_SHEET_TAB=EmailLog
SLACK_CHANNEL=#email-alerts
```

To use a different LLM, change only the three `LLM_*` variables — no workflow edits needed.

### 5. Create the Google Sheet

Create a spreadsheet with a tab named `EmailLog` with these headers:

```
Date | Sender | Subject | Category | Priority | Summary | Intent | ReplyGenerated | LoggedAt
```

Copy the spreadsheet ID from the URL and set it as `GOOGLE_SHEET_ID`.

### 6. Activate

Toggle the workflow **Active** in n8n. It polls Gmail every minute.

---

## Prompt Engineering

All prompts live in `prompts/` and are designed for **reliable, structured output**:

| Prompt | Strategy | Output |
|---|---|---|
| `filter_prompt.txt` | Single word only; fail-safe defaults to PROCESS | `PROCESS` or `IGNORE` |
| `analyze_prompt.txt` | Strict JSON schema with fixed enums; `temperature: 0.2` | `{ category, priority, summary, intent, shouldReply }` |
| `reply_prompt.txt` | Persona-locked, word-capped, no invented facts; `temperature: 0.7` | Plain-text reply ≤120 words |
| `slack_alert_prompt.txt` | Character-capped, fixed format | `🚨 [Name] (Category): ask — action required.` |
| `weekly_digest_prompt.txt` | Structured executive summary ≤150 words, no markdown headers | Digest paragraph |

Key design choices:
- **Two-stage filtering** avoids spending triage tokens on newsletters and bot emails
- Triage and reply prompts are **tuned independently** — better accuracy than a single combined call
- All LLM calls use **HTTP Request nodes**, not n8n native AI nodes — no vendor lock-in

---

## Workflow Nodes (20 total)

| Node | Type | Purpose |
|---|---|---|
| Gmail Trigger | gmailTrigger | Polls inbox every minute |
| Set Variables | set | Normalizes subject, from, body, threadId |
| PROCESS or IGNORE | httpRequest | Stage 1 AI filter |
| Should Process? | if | Branch on PROCESS/IGNORE |
| Ignore Email | noOp | Terminal for skipped emails |
| AI Triage | httpRequest | Stage 2 classification |
| Extract Triage JSON | set | Parses LLM JSON to named fields |
| Should Reply? | if | Branch on `shouldReply` flag |
| Generate Reply | httpRequest | LLM drafts the reply |
| Extract Reply Text | set | Pulls reply string from response |
| Create Gmail Draft | gmail | Saves as draft on original thread |
| Is Urgent? | if | Branch on `priority == "Urgent"` |
| Build Slack Alert | set | Formats the alert string |
| Send Slack Alert | slack | Posts to `#email-alerts` |
| Skip Reply | noOp | Terminal for non-urgent no-reply |
| Merge for Log | merge | Converges all branches |
| Log to Google Sheets | googleSheets | Appends one row per email |
| Error Handler | errorTrigger | Catches any node failure |
| Notify on Error | slack | Posts error to Slack |
| Installation Complete | noOp | Terminal success marker |

---

## Known Limitations

- **Polling, not push**: Gmail Trigger checks every minute. For near-real-time, use n8n Cloud with Gmail push webhook.
- **JSON parsing**: The "Extract Triage JSON" node uses inline `JSON.parse()` with no try/catch — a malformed LLM response will throw and route to the error handler. Add a Code node with validation for production use.
- **Merge mode**: The Merge node is set to `passThrough input1`. If both inputs arrive simultaneously, input2 data is dropped before logging.

---

## Roadmap

- [ ] Language detection — reply in the sender's language
- [ ] PDF attachment summarization before reply generation
- [ ] Google Calendar integration — auto-check availability for meeting requests
- [ ] Sentiment/tone detection to adjust reply warmth
- [ ] Gmail label tagging for processed emails (`AI-Processed`)
- [ ] Auto-create Jira/Trello tickets for support emails
- [ ] Weekly digest sub-workflow (prompt already written)
- [ ] RAG over company docs for context-aware client replies
- [ ] Feedback loop — fine-tune reply style from edited drafts
- [ ] CRM sync (HubSpot / Salesforce) for client emails

---

## What This Project Demonstrates

- **Event-driven automation** with n8n trigger nodes
- **Multi-service API integration** — Gmail, Google Sheets, Slack, LLM APIs via OAuth2 and HTTP
- **Structured prompt engineering** — deterministic JSON output, enum constraints, persona locking, token budgets
- **Conditional workflow branching** — multi-path logic with a clean merge before logging
- **Error observability** — dedicated error handler with Slack notifications for any failure
- **Security best practices** — all secrets in environment variables, never in workflow JSON
- **Model-agnostic LLM integration** — HTTP Request nodes work with any provider

---

## License

[MIT](LICENSE) — free to use, modify, and build on.

---

*Built by Haseeb Zahid*

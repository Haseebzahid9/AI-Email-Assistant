# Workflow Architecture Notes

## Design Decisions

- **Two-stage AI filtering**: A cheap single-word PROCESS/IGNORE call runs first to skip junk mail before the expensive triage call. Saves tokens on newsletters, GitHub notifications, and automated emails.
- **HTTP Request nodes for LLM**: Avoids vendor lock-in. Works with Claude, OpenAI-compatible APIs, or any self-hosted model endpoint by changing `LLM_API_URL` and `LLM_API_KEY` env vars.
- **Reply as Draft, not Send**: All generated replies are saved as Gmail Drafts. Human reviews before sending — avoids accidental auto-sends to clients.
- **Separate triage and reply prompts**: Classification and drafting are tuned independently. Better accuracy on both tasks than a single combined prompt.
- **Merge node before logging**: All branches (replied, urgent, skipped) converge at a single Merge node so every processed email gets one consistent log row.
- **Error Handler node**: n8n's built-in ErrorTrigger catches any node failure and posts a Slack notification — no silent failures.

## Environment Variables

All sensitive values (API keys, sheet IDs, channel names) live in environment variables, never hardcoded in the workflow JSON. This makes the workflow safe to share publicly.

## Google Sheets Schema

| Column | Value |
|---|---|
| Date | Original email date |
| Sender | From address |
| Subject | Email subject |
| Category | AI-classified category |
| Priority | Urgent / High / Medium / Low |
| Summary | One-sentence AI summary |
| Intent | Detected sender intent |
| ReplyGenerated | true / false |
| LoggedAt | Timestamp of processing |

## Known Limitations

- Gmail Trigger polls every minute — not true real-time (n8n Cloud supports push via webhooks)
- LLM JSON parsing uses `JSON.parse()` inline; if the model returns malformed JSON the Extract node will error — add a Code node with try/catch for production hardening
- Merge node is set to `passThrough input1` — if both inputs arrive simultaneously, input2 data is dropped before logging; adjust merge mode if full data from both branches is needed

## Suggested Improvements

- Add a Code node after AI Triage to validate and sanitize the JSON before Extract
- Store raw AI responses in a separate audit sheet for debugging
- Add a weekly digest sub-workflow triggered by a Schedule node every Monday 9am
- Add a Gmail label step to tag processed emails (e.g. "AI-Processed")

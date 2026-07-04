# Build Plan — Supplier Statement Tracker

Status: **planning complete, build blocked on credentials + 3 open decisions (see bottom).**
Source brief: `statement-tracker-build-brief.md`

---

## 0. Confirmed against the live instance

Checked via `list_credentials` on `your self-hosted n8n instance`:

| Credential | Type | Status |
|---|---|---|
| OpenRouter API | `openRouterApi` | ✅ exists (id `a2VSnMf7Q3etliJ2`) |
| Gmail account | `gmailOAuth2` | ✅ exists (id `vpyNsOo3MKqP4klT`) |
| Google Sheets | `googleSheetsOAuth2Api` | ❌ missing — **add before build** |
| Telegram | `telegramApi` | ❌ missing — **add before build** |

Note: the two existing credentials live under project "Omar Faruq <<redacted-email>>" (personal project), not your account. Not a blocker, just flag if you want them moved.

Confirmed exact node types/versions via `search_nodes` + `get_node_types` (so the build uses real parameter names, not guesses):

| Purpose | Node type | Version |
|---|---|---|
| Email trigger | `n8n-nodes-base.gmailTrigger` | 1.4 |
| Label / addLabels | `n8n-nodes-base.gmail` (resource `label`/`message`) | 2.2 |
| Classifier | `@n8n/n8n-nodes-langchain.textClassifier` | 1.1 |
| Classifier model | `@n8n/n8n-nodes-langchain.lmChatOpenRouter` | 1 |
| Vision extraction | `n8n-nodes-base.httpRequest` | 4.4 |
| Normalize/validate | `n8n-nodes-base.code` | 2 |
| Sheets upsert/read | `n8n-nodes-base.googleSheets` | 4.7 |
| Schedule | `n8n-nodes-base.scheduleTrigger` | 1.3 |
| Telegram send | `n8n-nodes-base.telegram` | 1.2 |
| Routing | `n8n-nodes-base.switch` | 3.4 |

---

## 1. Workflow A — "Statements: Ingest & Extract"

```
Gmail Trigger (poll 5 min, simple=false, downloadAttachments=true)
  → Text Classifier (categories: Statement / Order / Debit / Inquiry; fallback="other" → 5th branch)
       ├─ Statement → Gmail addLabels("Statements")
       │      → Split Attachments (Code, run-once-for-all-items: fan out 1 item per PDF binary key)
       │      → Prepare Base64 (Code, per-item: getBinaryDataBuffer → base64 string field)
       │      → Vision Extract (HTTP→OpenRouter, onError continueErrorOutput, retryOnFail 3x)
       │            ├─ success → Normalize + Validate (Code)
       │            └─ error   → Mark Needs Manual (Set: status="Needs Manual", notes=error)
       │      → [both branches fan in] → Google Sheets upsert (Statements tab)
       ├─ Order    → Gmail addLabels("Orders")
       ├─ Debit    → Gmail addLabels("Debit")
       ├─ Inquiry  → Gmail addLabels("Inquiry")
       └─ Other (fallback) → no label, no extraction (dead-ends intentionally per brief §4)
```

### Node-by-node detail

**Gmail Trigger** — `pollTimes` every 5 min; `simple: false` (need full body, not snippet-only) + `filters.downloadAttachments: true`... actually `downloadAttachments` lives under `options` when `simple` is false. Filter: `filters.readStatus: 'unread'`, optional `filters.q` left open (catches all mail) since classification handles routing.

**Brief deviation — no `snippet` field:** with `simple:false` (required to get attachments), Gmail Trigger's output has `{subject, from, text, html, ...}` but **no `snippet`** field (that only exists in simple mode, which drops attachments). The classifier prompt will use `subject + from + text.substring(0,500) + attachment filenames` instead of snippet — same signal, no loss.

**Text Classifier** — `inputText` built inline via `expr()`:
```
Subject: {{ $json.subject }}
From: {{ $json.from }}
Attachments: {{ Object.keys($binary || {}).filter(k => k.startsWith('attachment_')).map(k => $binary[k].fileName).join(', ') }}
Body (excerpt): {{ ($json.text || '').substring(0, 500) }}
```
4 categories (Statement/Order/Debit/Inquiry) each with name **and** description per brief §5a. `options.fallback: 'other'` → unmatched mail routes to a 5th "Other" output instead of being silently discarded (this is the Text Classifier's built-in catch-all, replacing the brief's manual "Other" category). Model sub-node: `lmChatOpenRouter`, `model: 'anthropic/claude-haiku-4.5'`, `temperature: 0.1`.

**Split Attachments (Code, run-once-for-all-items)** — Gmail Trigger names attachments `attachment_0`, `attachment_1`, ... not `data`. This node iterates every input item's `$binary` keys matching `/^attachment_/` and emits one output item per PDF attachment, carrying that single binary plus the parent email's `subject/from/messageId/date`. (A genuine Code-node case — fanning items out by binary key isn't expressible as an expression or Split Out, which only splits JSON arrays.)

**Prepare Base64 (Code, per-item)** — `getBinaryDataBuffer` → `Buffer.from(...).toString('base64')`, output as a JSON string field (`pdfBase64`) for the HTTP body. Re-attaches `binary` so the original file survives for debugging/links if needed.

**Vision Extract (HTTP Request)** — `authentication: 'predefinedCredentialType'`, `nodeCredentialType: 'openRouterApi'`, POST `https://openrouter.ai/api/v1/chat/completions`, JSON body per §5b prompt, model `google/gemini-2.5-flash`, `response_format: json_object`. `onError: 'continueErrorOutput'` + `retryOnFail: true, maxTries: 3, waitBetweenTries: 5000` (transient API blips retry before counting as a real failure, per `n8n-error-handling`).

> **Build-time verification needed:** the brief flags that the exact OpenRouter content-block shape for sending a PDF (`type: "file"` with a `file_data` data-URI, vs an image-block fallback) must be confirmed for the chosen model before this node is wired for real. I'll verify this against OpenRouter's current docs during the build step, not guess from training data.

**Normalize + Validate (Code)** — implements brief §5c exactly: strip ```json fences, fuzzy-match `branch` against `KNOWN_BRANCHES`, reconciliation check (`|sum - total| > 1` → note + confidence −0.2), `status = "OK"` only if `confidence >= 0.75 AND branch != "Unknown"`, else `"Needs Manual"`. Adds `email_date`, `gmail_message_id`, `gmail_link`, `processed_at`.

**Mark Needs Manual (Set)** — runs only on the HTTP error branch; builds the same row shape as Normalize's output but with `status: "Needs Manual"`, `notes: "extraction failed: " + error message`, all numeric buckets `0` — so it lands in the same Sheets upsert with the same columns, never silently dropped (brief §9 edge case).

**Google Sheets upsert** — `appendOrUpdate`, `documentId`/`sheetName` via resource-locator `mode: 'list'`, `columns: { mappingMode: 'defineBelow', matchingColumns: ['branch','supplier','statement_date'] }` (required for upsert — see §6).

---

## 2. Workflow B — "Statements: Nightly Telegram Digest"

```
Schedule Trigger (daily 20:00, tz Africa/Johannesburg)
  → Google Sheets read (Statements tab)
  → Aggregate (Code, run-once-for-all-items): group OK rows by branch, sum(total),
       top-5 creditors/branch, count Needs-Manual, grand total
  → Telegram sendMessage (Markdown, parse_mode HTML/Markdown, per brief §7 template)
```

Per-branch sections only render when that branch has rows (brief §7: "Show only branches that have data"). The Needs-Manual count is **always** shown, even when zero, so a silent gap is never possible.

---

## 3. Google Sheet structure (brief §6)

- **`Statements`** tab — written by Workflow A only. Columns exactly as brief's table.
- **`Dashboard`** tab — `QUERY()` formulas only, never written by either workflow (so re-runs never clobber them). I'll create both tabs via the Google Sheets node once the sheet ID/credential exists, then paste the `QUERY` formulas directly (formulas aren't something the Sheets *write* operation should touch — they go in once, by hand via the node's raw update).

---

## 4. Error handling (per `n8n-error-handling`)

- Vision Extract: `onError: continueErrorOutput` + wired to "Mark Needs Manual" (both halves — set AND wire, verified after creation via `n8n_get_workflow`/`get_workflow_details`).
- Gmail/Sheets/Telegram nodes: `retryOnFail: true, maxTries: 3, waitBetweenTries: 5000` (transient API/network blips self-heal before they'd ever need a human).
- No workflow-level Error Workflow is in the brief's scope — I'd recommend one (catch-all → Telegram or email alert on a *different* channel) as a fast follow once both workflows are running; flagging it here rather than silently adding scope.

---

## 5. Validate → verify → test → activate (per brief §8, non-negotiable)

1. `validate_workflow` on the generated SDK code for both workflows; fix every error.
2. `create_workflow_from_code`, then `get_workflow_details` to pull `connections` and confirm: all 5 classifier outputs wired, Vision Extract's error output wired, Sheets upsert reachable from both Normalize and Mark-Needs-Manual.
3. `test_workflow` with `prepare_test_pin_data` — Workflow A against **one sample statement email** first (real Gmail label writes + a real Sheets row + a real OpenRouter call — confirming with you before this fires, since it has live side effects). Workflow B tested after the sheet has a few rows.
4. `publish_workflow` (activate) only after 1–3 pass clean.

---

## 6. Open decisions (need your call before/while building)

1. **Google Sheet** — create a brand-new sheet, or do you have an existing spreadsheet ID to use?
2. **Vision model** — brief assumes `google/gemini-2.5-flash` for cost. Keep that, or upgrade to Claude Sonnet / Gemini 2.5 Pro for accuracy (slower scanned PDFs, pricier)? A "cheap first, escalate low-confidence rows" two-tier option is also on the table if you want it.
3. **Telegram chat ID** — once you create the bot (see credentials doc), send it one message and I can fetch the chat ID via the Bot API, or you can grab it from `@get_id_bot`.
4. **Digest timezone** — SAST (`Africa/Johannesburg`) assumed correct for the client's branches. Confirming, not asking unless you want it different.

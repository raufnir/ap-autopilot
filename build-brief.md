# Build Brief — Supplier Statement Tracker (n8n + Vision LLM + Telegram)

**For:** Claude Code (with `n8n-mcp` connected to the live n8n instance)
**Goal:** Build, validate, test, and activate the workflows described below using the n8n MCP tools. Do **not** activate until the validate → verify-connections → test gates all pass.

---

## 0. What we're building (context)

The end user is a hardware-supply business owner who operates **three branches**:

| Branch | Notes |
|---|---|
| Dunnottar Hardware | 78 Birkenruth Ave |
| Selcourt Hardware | 204 Nigel Rd, Springs |
| Eastvale Hardware | Fariha General Traders |

Suppliers (creditors) email him **statements** showing what each branch *owes*, aged into buckets (Current / 30 / 60 / 90 / 120+ days). The PDFs are **often scanned image-PDFs** (no selectable text), with **inconsistent layouts** between suppliers. He needs:

1. Incoming Gmail **auto-classified** (Statement / Order / Debit / Inquiry) and **labelled** so he can find them.
2. Each **statement PDF parsed** — even scanned ones — into structured rows.
3. A **structured Google Sheet** that mirrors an "Aged Outstanding by Branch" dashboard.
4. A **short nightly Telegram message**: per branch, *who he owes and how much*, plus how many statements couldn't be parsed.

Build this as **two separate workflows** (rationale: ingestion is event-driven, the digest is scheduled — keep them decoupled).

---

## 1. Config values (set these as workflow-level constants / a "Set" node at the top)

```
TIMEZONE            = Africa/Johannesburg   # client is in SA (SAST, UTC+2)
DIGEST_TIME         = 20:00                 # nightly Telegram send time
EXTRACTION_MODEL    = google/gemini-2.5-flash   # vision OCR via OpenRouter; native PDF support, cheap
CLASSIFIER_MODEL    = anthropic/claude-haiku-4.5 # text-only, cheap (already in his stack)
CONFIDENCE_FLOOR    = 0.75                  # below this → mark row "Needs Manual"
KNOWN_BRANCHES      = ["Dunnottar Hardware", "Selcourt Hardware", "Eastvale Hardware"]
TELEGRAM_CHAT_ID    = <<USER FILLS IN>>
GOOGLE_SHEET_ID     = <<create/confirm, see §6>>
```

> The vision model **must accept the PDF as input** (base64 file or image content). Gemini 2.5 Flash via OpenRouter does this natively and OCRs scanned pages well. If a chosen model rejects raw PDF, fall back to converting PDF pages → images first, then send images. Verify the exact OpenRouter content-block format for the chosen model before wiring the HTTP node.

---

## 2. Credentials to reuse (already on the instance)

- **Gmail OAuth2** (read + modify labels)
- **OpenRouter** API key (for both classifier and vision extraction)
- **Google Sheets OAuth2**
- **Telegram** bot credential (user will create the bot + provide chat ID)

Confirm each exists via the MCP before building; don't hardcode keys in node parameters.

---

## 3. Architecture

```
WORKFLOW A — "Statements: Ingest & Extract"   (event-driven)
────────────────────────────────────────────────────────────
Gmail Trigger (poll new mail)
   → AI Classifier  (text: subject + from + snippet + attachment names)
        → Switch on category
             ├─ Statement → Gmail: add label "Statements"
             │       → Download PDF attachment(s)
             │       → Vision Extract (OpenRouter, returns strict JSON)
             │       → Normalize + Validate (Code)
             │       → Google Sheets: upsert into "Statements" tab
             ├─ Order    → Gmail: add label "Orders"
             ├─ Debit    → Gmail: add label "Debit"
             └─ Inquiry  → Gmail: add label "Inquiry"
   (Continue-On-Fail on extraction so one bad PDF can't halt the run)

WORKFLOW B — "Statements: Nightly Telegram Digest"   (scheduled)
────────────────────────────────────────────────────────────
Schedule Trigger (daily @ DIGEST_TIME, TIMEZONE)
   → Google Sheets: read "Statements" tab
   → Aggregate (Code): totals per branch + top creditors + needs-manual count
   → Telegram: sendMessage (concise Markdown)
```

---

## 4. Workflow A — node-by-node

**A1. Gmail Trigger** — poll every 2–5 min, "message received", download attachments = true.

**A2. AI Classifier** — a Text Classifier node *or* Basic LLM Chain using `CLASSIFIER_MODEL`. Classify from `subject + from + snippet + attachment filenames` only (cheap, no vision yet). Categories + routing rules:

```
Statement → a supplier/creditor account statement, aging balances, "amount due",
            "statement of account", attached PDF ledger.
Order     → purchase orders, order confirmations, quotes, invoices for new goods.
Debit     → debit notes / debit advices / bank debit notifications.
Inquiry   → questions, follow-ups, general correspondence, no document to extract.
Other     → anything that fits none of the above (no label, no extraction).
```
> Use the full `{{ $json.subject }} {{ $json.snippet }}` (not snippet alone — snippet truncates).

**A3. Switch** — one output per category → **Gmail node ("add label")** per branch. Create the four labels if missing.

**A4. (Statement branch) Download attachment** — ensure the PDF binary is available (Gmail Trigger can attach; otherwise a Gmail "get message" with attachment download).

**A5. Vision Extract** — HTTP Request → OpenRouter `/chat/completions` with `EXTRACTION_MODEL`, sending the PDF as base64 file/image content + the prompt in §5. **Force JSON output.**

**A6. Normalize + Validate (Code node)** — see §5 logic.

**A7. Google Sheets — upsert** into the `Statements` tab. **Upsert key** = `branch + supplier + statement_date` (prevents duplicate rows when the same statement re-arrives). Map every field from the JSON.

---

## 5. The two AI prompts

### 5a. Classifier (text)
> System: "You are an email triage classifier for a hardware business. Read the email metadata and return exactly one category: Statement, Order, Debit, Inquiry, or Other. Definitions: [paste the rules from A2]. Return only the category word."

### 5b. Vision extraction (the important one)
Send the PDF + this instruction. Demand **strict JSON, no prose, no markdown fences**:

```
You are extracting a supplier account statement (often a scanned image). Read the
document carefully, including handwritten or low-quality text. Return ONLY a JSON
object with this exact shape:

{
  "branch": "<which of the buyer's businesses this statement is addressed TO,
              i.e. the 'bill-to' / account-holder name. Match to one of:
              Dunnottar Hardware, Selcourt Hardware, Eastvale Hardware.
              If unclear, return the raw name you see>",
  "supplier": "<the creditor / company that ISSUED the statement>",
  "account_no": "<account/reference number, or null>",
  "statement_date": "<YYYY-MM-DD, or null>",
  "current": <number>,
  "d30": <number>,
  "d60": <number>,
  "d90": <number>,
  "d120plus": <number>,
  "total": <number>,
  "currency": "<e.g. ZAR>",
  "confidence": <0.0-1.0, your honest certainty the numbers are correct>,
  "notes": "<anything ambiguous, e.g. 'totals do not reconcile'>"
}

Rules:
- Numbers only (no currency symbols, no thousands separators).
- If a bucket is absent, use 0.
- Never invent values you cannot read — lower the confidence instead.
```

### 5c. Normalize + Validate (Code node logic)
```
1. Parse the JSON (strip stray ```json fences first).
2. Branch normalization: fuzzy-match `branch` against KNOWN_BRANCHES.
   No confident match → branch = "Unknown".
3. Reconciliation check: if |(current+d30+d60+d90+d120plus) - total| > 1
   → append "sum mismatch" to notes, drop confidence by 0.2.
4. Status:
      confidence >= CONFIDENCE_FLOOR  AND branch != "Unknown"  → "OK"
      else → "Needs Manual"
5. Add: email_date, gmail_message_id, gmail_link
   (https://mail.google.com/mail/u/0/#inbox/{id}), processed_at.
6. Output one clean item for the Sheets upsert.
```

---

## 6. Google Sheet structure

**Tab 1 — `Statements`** (raw master, one row per supplier-statement; the workflow writes here):

| email_date | branch | supplier | account_no | statement_date | current | d30 | d60 | d90 | d120plus | total | currency | status | confidence | notes | gmail_link |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|

**Tab 2 — `Dashboard`** (mirrors the "Aged Outstanding by Branch" view; built with `QUERY` formulas so it auto-refreshes — the workflow does **not** write here):

- **Branch summary block** (top): for each branch, total outstanding + supplier count + count Needs-Manual. Use:
  `=QUERY(Statements!A:P, "select B, sum(K) where M='OK' group by B label sum(K) 'Outstanding'", 1)`
- **Per-branch aging table**: replicate the screenshot columns (Supplier / Date / Current / 30 / 60 / 90 / 120+ / Total), sorted by Total desc, filtered per branch.

> Claude Code: create the sheet via the Google Sheets node (or have the user create it and supply `GOOGLE_SHEET_ID`), then write the `QUERY` formulas into `Dashboard`. Keep the raw tab and the view tab separate so re-runs never clobber formulas.

---

## 7. Workflow B — Nightly Telegram digest

**B1. Schedule Trigger** — daily at `DIGEST_TIME`, workflow timezone = `TIMEZONE`.

**B2. Google Sheets — read** the `Statements` tab.

**B3. Aggregate (Code)** —
```
- Group OK rows by branch → sum(total) = branch outstanding.
- Within each branch, take top 5 creditors by total owed (supplier + amount).
- Count rows where status = "Needs Manual".
- Grand total across branches.
```

**B4. Telegram — sendMessage** (Markdown). Keep it short — it's a reminder, not a report:

```
🧾 *Outstanding — {date}*

*Dunnottar Hardware* — R 174,277
 • ARAF Industries — R 99,094
 • Redstar Group — R 53,080
 • GEO Building Plumbing — R 6,246

*Selcourt Hardware* — R 26,584
 • <top creditor> — R …

*Eastvale Hardware* — pending

*Total owed:* R 200,861
⚠️ 9 statements need manual entry
```

(Show only branches that have data; list top creditors per branch; always show the manual-entry count so nothing is silently missed.)

---

## 8. Build / validate / test lifecycle (do not skip)

1. **Build** each workflow with the n8n MCP. Use `search_nodes` to confirm node types; don't guess parameter names.
2. **Validate** — run `validate_workflow` on the JSON, then `n8n_validate_workflow({id})` once created. Fix every error and re-validate.
3. **Verify connections** — pull the workflow and read the `connections` object. Confirm: the Switch's 5 outputs each go somewhere, the Statement branch chains through to Sheets, Continue-On-Fail is set on the extraction node, and no branch dead-ends.
4. **Test with real side effects in mind** — extraction + Sheets writes + Telegram sends are real. Test Workflow A against **one sample statement email** first; test Workflow B after the sheet has a few rows. Inspect executions before trusting output.
5. **Activate** only after 1–4 pass.

---

## 9. Edge cases to handle explicitly

- **Scanned / encoded-font PDFs** → the vision model OCRs them; if confidence < floor, row = **Needs Manual** (never dropped silently).
- **Multiple attachments** in one email → process each; one row per statement.
- **Unknown branch** → row still saved, branch = "Unknown", status = Needs Manual.
- **Sum-vs-total mismatch** → flagged in notes + confidence penalty.
- **Duplicate statement re-sent** → upsert key dedupes.
- **Non-statement mail** → labelled only, no extraction.
- **Extraction API failure** → Continue-On-Fail; log + mark Needs Manual rather than halting the batch.

---

## 10. Confirm with the user before/while building

1. **Telegram chat/channel ID** (user is creating the bot).
2. **Google Sheet** — create new, or use an existing ID?
3. **Digest timezone** — SAST (client) assumed; change if Rauf wants it on his own clock.
4. **Vision model** — Gemini 2.5 Flash assumed for cost; offer Claude Sonnet / Gemini 2.5 Pro as an accuracy upgrade, or a two-tier "cheap first, escalate low-confidence rows to a stronger model" option.

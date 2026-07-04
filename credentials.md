# Credentials Needed — Statement Tracker

Checked the live instance (`your self-hosted n8n instance`) via `list_credentials`. Status:

## Already on the instance — nothing to do
| Credential | Type | Used by |
|---|---|---|
| OpenRouter API | `openRouterApi` | Classifier model + Vision Extract (Workflow A) |
| Gmail account | `gmailOAuth2` | Gmail Trigger + label nodes (Workflow A) |

## Add these before I build

### 1. Google Sheets — type `Google Sheets OAuth2 API`
- In n8n: Credentials → New → search "Google Sheets" → OAuth2 → sign in with the Google account that should own the tracker spreadsheet.
- Used by: Workflow A (write `Statements` tab), Workflow B (read `Statements` tab).
- Name it something findable, e.g. `Google Sheets - Statement Tracker`.

### 2. Telegram — type `Telegram API`
- Create a bot via Telegram's **@BotFather** (`/newbot`, pick a name) — BotFather gives you a bot token.
- In n8n: Credentials → New → search "Telegram" → paste the **Access Token**.
- Used by: Workflow B (nightly digest send).
- You'll also need the **chat ID** to send to (a DM with the bot, or a group/channel it's a member of) — see plan.md §6.3 for how to get it once the bot exists.

## Not a credential, but needed before build
- **Google Sheet ID** — either give me an existing spreadsheet to write into, or say "create new" and I'll create one and create both tabs (`Statements`, `Dashboard`).

Once Google Sheets + Telegram credentials exist on the instance, tell me and I'll proceed with the build.

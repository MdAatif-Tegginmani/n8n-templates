# Telegram Maintenance Bot (n8n)

This README explains how to import, configure, and run the attached n8n workflow **“Telegram bot”** to capture maintenance tasks from Telegram messages, parse them with AI, prevent duplicates with Supabase, and notify the user of outcomes.

---

## 🚦 What the Workflow Does

- **Listens** to incoming Telegram messages via a Telegram bot.
- **Detects “completion” messages** (keywords: fixed, completed, done) and parses them into a single “completed” task with optional cost using AI.
- **Detects general “new tasks” messages** and splits them into multiple “pending” tasks using AI.
- **Normalizes and validates data**, then checks Supabase for duplicates (for new tasks).
- **Creates or updates records** in a Supabase table `maintenance_tasks`.
- **Replies to the user** on Telegram acknowledging actions and results.

---

## 🛠 Prerequisites

- An n8n instance (self-hosted or cloud).
- A Telegram Bot token (create with [@BotFather](https://core.telegram.org/bots#botfather)).
- A Supabase project with a table `maintenance_tasks` (see [Supabase table DDL example](#supabase-table-ddl-example)).
- API key(s) for Google Gemini (PaLM) via n8n “Google Gemini(PaLM) Api” credential type, or adjust to any other supported LLM.
- Publicly reachable webhook URL for Telegram triggers (n8n Cloud or self-hosted with HTTPS/tunnel).

---

## 🚀 Install and Import

1. **Open n8n** → *Settings* → *Credentials and Community Nodes* (if needed).
2. Ensure the following built-ins are available:
   `Telegram Trigger`, `Telegram`, `Code`, `IF`, `Split in Batches`, `Supabase`, and the n8n LangChain nodes for AI.
3. **Create credentials:**
   - **Telegram account:** Paste the bot token from BotFather.
   - **Supabase account:** Set Base URL and Service Role or anon key (Service Role recommended for insert/update).
   - **Google Gemini(PaLM) Api:** Set Gemini/PaLM API key.
4. **Workflows** → *Import from File* → select the provided `Telegram-bot-2.json`.
5. Open the imported workflow and map node credentials where marked:
   - Telegram Trigger and Telegram send nodes → Telegram account.
   - Supabase nodes → Supabase account.
   - Google Gemini Chat Model nodes → Gemini credentials.

---

## 🧭 High-Level Flow

1. **Telegram Trigger** receives all updates (`*`) and forwards only message content to Code node.
2. **Code** extracts: `user_id`, `user_name`, `chat_id`, `message_text`, `message_id`.
3. **IF node** routes based on `message_text` containing any of: `fixed`, `completed`, `done`.
   - If matched → **“completion” branch**.
   - Else → **“new tasks” branch**.

### Completion Branch (Mark Task Complete)

- **AI Agent (Gemini):** Parses a “completion” message into strict JSON:
  - `description`
  - `property` (mapped from codes: PP→Potts Point, SH/Flinders/Flinder/Findler→Surry Hills, AL/Allen→Allen, CEN/Central→Central, PY/Pyrmont→Pyrmont)
  - `cost` (number or null)
  - `status: "complete"`
- **Code1:** Strips code fences, parses JSON, validates fields, normalizes to lowercase, adds user metadata and `completed_at` timestamp.
- **Get many rows (Supabase):** Looks up a matching pending task by `ilike description` and `property`.
- **If1:**
  - If matching row(s) found, update the first match: set `status=complete`, set `cost` and `completed_at`.
    Send Telegram confirmation “Task completed … Cost: …”.
  - If none found, create a new row with `status=complete`:
    Send Telegram confirmation “Task completed … Cost: …”.

> **Note:** The query uses a partial match on description (`ilike %description%`). Consider tightening logic if false matches occur.
> Cost is optional; when absent, it stores null.

### New Tasks Branch (Parse and Create Pending Tasks)

- **AI Agent1 (Gemini):**
  Given free-form text, returns ONLY a JSON array of tasks:
  ```json
  [
    { "description": "...", "property": "Potts Point", "status": "pending" },
    ...
  ]
  ```
  Uses the same property mapping rules as above; strips room numbers from descriptions; one task per line.
- **Code2:** Strips code fences if present, parses array or falls back to line-based parsing. Normalizes description/property/status, attaches user metadata.
- **Loop Over Items1:** Iterates tasks one-by-one.
- **Code10 + Debug:** Normalizes again to enforce lowercase. Logs what will be used for DB filters.
- **Get many rows1 (Supabase):** Checks for duplicates: pending tasks where property matches and description `ilike %desc%`.
- **Code6:** Converts Supabase results into a `found_duplicates` boolean and attaches original task.
- **If2:**
  - If duplicates found:
    **Code4** → build “Similar task already exists” message.
    **Send a text message1 (Telegram)** to notify, then loop continues.
  - If no duplicates:
    **Create a row1 (Supabase)** to insert the new task.
    **Code9** merges DB result with original task context.
    **Code3** builds a “Task added” message.
    **Send a text message (Telegram)** to notify success.

---

## 🛠️ Customization

- **Property mappings:** Edit the system messages in AI Agent and AI Agent1 to add/remove mappings as needed.
- **Deduplication strategy:** In Get many rows1 and Get many rows, adjust:
  - `description` filter from `ilike %desc%` to tighter matching (e.g., equality or trigram).
  - Include `user_id`/`chat_id` if dedup should be per-user.
- **Completion matching keywords:** In the IF node, add or remove keywords (“fixed”, “completed”, “done”).
- **Status values:** Standardize on `pending`/`complete`; adjust UI and DB constraints accordingly.
- **Cost parsing:** If costs may include currency symbols or ranges, extend Code1 or the AI prompt to sanitize.

---

## 💬 Telegram Setup Tips

- Ensure the Telegram Trigger node shows a “Production URL” and that n8n is reachable on HTTPS.
- Telegram webhooks auto-register through the node when the workflow is activated and credentials are valid.
- If running locally, use a tunnel (e.g., [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/) or [ngrok](https://ngrok.com/)) and set n8n’s `WEBHOOK_URL` environment variable so Telegram can reach it.

---

## 🗄 Supabase Table DDL Example

```sql
create table if not exists maintenance_tasks (
  id uuid primary key default gen_random_uuid(),
  description text not null,
  property text not null,
  status text not null check (status in ('pending','complete')),
  cost numeric(10,2),
  user_name text,
  user_id text,
  chat_id text,
  created_at timestamptz default now(),
  completed_at timestamptz
);
create index on maintenance_tasks (status);
create index on maintenance_tasks (property);
create index on maintenance_tasks using gin (description gin_trgm_ops);
```

Enable `pg_trgm` extension for better `ilike` performance:

```sql
create extension if not exists pg_trgm;
```

Grant RLS/permissions as appropriate; if using Service Role key in n8n, ensure policies allow insert/update/select required by the workflow.

---

## ⚠️ Known Edge Cases and Fixes

- **AI outputs fenced JSON:** Strip code fences before parsing.
- **Ambiguous or missing property names:** The prompts map known abbreviations, but unknown ones will need manual handling or a fallback rule.
- **Duplicate detection is fuzzy:** `ilike %description%` may match unrelated tasks. Consider:
  - Store and compare a normalized hash of description+property.
  - Use similarity threshold with `pg_trgm` (e.g., `similarity > 0.4`).
- **Completion without existing pending task:** The workflow creates a completed row if none exists; if this is undesirable, change If1 to notify the user instead of creating.

---

## 🧪 How to Test

1. **Activate the workflow.**
2. From Telegram, send messages to the bot:
   - `Leaking sink in AL 204` → should create a pending task with property “Allen” and cleaned description (e.g., “leaking sink”).
   - `Fixed leaking sink in AL 204 – $45` or `done sink leak in AL` → should mark matching pending task complete with cost 45.00; if no match, it creates a completed record and confirms.
3. Check n8n Executions for each run and Supabase table for new/updated rows.

---

## 🛡 Maintenance and Operations

- Rotate Telegram and Supabase keys periodically.
- Monitor n8n Executions and set error handling on nodes if needed.
- Version prompts and code nodes; changes in LLM models may affect output format—keep the “Return ONLY valid JSON” instruction strong.
- Consider adding retry logic for Supabase nodes on transient failures.

---

## 🗺 File/Node Map (Important Nodes)

- **Telegram Trigger** → receives Telegram updates.
- **Code** → extracts user/chat/message metadata.
- **If** → routes to completion vs new tasks branches based on keywords.
- **AI Agent + Google Gemini Chat Model** → parses completion message into one task JSON.
- **Code1** → cleans/parses AI JSON, validates, adds metadata.
- **Get many rows (Supabase)** → find pending match for completion.
- **If1** → update existing row or create new completed row.
- **Code5 + Send a text message2** → user confirmation for completion.
- **AI Agent1 + Google Gemini Chat Model1** → parse multi-task messages into array.
- **Code2** → normalize/prepare tasks.
- **Loop Over Items1** → iterate tasks.
- **Debug** → logs normalized filters.
- **Get many rows1** → duplicate check for pending tasks.
- **Code6 + If2** → route duplicate vs create.
- **Create a row1** → insert new task.
- **Code9** → merge DB result with context.
- **Code3/Code4** → craft Telegram response for add/duplicate.
- **Send a text message / Send a text message1** → notify user.

---

## 🔒 Security

- Use HTTPS for webhooks.
- Store API keys in n8n credentials, not in plain text.
- Use Supabase Row Level Security and least-privilege keys where possible.
- Avoid logging sensitive user content in plain logs; use Debug sparingly in production.

---

## 🐞 Troubleshooting Checklist

- **Telegram webhook not firing:**
  Workflow activated? Telegram credential valid? Public HTTPS URL reachable?
- **AI node errors:**
  Gemini API key set? Model quota limits? Tighten prompt to force valid JSON.
- **Supabase errors:**
  Credentials and URL correct? Table exists? Policies permit operations?
- **Duplicate noise:**
  Tighten `ilike` patterns, or implement hashes/similarity thresholds.
- **Wrong property mapping:**
  Update system prompts with missing aliases; consider adding a post-AI mapping Code node to enforce canonical names.

---

> *If desired, this workflow can be extended to handle photos/voice notes, richer statuses, approval flows, or scheduling.*
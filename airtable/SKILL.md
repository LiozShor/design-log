---
name: airtable
description: "Read or write Airtable data — bases, tables, `tbl…`/`rec…`/`fld…` IDs, `filterByFormula` syntax, field types and their coercion rules, batch limits, and the Meta API's boundaries. Covers the MCP tools, the REST API, and pyairtable. Triggers: 'read the Airtable records', 'update this base', 'what is the filterByFormula for…', 'add a field', 'why did my typecast not create the option', 'list the tables in this base'. Do NOT use for n8n workflows that happen to contain Airtable nodes, or for Google Sheets or Excel."
allowed-tools: Bash, Read
---

# Airtable Client

You are an Airtable client that helps users access their bases, tables, and records using curl for reads and Python `pyairtable` for writes.

## When this triggers

- User mentions Airtable, a base, table, record, `filterByFormula`, `pyairtable`, or a `tbl…`/`rec…`/`fld…` ID.
- User asks to query, list, find, count, create, update, upsert, or delete records.
- User asks about schema/fields of an Airtable table.
- A 422 came back from an Airtable request and the user wants help debugging.

## When this does not trigger

- n8n Airtable nodes inside a workflow JSON — use the n8n skill (this skill is for direct API access only).
- Google Sheets, Excel, CSV, or any non-Airtable tabular data.
- Designing/altering long-term schema in this project — append a note to `docs/airtable-schema.md` instead of guessing.
- Editing the `.env` file or rotating tokens — out of scope.

## Required inputs

Before running anything, confirm you have:

- `AIRTABLE_API_KEY` in env (load with `source <project-root>/.env`). First-time setup guide (Python, pyairtable, token creation) in `references/query-patterns.md` § First: Check Prerequisites.
- **Base + table IDs** loaded from config: `source ~/.claude/skills/airtable/config.env` sets `$BASE_ID` and the table-ID vars (`$CLIENTS_TID`, `$REPORTS_TID`, `$DOCUMENTS_TID`, `$PENDING_TID`, `$TEMPLATES_TID`). Template: `config.example.env`. Edit `config.env` only to redeploy on another base — never hardcode raw `app…`/`tbl…` IDs.
- The **table ID** (`tblXXX`) or table name — prefer the ID var.
- For writes/deletes: the user's explicit go-ahead and the exact record IDs or filter.
- For filtered reads: confirmed field names (satisfy the HARD GATE below).

## Workflow

1. Source `<project-root>/.env` (API key) and `~/.claude/skills/airtable/config.env` (base + table IDs); verify `AIRTABLE_API_KEY` and `$BASE_ID` are set.
2. Identify the operation type — read, write, delete, schema introspect.
3. **Satisfy the HARD GATE** (below) — confirm field names before any `filterByFormula`/`sort`.
4. Load `references/query-patterns.md` and build the request from its templates: curl `--data-urlencode` for reads; curl PATCH/POST or `pyairtable` for writes (after the user's permission — privacy rule).
5. On 422: STOP, run the recovery sequence in `references/query-patterns.md` § If you get a 422 — do not retry the same query with tweaks.
6. Format results as a clean table or summary, never as raw JSON dumps.

## Project Context

This project's Airtable base is your CRM base (this template was built around a client/reports/documents CRM — adapt table names to yours). Its base + table IDs are externalized to `config.env` (load `$BASE_ID` and the table-ID vars with `source ~/.claude/skills/airtable/config.env`; template `config.example.env`). The full schema is documented at `docs/airtable-schema.md`. The API key is stored in `.env` at the project root — load it with `source <project-root>/.env` before running any commands.

## HARD GATE — Schema check before ANY query

**Do not write a `filterByFormula` or `sort` until you have confirmed the actual field names of the target table.** Field-name guessing is the #1 cause of 422s in this skill. This gate fires even when the skill is invoked from inside another skill / sub-agent context.

Pick ONE of these to satisfy the gate:

1. You have just read `references/schema-quick-reference.md` in this conversation and are using ONLY fields listed there.
2. You have just read `docs/airtable-schema.md` for the target table in this conversation.
3. You ran a no-filter probe and saw the actual field names:
   ```bash
   curl -s -G "https://api.airtable.com/v0/$BASE_ID/$TABLE_ID" \
     -H "Authorization: Bearer $AIRTABLE_API_KEY" \
     --data-urlencode "maxRecords=1" | python3 -m json.tool
   ```

If none of the three are true, run the probe FIRST. Do not skip this step to "save a tool call" — every guessed query that 422s costs more than the probe.

Also: **lookup / formula / rollup fields cannot be matched with `=`.** Use `SEARCH('value', {field})` or `FIND('value', ARRAYJOIN({field}))`. Example: `reports.client_id` is a **lookup array** (values like `['CPA-210']`) — `{client_id}='CPA-210'` will not match; use `FIND('CPA-210', ARRAYJOIN({client_id}))`. By contrast `pending_classifications.client_id` is plain text (so `{client_id}='CPA-210'` works), and `clients.client_id` is the formula field. (Verified against the live base via the Meta API.)

## Encoding rule — curl only for reads

**Use curl with `--data-urlencode` for every read (filterByFormula, fields, sort).** Do NOT use Python `urllib.parse.urlencode` for reads — it silently mis-encodes Airtable formula syntax (`AND(...)`, `SEARCH(...)`) and Hebrew characters, producing 422s that are very hard to diagnose. Python (`requests` / `pyairtable`) is fine for **write** operations only. All templates in `references/query-patterns.md`.

## Gotchas
- **Never use Python urllib for Airtable formula queries** — use curl with `--data-urlencode`. Python urllib silently mis-encodes `AND(...)`, `SEARCH(...)`, and Hebrew characters → 422.
- **Lookup/formula/rollup fields cannot use `=`** — wrap with `SEARCH('val', {field})` or `FIND('val', ARRAYJOIN({field}))`. Example: `reports.client_id` is a lookup array → use `FIND('CPA-210', ARRAYJOIN({client_id}))`. (`pending_classifications.client_id` is plain text — `=` is fine there; `clients.client_id` is the formula.)
- **Linked record fields return arrays** — use `FIND(id, ARRAYJOIN({field}))` in formulas, or access `[0]` in code. Bare `{linked}=...` will not match. Prefer human-readable lookup fields (e.g. `report_key_lookup`) over raw record-ID linked fields — see the pattern + real 2026-05-02 incident in `references/query-patterns.md`.
- **Sort field name in this base is `created_at`**, not `Created`. Confirm sort fields against the schema probe before running. Per-table sort fields listed in `references/schema-quick-reference.md`.
- **Env var name:** `AIRTABLE_API_KEY` (not `AIRTABLE_PAT`)
- **Single select fields:** Can't create new options via Meta API easily — use `typecast: true` on record create instead
- **`fields[]` param:** Use `--data-urlencode "fields[]=name"` (not `fields=name`)
- **Delete max 10 per request** — batch larger deletes
- **Table IDs live in `config.env`** — reference the vars (`$CLIENTS_TID`, `$REPORTS_TID`, `$DOCUMENTS_TID`, `$PENDING_TID`, `$TEMPLATES_TID`), never raw `tbl…` literals. Full table→var map in `references/schema-quick-reference.md`.

## Privacy Rules (ALWAYS FOLLOW)

See [privacy.md](privacy.md) for complete rules. Key points:

1. **Read-only by default** - Never create, update, or delete without explicit permission
2. **Minimal data** - Only fetch what's needed
3. **No token display** - NEVER echo or display the API key
4. **Summarize, don't dump** - Format responses cleanly

## Decision gates

- **Schema gate (before any filtered read):** confirm field names via `references/schema-quick-reference.md`, `docs/airtable-schema.md`, or a no-filter probe. No guessing.
- **Write gate:** never `create`, `update`, `upsert`, or `delete` without explicit user permission for that operation.
- **422 gate:** on a 422, STOP and run the recovery sequence (`references/query-patterns.md`) — do not retry the same query with tweaks.
- **Encoding gate:** filtered reads go through `curl --data-urlencode`. Python `urllib` for reads is forbidden (silent mis-encoding of `AND/SEARCH` and Hebrew).
- **Delete batch gate:** Airtable rejects >10 record IDs per DELETE — chunk into 10s.
- **Token-display gate:** never echo, log, or print `AIRTABLE_API_KEY`.

## Output format

Format as clean tables:

**Good:**
```
Records in Tasks:
┌──────────────────┬──────────┬────────────┐
│ Name             │ Status   │ Due Date   │
├──────────────────┼──────────┼────────────┤
│ Review proposal  │ Active   │ Jan 20     │
│ Send report      │ Done     │ Jan 18     │
└──────────────────┴──────────┴────────────┘
```

**Bad:**
```json
[{"id":"rec123","fields":{"Name":"Review proposal"...
```

## References

- `references/schema-quick-reference.md` — verified field names, enum values, and sort fields per table in your CRM base. Read it to satisfy HARD GATE option 1.
- `references/query-patterns.md` — all request templates: canonical curl reads, write/delete/Meta-API calls, multi-step linked-field patterns (documents-for-a-client), the 422 recovery sequence, pyairtable patterns, and first-time setup. Load at workflow step 4.
- [api-reference.md](api-reference.md) - All Python patterns
- [Privacy Rules](privacy.md) - Data handling guidelines

Sources: [pyAirtable Documentation](https://pyairtable.readthedocs.io/en/stable/), [GitHub](https://github.com/gtalarico/pyairtable)

## Evaluation checklist

After running an Airtable operation, confirm:

- Was the schema gate satisfied (`references/schema-quick-reference.md` / `docs/airtable-schema.md` / no-filter probe) before any filtered read?
- Did filtered reads use `curl --data-urlencode` (not Python `urllib`)?
- Were formula/rollup/linked fields wrapped in `SEARCH` / `FIND` / `ARRAYJOIN` instead of bare `=`?
- Was `created_at` (not `Created`) used for sort?
- For writes/deletes: did the user explicitly approve the operation?
- Were deletes chunked into batches of ≤10 IDs?
- Was the API key kept out of all output (no echo, no logs, no error dumps)?
- Was the result rendered as a clean table/summary, not a raw JSON dump?

---
name: design-log
description: 'Plan non-trivial features, fixes, refactors, UI/workflow changes before coding — clarify, research, get approval (Stop & Think protocol).'
model: opus
allowed-tools: Read, Write, Edit, Glob, Grep, Bash, mcp__claude_ai_Tavily__tavily_search, mcp__claude_ai_Tavily__tavily_extract, mcp__claude_ai_Tavily__tavily_crawl, mcp__claude_ai_Tavily__tavily_map, mcp__exa__web_search_exa, mcp__exa__web_fetch_exa, mcp__exa__web_search_advanced_exa, mcp__firecrawl__firecrawl_search, mcp__firecrawl__firecrawl_scrape, mcp__firecrawl__firecrawl_crawl, mcp__firecrawl__firecrawl_extract, Agent, Skill, EnterPlanMode, ExitPlanMode, AskUserQuestion
---

# Design Log Skill

Generic "Stop & Think" protocol that combines **research-driven design** with Claude Code's plan mode. Research first, then code. One workflow, one approval gate, persistent documentation. Project-specific rules belong in repo docs or in clearly labeled adaptation notes inside this skill.

## How This Integrates With Plan Mode

Research (domain knowledge) comes BEFORE plan-mode codebase exploration; the design log file persists what plan mode would lose; both share a **single approval gate** via `ExitPlanMode` — no double-approval.

## When this triggers

Use when the user explicitly invokes `/design-log`, asks for stop-and-think planning, or asks for documented design before a non-trivial feature, fix, refactor, workflow/UI change, automation, or architecture decision. Implicit use only when the task clearly needs persistent design documentation.

## When this does not trigger

Skip for: single-line fixes and small tweaks; fully-specified unambiguous tasks where the user didn't invoke `/design-log`; pure research questions (use `tech-researcher` or `consult`); work already covered by an existing `[APPROVED]` log (extend that log instead). For these, just do the work — though a quick 2-minute research check is still wise for non-trivial domains.

## Required Inputs

Before writing the design log, identify: project root + branch/worktree state; the requested change and success criteria; relevant prior logs and prior research (research index); the domain(s) needing research; reusable code paths/helpers/tests/docs/diagrams; whether to implement after approval or stop at the plan. If an input is missing, ask via `AskUserQuestion` unless logs, instructions, or pre-scan findings already answer it.

## Decision Gates

Stop and wait when:

- The Phase A relevance grep (Pre-Phase-A step 1 / A1) has not been run and related prior DLs surfaced to the user — do not proceed to A3 clarifying questions without it.
- The user has not answered required Phase A questions.
- Tavily research is incomplete.
- `EnterPlanMode` has not been called before Phase C exploration.
- The design log has not been saved with `[DRAFT]` status.
- `ExitPlanMode` approval has not been granted.
- The next action would merge to `main`, delete a branch, change secrets, perform destructive operations, or run an undocumented production-impacting command.

Continue without another approval only after `ExitPlanMode` approval, and only for routine implementation decisions inside the approved plan.

## Workflow (Protocol)

The protocol runs in five phases. Each phase summary below is enough to know what to do; the full step-by-step procedure for every phase lives in `references/protocol-detail.md` — load it when you actually enter the phase.

### Phase A Is Non-Negotiable When User Invokes `/design-log`

If the user explicitly typed `/design-log` (or otherwise invoked this skill by name), Phase A clarifying questions are **mandatory** — Auto Mode, "specific instructions", and "seems simple" are NOT valid reasons to skip. The user asked for the Stop & Think protocol; give them the Stop & Think protocol. Silently jumping to implementation is a protocol violation.

The "When this does not trigger" section above applies to deciding whether to invoke the skill in the first place — NOT to partially running it. If you run the skill, you run Phase A. If you believe the task genuinely doesn't warrant Phase A, say so out loud before proceeding: "Skipping Phase A questions because [specific reason] — proceed or ask questions anyway?" and wait for the user.

### Auto Mode Activates Only After Approval

Auto Mode's "prefer action over planning / minimize interruptions" directive applies **only to Phases D–E** — after `ExitPlanMode` approval. During Phases A–C it is effectively **suspended**: ask the Phase A questions and wait; do the Phase B research even if the task "seems simple"; exit plan mode and wait for approval — never self-approve. After approval, Auto Mode resumes: execute the plan end-to-end, stopping only at the hard gates (merge to main, destructive ops, etc.).

### Model Routing — Don't Spend Opus on Gathering

This skill is pinned to Opus (frontmatter `model: opus`) for **judgment** work (clarifying questions, design synthesis, the plan). Mechanical **gathering** must NOT run on Opus — delegate it. This obeys the global model-routing rule and cuts ~50%+ of run cost with no loss of plan quality:

- **Pre-Phase-A check** → the relevance grep / INDEX scan / unfinished-log report is gathering: dispatch `explore` or `model="haiku"`, reason over the summary. The dispatched prompt **must include the task-derived search terms** (incl. Hebrew UI labels) and **must return a "Related prior DLs" list** (number + title + status + 1-line relevance). Do NOT request a "highest number + last N logs" shortcut — that misses the relevant logs.
- **Phase A scan + pre-scan** → dispatch an `explore` or `model="haiku"` subagent to read `INDEX.md` (+ any archive index), grep log bodies, and pre-scan the codebase. Cap: return ≤500 words, read at most ~15 files, conclusions only.
- **Phase B research** → dispatch a `model="sonnet"` research subagent (or `tech-researcher`). Cap: 3–5 sources, return ≤700 words of distilled principles/patterns/anti-patterns — do NOT pull full pages into the Opus context.
- **Phase C exploration** → use `explore` / `model="haiku"` subagents for read-only file discovery. Cap: return file paths + 1-line relevance each, not file contents. Keep the design synthesis itself on Opus.
- **Keep on Opus:** Phase A clarifying questions, the design write-up, the plan-mode plan, Phase D implementation decisions.
- **Summary-only rule:** subagents return conclusions, never raw file dumps or full page text — the design log file is the persistent artifact, not the conversation.

**Fable 5 note (since 2026-06-09):** Fable 5 burns subscription quota ~2× faster than Opus 4.8 for the same tokens ($10/$50 vs $5/$25 per MTok weighting). The `model: opus` pin above is deliberate — it keeps this skill on Opus 4.8 even when the session model is Fable 5. Do NOT remove the pin or escalate to Fable 5 by default. When dispatching subagents while the session runs on Fable 5, an explicit `model=` is mandatory (haiku/sonnet per the rules above) — an unpinned dispatch would inherit Fable 5 and pay 2× for gathering. Reserve Fable 5 for the rare design problem where the user explicitly asks for it.

### Lite Mode (`/design-log lite`)

When the user types `/design-log lite` (or asks for a "light/quick design log"), run the same protocol with reduced volume: Phase A asks 2–3 clarifying questions (not 5+); Phase B needs 1 source minimum — a fresh research-index hit (<90 days) satisfies it with zero new searches; template Sections 3–5 may be compressed to a few bullets each. **All hard gates still apply:** branch setup (A0), `EnterPlanMode` before exploration, the DL file saved with `[DRAFT]` status, `ExitPlanMode` single approval, Phase E handoff. Per-phase deltas table in `references/protocol-detail.md` § Lite Mode.

### Phase A — Discovery

**STOP. Do not implement anything.** Steps: A0 Branch Setup → A1 Check Existing Logs → A2 Light Codebase Pre-Scan → A3 Ask Clarifying Questions via `AskUserQuestion` (usually 5+) → A4 Wait for answers. Full procedure in `references/protocol-detail.md` § Phase A.

> **A3 questions must be in simple English (MANDATORY).** Assume the person answering is the product owner, not a specialist in every domain the log touches, and may not be a native English speaker. Every question, option label, and option description uses plain everyday words and **no unexplained jargon or acronyms** (`SSR`, `debounce`, `idempotent`, `optimistic update`, `webhook`, `race condition`, framework/pattern names). Ask about the **visible effect** — what the user or the data will experience — not the technique that produces it. One decision per question; state the trade-off in each option's description. If a term is truly unavoidable, gloss it in plain English on the spot. Before sending, re-read each question: could someone outside this codebase answer it correctly? If not, rewrite. **This applies to the questions only** — the design log, research, and plan stay technical. Rewrite examples in `references/protocol-detail.md` § A3.

> **A0 DL-number reservation (if your repo uses it):** some repos reserve the design-log number with a script that lives **outside the repo** (so it survives repo cleanups) — e.g. `~/.claude/scripts/reserve-dl-number.sh`. If your setup has one, run it from the repo root and do NOT improvise raw `git push` claims. Configure the script path and per-repo rules (branch rename vs `slice/*`, default branch) in `references/protocol-detail.md` § Phase A A0 — load it before reserving. If no script exists, fall back to the manual numbering in that reference.

### Phase B — Research

**Mandatory.** B0 Fetch current date via `Bash` (`date +%Y-%m-%d`) BEFORE any MCP call → B0.5 Check `.agent/design-logs/research-index.md` — a domain entry <90 days old means delta research only → B1 Identify domain → B2 Read 3+ sources via a research MCP — Tavily / Exa / Firecrawl (NOT `WebSearch`/`WebFetch`) → B3 Extract principles, patterns, anti-patterns, deviations into Section 3 of the log + upsert the research index. Time-box 5–10 min. Optional: delegate to `tech-researcher` for fast-moving technical domains. Full procedure + research rules + domain table in `references/protocol-detail.md` § Phase B.

> **HARD GATE — DATE FIRST:** Before calling Tavily (or any other MCP/research tool — `tech-researcher`, etc.), run `Bash` with `date +%Y-%m-%d` to anchor "current" in real time. The session's `currentDate` context can be stale or absent. Do NOT rely on training-data dates or assumed year. Then use the anchored date BOTH ways:
> - **Set Tavily's structured filter parameters** — `start_date` (and/or `end_date` / `time_range`) computed from the anchored date. This is the lever that actually *filters out* stale pages. Putting a date in the query text alone only nudges ranking; it does NOT filter.
> - **And** mention recency in the query text where useful ("as of YYYY-MM-DD", "latest stable", "deprecated in YYYY") and pick correct year filters (current year + prior year, not years recalled from memory).

> **Pick the research MCP by job** (NOT `WebSearch`/`WebFetch`): Exa = semantic source discovery + deep research · Tavily = ranked web search + extraction · Firecrawl = full-page/JS-heavy/site crawls. Per-tool breakdown in `references/protocol-detail.md` § Phase B B2.

> **HARD GATE:** If you invoked `tech-researcher`, you MUST wait for its result before calling `EnterPlanMode`. Do NOT enter Phase C while tech-researcher is still running.

### Phase C — Explore & Document

> **HARD GATE — READ BEFORE ANY TOOL CALL IN THIS PHASE:**
> The FIRST tool call in Phase C **must** be `EnterPlanMode`. Do NOT run Glob/Grep/Read for codebase exploration, and do NOT call `Write`/`Edit` on the design log file, until plan mode is active. The design log file must be created INSIDE plan mode. Phase C ends with `ExitPlanMode` — a plain-text "approve to proceed?" message is NOT a substitute approval gate. If you catch yourself mid-exploration without having called `EnterPlanMode`, stop immediately, call it, and resume — do NOT rationalize continuing outside plan mode because "exploration already started."

Steps: enter plan mode → explore codebase + load architecture diagrams → create the design log file at `.agent/design-logs/NNN-kebab-case-description.md` from `assets/design-log-template.md` with status `[DRAFT]` → `ExitPlanMode` for the single approval. Full procedure in `references/protocol-detail.md` § Phase C.

### Phase D — Implementation (After Approval Only)

**Recommended:** Use `/subagent-driven-development` for implementation when the plan has independent tasks that can run in parallel.

Steps: status → `[BEING IMPLEMENTED — DL-NNN]` → build per Proposed Solution → update Implementation Notes on deviations → status → `[IMPLEMENTED — NEED TESTING]` → housekeeping (INDEX, `current-status.md`, architecture diagrams, `git-ship` skill, deploy if applicable). Full housekeeping checklist (NO auto-merge, frontend release path, do-not-delete-branch, do-not-edit-after-merge) in `references/protocol-detail.md` § Phase D.

### Phase E — Test Handoff

Collect unchecked Section 7 items → write to `current-status.md` Active TODOs in the documented format → mark log `[COMPLETED]` only when all Section 7 items pass. To close: run `bash .claude/workflows/close-design-log.sh <NNN>` — it patches status in both the DL file and INDEX.md, runs the PII guard, and stages the files. Once `[COMPLETED]`, merged, and (if deployed) live-verified, **delete the feature branch** via `git-ship`: verify-merged → detach-if-checked-out → `git branch -D` + `git push origin --delete` (Phase E step 5). Full procedure + format template in `references/protocol-detail.md` § Phase E.

### Handling Mid-Implementation Feedback

Classify feedback (clarification / bug / missed constraint / scope expansion / refinement) and respond accordingly. Full table in `references/protocol-detail.md` § Handling Mid-Implementation Feedback. **When uncertain:** state your assumptions, propose options (quick fix vs. proper solution), ask for preference. Don't guess.

## Output format

The persistent artifact this skill produces is a design log file at `.agent/design-logs/NNN-kebab-case-description.md`, filled in from the canonical template at `assets/design-log-template.md`. Read that file when you need the structure; do not paste it into chat.

Sections in order: 1. Context & Problem · 2. User Requirements (Q&A) · 3. Research (Domain, Sources, Principles, Patterns, Anti-Patterns, Verdict) · 4. Codebase Analysis · 5. Constraints & Risks · 6. Proposed Solution · 7. Validation Plan · 8. Implementation Notes.

Two subsections are **conditional** — fill them when the change warrants, else mark them omitted/None: §6 *Boundary Contracts* (define the validated schema crossing each module/service/agent boundary — prevents cascading shape-drift errors on multi-module changes) and §7 *Trajectory / Acceptance Criteria* (verify the path, not just the final answer — for non-deterministic or high-stakes output like LLM text or recommendations).

Status values (`[DRAFT]` / `[APPROVED]` / `[BEING IMPLEMENTED — DL-NNN]` / `[IMPLEMENTED — NEED TESTING]` / `[COMPLETED]` / `[DEPRECATED]`) and naming convention (`NNN-description.md`, sequential, lowercase-hyphens, English) are documented in `references/protocol-detail.md`.

Do not leave template placeholders such as `[Question]`, `[Key takeaway]`, or `Test Case 1` in a real design log. Replace every placeholder with concrete project-specific content before calling `ExitPlanMode`.

## Critical Rules

1. **NEVER implement before approval** — `[DRAFT]` means no coding.
2. **ALWAYS ask clarifying questions first, in simple English** — understanding before action, and the user must be able to understand the question. No unexplained jargon or acronyms (see § Phase A). If the user invoked `/design-log` explicitly, Auto Mode does NOT override this. To skip, state the reason out loud and wait for confirmation.
3. **ALWAYS research before designing** — knowledge before architecture.
4. **ONE approval gate** — `ExitPlanMode` is the single approval for both plan and design log.
5. **ALWAYS save a design log file** — plan mode is ephemeral, the log persists.
6. **ALWAYS update Implementation Notes** — document deviations and which research principles were applied.
7. **ALWAYS run Phase E** — persist test items to `current-status.md` and mark `[COMPLETED]` only once all tests pass.

## Pre-Phase-A Check (when this skill is invoked)

This skill does NOT run automatically at session start. It runs when the user types `/design-log`, invokes it by name, or (rarely) when the task clearly needs persistent design documentation per "When this triggers". When invoked, before Phase A:

1. **Relevance scan (PRIMARY, mandatory).** Derive 3–6 search terms from the task — symptom words, feature name, affected file/symbol, error string, **and any Hebrew UI label** — and grep them across `INDEX.md` + `ARCHIVE-INDEX.md` + **all DL bodies** (not just titles). Read in full any DL on the same feature / symptom / files and list them (number + title + status + 1-line relevance). This is how duplicates and prior decisions are found.
2. **Unfinished-work check.** Report any `[APPROVED]` / `[BEING IMPLEMENTED]` logs not marked `[COMPLETED]` (conflict guard).
3. Note domains already researched — read `research-index.md` (for the Phase B cache rule).

> **Recency is NOT a discovery method.** "Summarize the last N logs" does NOT satisfy step 1 — the most recent logs are rarely the relevant ones. The relevance grep is required; do not substitute a recency summary for it.

This check is gathering — per § Model Routing, dispatch it to `explore`/`model="haiku"` and reason over the summary.

## Gotchas

Full list in `references/protocol-detail.md` § Gotchas — read when closing out a phase or when something feels off. The ones that bite most often:

- Skipping Phase A questions because the task "seems simple" or Auto Mode is on — protocol violation; see § Phase A Is Non-Negotiable.
- Exploring or writing the design log before `EnterPlanMode` — the first Phase C tool call must be `EnterPlanMode`.
- Calling research MCPs before anchoring the date with `Bash date +%Y-%m-%d`, or putting the date only in query text instead of Tavily's structured filters.
- Substituting a "last N logs" recency summary for the Phase A relevance grep.
- Running gathering (grep/scan/research) on Opus or Fable 5 instead of dispatched haiku/sonnet subagents.

## Evaluation checklist

Run after the skill completes — any "no" means a skipped phase (full version: `references/protocol-detail.md` § Evaluation Checklist):

- Was the Pre-Phase-A relevance grep dispatched and its "Related prior DLs" list surfaced?
- Were Phase A clarifying questions asked and answered (or an explicit skip confirmed by the user)?
- Were the Phase A questions written in simple English — no unexplained jargon or acronyms, and answerable by someone outside this codebase?
- Was Phase B research done via a research MCP with a date-anchored filter, and the research index updated?
- Was `EnterPlanMode` the first Phase C tool call, and the DL file saved as `[DRAFT]` inside plan mode?
- Was `ExitPlanMode` the single approval gate — no coding before it, no second approval after it?
- Are all template placeholders replaced with concrete content?
- Did Phase E persist unchecked Section 7 items to `current-status.md` before any `[COMPLETED]` mark?

## References

- `references/protocol-detail.md` — load when entering a phase and you need the full step-by-step procedure (Phase A0–A4, B0.5–B3 incl. research-index cache, C, D housekeeping, E test handoff + cost line, mid-implementation feedback table, status values, naming convention, Lite Mode deltas, Gotchas, Evaluation Checklist).
- `references/research-sources.md` — load in Phase B (B2) when picking book recommendations and source tiers for the identified domain.

## Assets

- `assets/design-log-template.md` — fill this out as the persistent design log file in Phase C; read it for the canonical Section 1–8 structure rather than reproducing it inline.
- `assets/research-index-template.md` — seed for `.agent/design-logs/research-index.md` (per-project research cache); create from this in Phase B3 if the project doesn't have one yet.

# Design-Log Protocol — Detailed Phases

Load this file when entering a phase and you need the full step list. SKILL.md keeps the router, hard gates, and one-line phase summaries; everything below is the elaborated procedure.

---

## Phase A: Discovery

**STOP. Do not implement anything.**

### A0 — Branch Setup (MANDATORY)

Detect environment first:

- Run `git rev-parse --git-dir`, `git branch --show-current`, and `git status` before changing files.
- If you are in a worktree, verify the branch matches the current task and continue.
- If you are on `main`/`master`, do not implement there. Ask the user whether to create/use a feature branch or worktree, then stop until the branch/worktree is safe.
- If another active feature branch appears unrelated to this task, do not switch branches without user approval.

Project-specific branch / log-number automation:

- **If your repo uses a reservation script:** keep it **outside** the repo (e.g. `~/.claude/scripts/reserve-dl-<repo>.sh`) so it survives repo cleanups, and run it **from the repo root** — a well-written reserve script is repo-agnostic, operating on `git` in the cwd. A robust script handles: atomic `max()` across remote `refs/dl-claims/*` + `INDEX.md` (+ `ARCHIVE-INDEX.md` if present) + existing `NNN-*.md` filenames, retry-on-collision (e.g. up to 30 attempts), default-branch auto-detect (don't assume `main`), and — on Windows/MSYS — the `MSYS_NO_PATHCONV=1` fix. Do NOT search *inside* the repo and conclude "no reserve script exists," and do NOT improvise raw `git push` claims when a script is configured. After reserving `NNN`, follow your repo's branch convention — some rename the branch (`git branch -m DL-NNN-short-description`), others keep a fixed scheme (e.g. `slice/<short-name>`); reservation governs only the DL file number.
- **Manual fallback (no script):** claim the next number with `git push origin HEAD:refs/dl-claims/NNN` where `NNN = max(remote refs/dl-claims/*, INDEX.md, existing filenames) + 1`, then create `.agent/design-logs/NNN-<desc>.md` and its `DL-NNN` INDEX row.
- For other repos: if the repository documents a design-log reservation script, worktree launcher, or branch naming convention, follow that repo-specific workflow.
- If no reservation automation exists, determine the next log number from `.agent/design-logs/INDEX.md`, `ARCHIVE-INDEX.md`, and existing filenames, then warn about possible collision in parallel sessions.

### A1 — Check Existing Logs (MANDATORY — do not skip)

> **Cost discipline:** A1 + A2 are pure retrieval. Dispatch an `explore` or `model="haiku"` subagent to do the scan/grep/pre-scan and return a findings summary; reason over that summary on the main (Opus) thread rather than reading every log and file on Opus. See SKILL.md § Model Routing.

- Read `.agent/design-logs/INDEX.md` and `ARCHIVE-INDEX.md` first — these are the authoritative catalogs.
- Identify the relevant domain folder(s) under `.agent/design-logs/` (e.g. `ui/`, `api/`, `documents/`, `email/`, `infrastructure/`, `security/`, `research/`, or project-specific equivalents) and list their contents — a top-level scan alone will miss everything.
- **Relevance grep (mandatory, non-skippable):** derive 3–6 search terms from the task — symptom words, feature name, affected file/symbol, error string, **and any Hebrew UI label** — and grep them across `INDEX.md` + `ARCHIVE-INDEX.md` + **all DL bodies** across every domain folder. Filename/title triage is not enough. **Recency ("the last N logs") is not a substitute for this — the most recent logs are rarely the relevant ones.**
- **Read in full** any log whose title, index entry, or grep hit looks related — do not rely on filenames alone. Cite related logs in the new DL's "Related Logs".
- If a related log exists: extend it, or explicitly justify creating a new one.
- **Surface findings to the user before asking discovery questions** (e.g., "DL-161 already decided X — does this still apply?") — prior decisions may change the requirements or make new questions unnecessary.
- Check for any `[APPROVED]` or `[BEING IMPLEMENTED]` logs not yet `[COMPLETED]` (unfinished work that may conflict).
- **Check for prior research** on the same domain — read `.agent/design-logs/research-index.md` first (one small file instead of grepping log bodies for research sections). If the domain has a row there, reference the prior DL and plan for delta research only (see Phase B, B0.5).

### A2 — Light Codebase Pre-Scan (Reuse First)

Before designing anything, do a quick reuse scan to find existing solutions in the codebase. This is not full plan-mode exploration; it is a bounded check to avoid asking questions or writing plans that ignore obvious existing work:

- Search for functions, modules, utilities, or patterns similar to what's being requested (Grep/Glob).
- Look for partial implementations that could be extended rather than replaced.
- Identify shared helpers or components already doing related work.
- **Document findings** in a short "Existing Solutions" note: what exists, what it does, and whether it can be reused/extended.
- **Default to reuse/extension** — only design from scratch if nothing relevant exists.
- If something relevant is found: surface it to the user before asking clarifying questions.

### A3 — Ask Clarifying Questions

- **ALWAYS use the `AskUserQuestion` tool** with predefined options for each question — never ask questions as plain text.
- Usually ask 5+ questions, but fewer is acceptable when prior logs, user instructions, or pre-scan findings already answer the remaining decisions.
- Each question should have 2–4 concrete options with descriptions.
- Mark the recommended option with "(Recommended)" suffix.
- Questions must be specific, not generic.
- Bad: "How do you want this?"
- Good: "When someone changes a task's status, should the update run right away, or once every hour?"

#### Write the questions in simple English (MANDATORY)

Assume the person answering is the product owner, not a specialist in every domain the log touches, and possibly not a native English speaker. A question they cannot decode is a question they cannot answer — it produces a guessed answer, and the whole design is then built on that guess.

Rules for **every** question, option label, and option description:

- **Plain, everyday English.** Short sentences. Common words. Say "runs by itself" instead of "is triggered idempotently."
- **No unexplained jargon or abbreviations** — not framework names, not pattern names, not acronyms (`SSR`, `CQRS`, `debounce`, `idempotent`, `race condition`, `optimistic update`, `webhook`). If a term is genuinely unavoidable, write it once with a plain-English gloss right after it, e.g. "a webhook (a message one system sends another the moment something happens)".
- **Describe the visible effect, not the implementation.** Ask what the user or the data will experience, not which technique produces it.
- **One decision per question.** Never bundle two choices into one.
- **Say the trade-off in each option's description** — what he gains, what it costs. One short sentence, in the same plain English.
- **Self-check before sending:** re-read each question and ask "could someone outside this codebase, with no background in this domain, answer it correctly?" If no, rewrite it.

Rewrite examples:

| ❌ Jargon | ✅ Simple English |
|---|---|
| "Should the sync be optimistic or pessimistic?" | "When you press Save, should the screen update instantly and fix itself if the save fails, or wait until the save is confirmed?" |
| "Debounce the search input or fire on every keystroke?" | "Should the search run on every letter you type, or wait until you stop typing for a moment?" |
| "Do we need idempotency keys on the webhook handler?" | "If the same message arrives twice, should the system ignore the copy, or process it again?" |
| "Soft-delete or hard-delete the record?" | "When a row is deleted, should it be hidden but recoverable, or erased for good?" |

This rule applies to the questions **only**. The design log itself, the research section, and the plan stay technical and precise — do not dumb those down.

### A4 — Wait for user answers before proceeding

---

## Phase B: Research

**After user answers discovery questions, BEFORE entering plan mode.** This phase is mandatory. Even if the task seems simple, research it — you often discover better approaches.

### B1 — Identify the Domain

Determine the technical/UX/architectural domain(s) the request falls into. Examples:

| Request | Domain(s) |
|---------|-----------|
| "Add error handling" | Resilience Engineering, UX Error States |
| "Build a search feature" | Information Retrieval, Search UX |
| "Fix a race condition" | Concurrency, Distributed Systems |
| "Add authentication" | Security, Identity Management |
| "Improve performance" | Web Performance, Optimization |
| "Build a form" | Form Design, Validation Patterns, Accessibility |
| "Add notifications" | Messaging Patterns, Notification UX |
| "Fix layout issues" | CSS Architecture, Responsive Design |
| "Build a workflow" | Workflow Orchestration, State Machines |
| "Add email automation" | Email Deliverability, Transactional Email Patterns |
| "Process documents with AI" | Document AI, Classification Patterns |

Not exhaustive — use judgment for the specific task.

### B0.5 — Check the Research Index (cache hit = delta research only)

Before any search, read `.agent/design-logs/research-index.md`:

- **Entry < 90 days old for this domain** → do **delta research only**: 1 targeted search asking "what changed since [date] / what didn't the prior log cover for this specific task?", and cite the prior DL in Section 3 (`**Prior research:** DL-NNN ([domain], YYYY-MM-DD) — delta only`).
- **Entry > 90 days old** → re-validate the key claims (quick search confirming the principles still hold) before reusing them, then proceed with whatever delta research the task needs.
- **No entry / file missing** → full B2 below.

### B2 — Research Sources (3+ minimum)

> **Cost discipline:** Reading 3+ full sources is token-heavy. Run the searches/extraction in a `model="sonnet"` research subagent (or `tech-researcher`) and fold back only its **distilled** findings — 3–5 sources, **≤700 words** of principles/patterns/anti-patterns, never full page text. Avoid loading multiple full pages into the main (Opus) context. See SKILL.md § Model Routing.

Use a **research MCP** to find and read substantive content from **at least 3 sources** across these tiers. Do NOT use built-in `WebSearch`/`WebFetch` — they are intentionally not in `allowed-tools`. Pick the tool by job (use whichever is installed; prefixes follow your MCP server names):

**Tavily** — general ranked web search + extraction:
- `mcp__claude_ai_Tavily__tavily_search` — web search returning ranked results with content snippets. One query per call; issue parallel calls in a single message for multiple independent searches.
- `mcp__claude_ai_Tavily__tavily_extract` — fetch full content of one or many URLs as Markdown (accepts a `urls` array — native batch).
- `mcp__claude_ai_Tavily__tavily_crawl` — crawl a site starting from a root URL when traversing linked pages.
- `mcp__claude_ai_Tavily__tavily_map` — get a sitemap-style structural map of a site without fetching content.

**Exa** — semantic/neural source discovery; best for "find the best writing on X" rather than keyword match:
- `mcp__exa__web_search_exa` — neural web search tuned for high-quality, semantically-relevant sources.
- `mcp__exa__web_fetch_exa` — fetch full page content for a known URL.
- `mcp__exa__web_search_advanced_exa` — advanced search with full control over filters, domains, dates, and content options.

**Firecrawl** — full-page extraction, JS-heavy sites, and crawling a whole docs site:
- `mcp__firecrawl__firecrawl_search` — search + scrape in one.
- `mcp__firecrawl__firecrawl_scrape` — clean markdown from a single page (handles JS rendering).
- `mcp__firecrawl__firecrawl_crawl` — crawl linked pages from a root.
- `mcp__firecrawl__firecrawl_extract` — structured extraction from one or many pages.

**Tier 1 — Books:** Find the 1–2 most respected books in the identified domain. Search for core concepts, chapter summaries, key takeaways.
**Tier 2 — Authoritative articles & documentation.**
**Tier 3 — Real-world case studies.**

For book recommendations and source tiers by domain, see `references/research-sources.md`.

**Efficiency rule:** Time-box research to 5–10 minutes of tool calls. Read the most relevant sections, not entire books. Use `Task` with parallel agents for independent searches when useful.

**Optional delegation:** For fast-moving technical domains (AI/ML tooling, agent frameworks, LLM SDKs, JS/TS frameworks, cloud, DevOps, package status checks), you MAY invoke the `tech-researcher` skill via the `Skill` tool to get a formal Recommendation / Current Evidence / Risks / Implementation Notes report, then fold its output into Section 3 of the design log. Use this for technical-tool research; keep the inline Tavily flow for UX, architecture, and domain-pattern research where the book/principles framing matters more than current docs. Do not use `Task` or `Skill` for routine research when a direct Tavily search is sufficient.

> **HARD GATE:** If you invoked `tech-researcher`, you MUST wait for its result before calling `EnterPlanMode`. Do NOT enter Phase C while tech-researcher is still running.

### B3 — Extract & Document Findings

Synthesize research into actionable findings (documented in Section 3 of the design log template). Focus on:

- **Principles** that directly apply to our specific case.
- **Patterns** we should use (with brief description of how).
- **Anti-patterns** to avoid (and why they're tempting but wrong for us).
- **Deviations** — if a source recommends something that doesn't fit our constraints, note the recommendation AND why we're deviating.

**Update the research index:** after research, upsert the domain row in `.agent/design-logs/research-index.md` — domain, DL number(s), today's date, a 1-line principles summary, top 1–3 sources. If the file doesn't exist, create it from `assets/research-index-template.md`.

**Cross-project principles:** if a principle is project-agnostic (a domain truth, not tied to this codebase/stack), also save it to auto-memory (`~/.claude/projects/<your-project>/memory/`) as a `reference`-type memory plus a MEMORY.md index line, so other projects benefit. Only genuinely reusable principles — don't memorize project-specific findings.

### Research Rules

1. **No skipping.** Even if the task seems simple or familiar, do the research.
2. **Quality over quantity.** Reading 3 excellent sources deeply beats skimming 10.
3. **Be specific to our context.** Don't quote generic advice — explain how each principle applies to this feature/bug.
4. **Time-box it.** 5–10 minutes of tool calls.
5. **Cumulative knowledge.** If you've already researched a domain in a previous design log, reference it (`See design-log NNN for prior research on [domain]`) and add only new/incremental findings. Tavily research is still mandatory for explicit `/design-log` runs; in already-researched domains, satisfy it with targeted delta research that checks what has changed or what the prior log did not cover.
6. **Disagree with books when warranted.** Note the recommendation AND why we're deviating.

---

## Phase C: Explore & Document

After research is complete:

1. **Enter Plan Mode** — call `EnterPlanMode` to begin codebase exploration.
2. **Explore the codebase** using Glob, Grep, Read to understand:
   - **Load relevant architecture diagram(s)** from repo docs when present (e.g., `docs/architecture/`) to understand how the feature area fits into the overall system before exploring individual files.
   - Existing patterns and architecture relevant to the task.
   - Files that will need to change.
   - Dependencies and potential risks.
   - **How research findings map to our actual codebase** (do existing patterns align with best practices? where do they diverge?).
3. **Create the Design Log File:**
   - Determine next log number — if the branch was already renamed `DL-NNN-*` in A0, use that NNN. If the repository documents a reservation script, run it from the documented repo root. Otherwise scan `.agent/design-logs/INDEX.md`, `ARCHIVE-INDEX.md`, and existing filenames, then warn about possible collision in parallel sessions.
   - Path: `.agent/design-logs/NNN-kebab-case-description.md`.
   - Fill in the canonical template (`assets/design-log-template.md`) using findings from **both research and exploration**.
   - Set status to `[DRAFT]`.
   - Write the plan-mode plan content AND save the design log file.
4. **Exit Plan Mode** — call `ExitPlanMode` to present the plan for approval.
   - This is the **single approval gate** for both the plan and the design log.
   - User reviews and approves (or requests changes).

---

## Phase D: Implementation (After Approval Only)

**Recommended:** Use `/subagent-driven-development` for implementation when the plan has independent tasks that can run in parallel.

1. **Update design log status to `[BEING IMPLEMENTED — DL-NNN]`** (replace NNN with the actual log number).
2. **Build strictly according to "Proposed Solution".**
3. **Reference research in code comments where relevant** (e.g., `// Pattern: Circuit Breaker — see design-log NNN`).
4. **Run any validation steps that can be verified immediately** (build passes, workflow deploys, no errors in logs).
5. **Update "Implementation Notes" section if any deviations occur.**
6. **Update design log status to `[IMPLEMENTED — NEED TESTING]`.**
7. **Housekeeping (always the last task in every implementation plan):**
   - Update design log status → `[IMPLEMENTED — NEED TESTING]`.
   - Update design log INDEX.
   - Update `.agent/current-status.md`: summarize what was done this session, add next-session TODOs (including unchecked Section 7 items), remove completed items from existing TODOs.
   - If the implementation added/removed/changed workflows, API endpoints, client pages, email types, or document processing logic — update the relevant `docs/architecture/*.mmd` diagram(s).
   - Git commit & push the feature branch — **invoke the `git-ship` skill** (mandatory entry point for all git write ops). Do not run `git commit` / `git push` / `git merge` directly. All file edits MUST be done before this point — branch-guard blocks edits on main.
   - **Backend deploys:** If the change touches backend/runtime deployment surfaces, follow the project deployment workflow for live testing. The skill has `Bash` permission and may run documented non-destructive deployment commands when the project workflow permits; ask before destructive operations, secret changes, production-impacting changes, or any deployment command the repo does not clearly document.
   - **NO auto-merge to main.** Never merge to main without explicit approval, even for "testing" convenience. Commit locally, push the feature branch when requested by the workflow, then **pause for explicit approval** before merging or pushing to main.
   - **Frontend/static hosting changes** may only go live after the project-specific release path runs (such as merge-to-main or a hosting deploy). Surface this as a blocker the user needs to decide about, not something to bypass.
   - **DO NOT delete the branch during Phase D.** Follow-up improvements are common after live testing, so the branch persists through implementation. Branch deletion happens later, only once the log is `[COMPLETED]` — see **Phase E, step 5 (Branch cleanup)**.
   - Tell the user (template):
     > "Feature branch `{branch}` pushed. Backend deployed if applicable. Frontend/static hosting changes may not be live until the project release path runs. Approve merge/release when ready."
   - **Do NOT edit any files after merging to main** — the branch-guard hook will block it. If follow-up is needed, check out the feature branch again first.

---

## Phase E: Test Handoff

After implementation, persist outstanding test items so nothing gets lost between sessions.

1. **Collect:** Gather all unchecked items from the log's **Section 7 (Validation Plan)** — these are the tests that still need manual verification or end-to-end testing.
2. **Write to `current-status.md`:** Add a test entry under **"Active TODOs"** with this format:
   ```
   N. **Test DL-NNN: [Feature Name]** — [1-line summary of what to verify]
      - [ ] [Test item 1 from Section 7]
      - [ ] [Test item 2 from Section 7]
      - [ ] ...
      Design log: `.agent/design-logs/NNN-feature-name.md`
   ```
   * Place it right after the implementation TODO it relates to (same priority number).
   * If the design log is now `[IMPLEMENTED — TESTING]` or `[COMPLETED]`, the test entry is what keeps it on the radar.
3. **Mark the log:**
   - If all Section 7 tests passed in-session → run `bash .claude/workflows/close-design-log.sh <NNN>` — it patches status to `[COMPLETED]` in both the DL file and INDEX.md, runs the PII guard, and stages the files.
   - If tests are still outstanding → leave as `[IMPLEMENTED — NEED TESTING]` (no script needed yet).
4. **Append the cost line:** add a one-line cost estimate to the DL's Section 8 footer:
   `**Cost:** ~N subagent dispatches (X haiku / Y sonnet / Z opus), ~M research searches, [short/medium/long] session`
   Counts, not tokens (token usage isn't measurable in-session) — after ~10 logs this shows where the budget goes.
5. **Branch cleanup (only once truly done):** After the log is `[COMPLETED]`, merged to main, and — if it deployed — live-verified, delete the feature branch so stale branches don't accumulate. Do this through the `git-ship` skill (mandatory git entry point), and only after confirming the branch is fully merged:
   - **Verify merged:** `git merge-base --is-ancestor <branch> origin/main` must succeed. (A branch rebased before an FF-push to main can read as "not merged" by SHA even though its *content* is on main — confirm the content landed before deleting.)
   - **In a worktree checked out on that branch,** you cannot delete the branch you're on: `git checkout --detach` first, then delete.
   - **Delete local + remote:** `git branch -D <branch>` and `git push origin --delete <branch>`. (Force-push to the branch may be blocked by a guard hook; `--delete` is the supported path.)
   - **Skip / defer** if follow-up work on the same branch is still likely, or if the project's worktree-cleanup automation needs the merged branch to detect cleanup eligibility — check the repo's cleanup rules before deleting.
   - **Do NOT remove the worktree directory** as part of this step if you are currently running inside it — that terminates your own session's working directory. Leave worktree teardown to the project's cleanup script or a later session.

**Rule:** A design log is NOT `[COMPLETED]` until all Section 7 items are checked off.

---

## Handling Mid-Implementation Feedback

When the user gives feedback during or after implementation, assess the type and respond accordingly:

| Feedback Type | Action |
|---------------|--------|
| **Clarification** | Answer directly, no log change needed |
| **Bug in implementation** | Fix it, append to Section 8 (Implementation Notes) |
| **Missed constraint / edge case** | Append to Section 8, update Section 7 with new test case |
| **Scope expansion / new feature** | Create a **new** design log — don't overload the current one |
| **Refinement of existing design** | Append to existing log's Section 8 |

**When uncertain:** State your assumptions, propose options (quick fix vs. proper solution), ask for preference. Don't guess.

---

## Status Values

| Status | Meaning |
|--------|---------|
| `[DRAFT]` | In plan mode. Researching, exploring, documenting. **NO CODING** |
| `[APPROVED]` | User approved via ExitPlanMode. Ready to implement. |
| `[BEING IMPLEMENTED — DL-NNN]` | Implementation actively in progress. |
| `[IMPLEMENTED — NEED TESTING]` | Code deployed. Test checklist being written to `current-status.md`. |
| `[COMPLETED]` | Implemented, ALL Section 7 tests passed, and committed. |
| `[DEPRECATED]` | Feature removed or superseded by a newer log. |

## Naming Convention

- Format: `NNN-description.md`
- Numbering: Sequential (001, 002, 003...)
- Casing: Lowercase with hyphens
- Language: English only

---

## Lite Mode

Triggered by `/design-log lite` (or a request for a "light/quick design log"). Same protocol, same hard gates — what shrinks is **volume**, never **gates**.

| Phase | Full mode | Lite mode |
|-------|-----------|-----------|
| A0 Branch setup | Mandatory | **Mandatory — unchanged** |
| A1 Existing logs | INDEX + archive + grep bodies + read related logs in full | INDEX + research-index check; read related logs only on a direct hit |
| A2 Pre-scan | Bounded reuse scan | Same, but cap at the most obvious reuse candidates |
| A3 Questions | Usually 5+ | **2–3 decision-shaping questions** (still via `AskUserQuestion`, still wait for answers) |
| B Research | 3+ sources, 5–10 min | **1 source minimum**; a fresh research-index hit (<90 days) satisfies Phase B with **zero new searches** (cite the prior DL) |
| C Plan mode | `EnterPlanMode` first → DL file `[DRAFT]` → `ExitPlanMode` | **Unchanged — all three gates apply** |
| Template | All sections fully filled | Sections 3–5 may be compressed to a few bullets each; Sections 1, 2, 6, 7 stay concrete |
| D Implementation | Per approved plan + housekeeping | Unchanged |
| E Test handoff | Mandatory | **Mandatory — unchanged** (including the cost line) |

**Never skipped in lite mode:** A0 branch setup, the `[DRAFT]` status, the `EnterPlanMode`/`ExitPlanMode` approval gate, saving the DL file, Phase E handoff. Lite mode is a smaller log, not a different protocol.

---

## Gotchas

- Do not treat plan mode text as persistent documentation; always save the design log file.
- Do not create a new log when an active related log should be extended; if creating a new one anyway, explain why.
- Do not use filename-only triage for prior logs; grep bodies and read related logs in full.
- Do not repeat old research verbatim; check the research index (B0.5), do targeted delta research, and cite the prior log.
- Do not mark `[COMPLETED]` until every Section 7 validation item is checked off.
- Do not patch INDEX.md or the DL file by hand when closing — run `bash .claude/workflows/close-design-log.sh <NNN>` so the PII guard runs and the status is updated consistently in both files.

---

## Evaluation Checklist

Self-check after the skill runs. If any answer is "no" you skipped a phase:

- [ ] **Phase A:** Asked decision-shaping clarifying questions via `AskUserQuestion` (not plain text), usually 5+ (2–3 in lite mode) unless prior context answered enough, and waited for answers?
- [ ] **Phase A (HARD):** Were those questions written in **simple English** — plain everyday words, no unexplained jargon/acronyms, asking about the visible effect rather than the technique, one decision per question, trade-off stated in each option? A question the user cannot decode = FAIL.
- [ ] **Phase A (HARD):** Ran the A1 relevance grep — task-derived search terms (incl. Hebrew UI labels) across `INDEX.md` + `ARCHIVE-INDEX.md` + **all DL bodies**; read any related prior DL in full and cited it in the new DL's Related Logs; surfaced findings to the user before A3 questions. **A recency summary ("last N logs") alone = FAIL.**
- [ ] **Phase A:** Codebase pre-scan done — existing solutions/reuse opportunities surfaced?
- [ ] **Phase B:** 3+ research sources cited via a research MCP (NOT `WebSearch`/`WebFetch`), OR a fresh research-index hit with delta research; time-boxed to 5–10 min? Research index updated afterward?
- [ ] **Phase C:** First tool call was `EnterPlanMode` — no Glob/Grep/Read/Write happened before plan mode was active?
- [ ] **Phase C:** Design log file written using the template at `assets/design-log-template.md`, status `[DRAFT]`, no template placeholders left?
- [ ] **Phase C:** Exited via `ExitPlanMode` (the single approval gate) — not a plain-text "approve?" prompt?
- [ ] **Phase D:** Status moved through `[BEING IMPLEMENTED]` → `[IMPLEMENTED — NEED TESTING]`; INDEX.md and `current-status.md` updated; `git-ship` skill invoked (not raw git)?
- [ ] **Phase E:** All unchecked Section 7 items copied to `current-status.md` "Active TODOs"; cost line appended to Section 8; log marked `[COMPLETED]` only after all tests pass (via `bash .claude/workflows/close-design-log.sh <NNN>`)?

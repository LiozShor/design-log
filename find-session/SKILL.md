---
name: find-session
description: 'Find a past Claude Code session to resume — I closed my session, find the session where we…, which session discussed X, which session talked to the other agent.'
allowed-tools: Bash, Read
---

# Find Session

Search Claude Code's local session logs (`~/.claude/projects/**/*.jsonl`) for a past conversation matching the user's description, and return resume instructions (or, when the user really wants an artifact the session produced, hand them that file directly).

## When this triggers

- "I closed / accidentally quit my session, help me find it."
- "Which session did we discuss / fix / build X in?"
- "Find the session where we worked on DL-142 / the upload bug / the nav redesign."
- "Where did the two project agents talk to each other?" (cross-project conversations count).
- Any request to locate a past Claude Code conversation by topic, error text, file, or feature so it can be resumed or referenced.

## When this does not trigger

- The user wants the *contents* of a file a session wrote, and the session itself is irrelevant → just `Read` the file; don't hunt sessions.
- The user wants git history, commits, or PRs → use `git log`, not session logs.
- The user is asking about the *current* live session or its context → that's not a search.
- The user wants to resume a specific known session id → they can `claude --resume <id>` directly; only help translate the worktree path.

## Required inputs

- A search hint: a distinctive phrase, error message, feature name, DL number, file path, or topic. Shorter and more distinctive beats vague ("preview load failed", "DL-142 react port", an agent or feature name > "that bug we fixed").
- If no hint is given, ask for one before searching. If the user only has a rough time ("this morning", "just now"), that alone is enough to rank by mtime — proceed.

## Where sessions live

- One project dir per `cwd` under `~/.claude/projects/`, named by the slugified path (`/` → `-`).
- **Worktrees are separate project dirs** — a freshly-closed session often lives in a `...-worktrees-...-claude-<repo>-YYYYMMDD-HHMMSS` dir, NOT the main repo dir. Always include them.
- **Cross-project topics span sibling dirs.** A conversation between two agents/projects has *two* session files in *two* different project dirs — search siblings and expect both ends to be relevant.
- Each `.jsonl` file is one session. Filename (minus extension) is the session id. File mtime ≈ last active time.

## Workflow

### 1. Narrow to recent sessions first
For "just now" / "today" / "accidentally closed", start with the most recently modified files across **all** project dirs. The active session's own file is usually the freshest — skip it.

```bash
ls -lt ~/.claude/projects/*/*.jsonl | head -20
```

### 2. Match on the hint (two complementary signals)
Use Python over the jsonl (faster and structured). For each candidate file:

- **User-message match** — check `type == "user"` content for the hint. Verbatim error strings and DL numbers appear in user messages and are gold.
- **Whole-file distinctive-term count** — for cross-cutting topics, `grep`/count a distinctive term across ALL lines (user + assistant + tool output). A term that appears many times (e.g. a distinctive agent or feature name ×33) is a strong ranking signal *even when the individual hits aren't in user messages*. Use this when the topic is a running thread, not a single pasted string.

Prefer literal-substring first, then loosen. Hebrew terms match exactly — don't transliterate.

### 3. Filter out skill-boilerplate noise
Skill-invocation text is embedded in sessions and produces false positives on generic words ("handoff", "other claude", "bridge", "inbox" all appear in loaded `SKILL.md` bodies). **Exclude any line containing `Base directory for this skill`**, and discount matches that only appear inside `<command-message>`, `<task-notification>`, or skill bodies. Require the signal in real conversation content.

### 4. Rank candidates
Sort by mtime descending, weighted by hit count. For the top 3–5 extract:
- First substantive user message (skip `<command-message>`, `<bash-stdout>`, `<persisted-output>`, `<task-notification>` wrappers).
- A snippet of matching content with ±200 chars of context.
- Last message `timestamp` as "last active".

### 5. Extract artifacts (when the hint names one)
If the hint references a file ("on an md file", "the config we wrote"), regex-extract file paths (match runs of word/dot/slash/dash chars ending in `.md`, etc.) mentioned in the session. This both confirms the match and may answer the user *without resuming* — hand them the artifact path directly.

### 6. Disambiguate, then return instructions
If multiple candidates are plausible, show the top 2–3 with first-user-message + context and ask which. Don't guess silently. For the confirmed match, print the Output format below.

## Decision gates

- **Ambiguous match (2+ plausible)** → present the shortlist and ask; never auto-pick.
- **Current live session** → always exclude it; it matches anything the user just typed.
- **Worktree session** → translate the project-dir slug back to the real worktree path and tell the user to `cd` there first — `claude --resume` fails from the wrong project.
- **Cross-project topic** → surface both ends (each session id + its `cd` path), and note which is the origin.
- **User wanted the artifact, not the session** → give the file path and offer resume as secondary; don't force a resume.
- **"Resume cancelled" stub** → if the last entry is `<local-command-stdout>Resume cancelled</local-command-stdout>`, that file is a stub; the real session is its parent — find the id it tried to resume.

## Output format

```
Session:     <uuid>
Project dir: <slug under ~/.claude/projects/>
Last active: <last-message timestamp>
Topic:       <one-line summary from first user message or match context>

To resume:
  cd <actual-worktree-or-repo-path>
  claude --resume <uuid>
```

For cross-project results, print one block per end and name the origin. When an artifact answered the request, lead with the file path and mark resume as optional.

## Gotchas

- **Don't match on tool-call content alone.** "preview load failed" inside an assistant's Grep output is a false positive — require a user-message hit OR a high whole-file count of a distinctive term (step 2).
- **Skill boilerplate is the #1 noise source** — exclude `Base directory for this skill` lines (step 3) or generic words will match dozens of unrelated sessions.
- **Don't read whole huge files into memory (>5 MB).** Stream line-by-line.
- **mtime lags** the last real message by seconds — for ordering inside a 1-minute window use the last message's `timestamp` field, not mtime.
- **Worktree path translation is mandatory**, not optional — the most common way a resume silently fails.
- **Both ends of a cross-project chat matter** — returning only the project the user named leaves half the conversation unfound.

## Evaluation checklist

After running, verify:
- Did I exclude the current live session?
- Did I filter skill-boilerplate false positives before ranking?
- For a cross-project topic, did I surface both session ends?
- Did I translate worktree slugs to real `cd` paths?
- If the hint named an artifact, did I hand over the file path (not just a resume line)?
- Did I ask before guessing when 2+ candidates were plausible?

## Evals

- `evals/` — one positive (topic search), one negative-trigger (wants file contents, not a session), one edge-case (cross-project two-ended match). See `~/.claude/skills/skill-improver/assets/eval-template.json` for the schema.

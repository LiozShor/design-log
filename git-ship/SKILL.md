---
name: git-ship
description: 'Commit and optionally push. MANDATORY before any git write — commit, push, merge, rebase, reset, branch -d. Triggers: ship it, commit, push. Do NOT enter here when the request also includes deploying and live-verifying the change ("deploy, push, delete the branch", "deploy and close it out", "run the browser tests and ship it") — that whole sequence belongs to your deploy-and-verify skill, which calls this one for the git mechanics.'
allowed-tools: Bash
---
# Git Ship

## When this triggers

- User says "ship it", "commit this", "push", "merge this branch", or asks to run `git commit`, `git push`, `git merge`, `git rebase`, `git reset`, or `git branch -d`.
- Any git write operation is about to happen in this project — this skill is the mandatory entry point.

## When this does not trigger

- Read-only git commands (`git status`, `git diff`, `git log`, `git show`, `git branch --show-current`) — those are fine without invoking this skill.
- Non-git tasks — no overlap with other sibling skills.

## Mandatory Invocation
This skill is the ONLY entry point for git write operations in this project. Before running `git commit`, `git push`, `git merge`, `git rebase`, `git reset`, `git branch -d`, or `git checkout main` — you MUST be inside this skill. Read-only commands (`status`, `diff`, `log`, `show`, `branch --show-current`) are fine without invocation.

## Required inputs

- Current branch name (`git branch --show-current`).
- The diff to be committed (`git diff` / `git diff --staged`).
- User confirmation before pushing or merging to main.

## Pre-Ship Validation (Multi-Tab Safety)
1. **Branch guard:** Run `git branch --show-current`. If on main/master, **REFUSE** and error: "You're on main. This should not happen — a feature branch should have been created at session start." Do NOT proceed with the commit.
2. Run `git diff` and review ALL hunks. Check if any changes don't belong to the current task/ticket (e.g., changes from another Claude Code tab session).
3. If suspicious mixed changes are detected: **warn the user**, list the foreign hunks, and suggest using `git add -p` to selectively stage only the relevant changes.
4. Verify the commit will only include files relevant to the current task.

## Workflow
1. Run `git status --porcelain` to see all changes.
2. **Stage explicit paths only. NEVER `git add -A` / `git add .` / `git add --all`** — a blanket add sweeps in a parallel session's work (this has happened: ~16 foreign files in one commit). Instead:
   - Derive the list of paths belonging to THIS task from the porcelain output.
   - **Show the user that exact file list** before staging.
   - Stage them by path: `git add path/one path/two`.
   - Any dirty file not on the list: name it and ask — do not stage, do not stash it unilaterally.
   - If a PreToolUse hook blocking blanket adds is installed, and it fires, fix the command; do not route around it.
3. Run `git diff --staged` and confirm the staged set matches the list you showed.
4. Generate a conventional commit message: `type(scope): description` (under 72 chars).
5. Commit. **If amending, verify with `git log -1 --stat`** that the amend replaced the commit rather than stacking a new one.
6. Ask the user: "Push to remote?" — only push if they confirm.
7. **Before any push that lands on main, run `git fetch && git rebase origin/main` first.** On conflict, stop and show it — a parallel session may have pushed conflicting docs.

## Adversarial Review Gate (before merge to main)
Before merging to main, dispatch **one read-only reviewer subagent** over the diff. It is what catches cross-seam bugs that a serial implementation pass misses.

- Agent type `general-purpose`, **`model="sonnet"`** — this is structured multi-file analysis, not top-tier reasoning work.
- Give it the changed-file list and the diff, and tell it to **read the files fresh with no assumptions from this conversation**.
- Ask it to report **violations only** — no praise, no summary:
  - tests asserting invented values that don't exist in the type definitions or fixtures
  - stale references to removed systems / dead code paths
  - conventions in `CLAUDE.md` or the repo that the diff breaks
  - hardcoded config values (ids, URLs, keys, emails, machine paths)
- Surface its findings to the user before merging. Do not auto-fix and merge silently.

## Post-Deploy Verification
A push is not a deploy, and a deploy is not "live". If this ship includes a deploy:
- Poll the build/deployment status until it actually reports success — never infer success from a clean push (a stalled GitHub Pages build once left a "shipped" change undeployed).
- Then curl the URL / query the prod table / screenshot the page, and **paste the evidence**.
- If your setup has a dedicated deploy-and-verify skill, delegate this to it instead of hand-rolling it.

## Final Report
Max 5 bullets: what was staged, the commit, push/merge status, deploy evidence, anything left open. No preamble.

## Post-Push Housekeeping (Before Merge)
After successful push, if there are any pending file updates (e.g., `.agent/current-status.md`, design log status), do them NOW — while still on the feature branch. Commit and push again if needed. The branch-guard hook blocks all edits on main.

## Merge to Main
Only after ALL file edits are committed and pushed, ask: "Merge to main and delete branch?"

### Worktree-aware merge (multi-worktree repos)
**Prefer the project wrapper** — `.claude/workflows/merge-and-push.sh <branch>` from the canonical clone (`~/my-project/`). It encodes the worktree-safe sequence (fetch → FF pull → `--no-ff` merge → push) so you don't have to hand-roll it. Run it from the canonical clone, **never from a session worktree** — `git checkout main` will error `fatal: 'main' is already used by worktree at '.../my-project'`.

```bash
cd ~/my-project
bash .claude/workflows/merge-and-push.sh <branch>
```

Fall back to manual only if the wrapper is unavailable:
```bash
cd ~/my-project
git fetch origin main
git checkout main && git pull --ff-only origin main
git merge --no-ff <branch> -m "Merge <branch>: ..."
git push origin main
```

Or fast-forward-only push (no checkout, leaves canonical clone untouched):
```bash
git push origin <branch>:main
```

### Default (single-checkout repos)
```bash
git checkout main && git merge {branch} && git branch -d {branch} && git push
```

**Do NOT edit any files after merging to main** — the branch-guard hook will block it.

## Post-Merge Deploy
If the merge touched anything under `api/` (Workers code, `wrangler.toml`, `api/src/**`), the Worker does NOT auto-deploy. Run the project wrapper from the canonical clone:
```bash
cd ~/my-project
bash .claude/workflows/deploy-worker.sh
```
The wrapper clears the stale `CLOUDFLARE_API_TOKEN`, passes `-c wrangler.toml` (autoconfig-hijack guard), and verifies the health endpoint after deploy. Frontend changes under `frontend/` deploy via `bash scripts/deploy-pages.sh "<msg>"` (when Pages git auto-deploy is unreliable).

## Closing a Design Log
After merging a DL branch, run `.claude/workflows/close-design-log.sh <NNN>` from the canonical clone — it patches the DL status header, updates `INDEX.md`, runs the PII guard, and stages the files for a closing commit.

## Decision gates

- **Branch guard:** on main/master → REFUSE and error, do not commit.
- **Mixed-change guard:** suspicious foreign hunks in the diff → warn the user and suggest `git add -p` instead of committing everything.
- **Push gate:** never push without explicit user confirmation ("Push to remote?").
- **Merge gate:** never merge to main without explicit user confirmation ("Merge to main and delete branch?"), and only after all edits are committed and pushed.
- **Post-main edit guard:** do not edit files after merging to main — the branch-guard hook blocks it.

## Output format

Report, in order: the branch checked, the generated conventional commit message (`type(scope): description`), whether the commit succeeded, and whether push/merge happened (or is pending user confirmation). Don't silently push or merge without stating it happened.

## Gotchas

- **Conflict-prone files:** `.agent/current-status.md` and `.agent/design-logs/INDEX.md` are appended by every parallel session and conflict on nearly every merge. Resolve by keeping ALL entries from both sides (DL number ordering: highest at top). If `.gitattributes` declares `merge=union` for these files, conflicts auto-resolve.
- Worktree-aware merges for multi-worktree repos must run from the canonical clone, never from a session worktree (`git checkout main` errors if `main` is checked out in another worktree).
- Post-merge Worker deploys don't happen automatically — run the project's deploy wrapper explicitly if `api/` changed.

## Evaluation checklist

- Was the branch guard checked before any write (never committing on main)?
- Was the diff reviewed for foreign/unrelated hunks before staging?
- Was the commit message in conventional format (`type(scope): description`)?
- Was the user asked before pushing, and before merging to main?
- Were conflict-prone files (`current-status.md`, `INDEX.md`) resolved by keeping both sides' entries?

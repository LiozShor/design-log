# Claude Code Skills

A collection of reusable [Claude Code](https://docs.claude.com/en/docs/claude-code) skills for
planning, research, version control, and working with Airtable, Tally, and HTML visualizations.

Each skill is a self-contained folder with a `SKILL.md` (the router + instructions) and optional
reference/asset files. They're written to be **portable** — no machine-specific paths, no hardcoded
IDs or secrets. Where a skill needs project values (an Airtable base, an env key), it reads them from
config/env and ships a `*.example` template.

## The skills

| Skill | What it does | Invoke / trigger |
|-------|--------------|------------------|
| **[design-log](design-log/)** | Research-driven **Stop & Think** planning protocol — clarify → research → plan-mode exploration → persistent design log → single approval gate → implement → test handoff. | `/design-log <task>` |
| **[airtable](airtable/)** | Read/write Airtable records via curl (reads) and `pyairtable` (writes), with a schema gate that prevents the field-name-guessing 422s. | Mentions Airtable, a base, or `tbl…/rec…/fld…` IDs |
| **[tally](tally/)** | Build and edit [Tally](https://tally.so) forms through the Tally MCP — blocks, pages, conditional logic, mentions/recall. | Mentions Tally, a form, or a form ID |
| **[tech-researcher](tech-researcher/)** | Compare libraries/frameworks/tools and verify current APIs against live sources before choosing or implementing. | "X vs Y", "is it deprecated?", "latest API" |
| **[find-session](find-session/)** | Search Claude Code's local session logs for a past conversation and return resume instructions. | "find the session where we…", "I closed my session" |
| **[git-ship](git-ship/)** | A safe single entry point for git writes — branch guard, explicit-path staging (never `git add -A`), conventional commits, ask-before-push, rebase-before-main, an adversarial review gate before merge, and post-deploy verification. | "ship it", "commit", "push" |
| **[visualize](visualize/)** | Turn conversation content or data into a self-contained single-file HTML visualization (deck, dashboard, one-pager, chart, etc.) — grounded in the real design when the subject already exists, served over localhost and verified in a browser before hand-off. | "visualize this", "make a deck/dashboard" |
| **[skills-build](skills-build/)** | Build a NEW agent skill from a real repeated workflow — decide if it deserves a skill, scaffold `SKILL.md`, validate structure. | "create a skill", "scaffold a SKILL.md" |
| **[skill-improver](skill-improver/)** | Review or improve an EXISTING skill — structure, triggers, permissions, splitting, eval coverage. | "review this skill", "improve my skill" |

## Install

Drop a skill folder into your skills directory:

- **Global (all projects):** `~/.claude/skills/<skill>/`
- **Per-project:** `<your-repo>/.claude/skills/<skill>/`

Then invoke it by name (`/design-log`, `/visualize`, …) or let it trigger automatically from its
description. Most skills need no setup; a few have a one-time config step documented in their `SKILL.md`:

- **airtable** — copy `airtable/config.example.env` → `config.env` and set your base/table IDs; set
  `AIRTABLE_API_KEY` in your env.
- **tech-researcher** — copy `tech-researcher/.env.example` → `.env` if you use key-based research MCPs.
- **design-log** — works out of the box; see `design-log/references/customization.md` to adapt log
  locations, research tools, or wire in a number-reservation / close script.
- **skills-build + skill-improver** — install **both together**: `skills-build`'s validation hard-gate
  and eval template reuse `skill-improver`'s `scripts/review-skill-structure.sh` and
  `assets/eval-template.json`. Installing `skills-build` alone leaves its quality gate non-functional.

`config.env` and `.env` are gitignored — only the `*.example` templates are tracked.

## Credits

The **design-log** skill extends [Yoav Abrahami's Design-Log Methodology](https://github.com/yoavaa/design-log-methodology).
The core principles are his; this package adds Claude Code plan-mode integration, a mandatory research
phase, hard gates, and a test-handoff phase.

## License

MIT © Lioz Shor. See [LICENSE](LICENSE). For design-log derivatives, credit Yoav Abrahami's original methodology.

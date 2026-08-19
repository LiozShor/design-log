---
name: visualize
description: >
  Create self-contained HTML visualizations (single .html file) from conversation content,
  data, or ideas. Trigger on: "visualize this," "make a deck/slide/infographic/dashboard,"
  "build a one-pager / poster / carousel / kanban / mind map / org chart / flowchart,"
  "show me a chart," "render as HTML," or any ask to convert text/data into a visual
  HTML artifact. Do NOT trigger when the user asks for a Figma design (use `figma-*`),
  a Tally form (use `tally`), a Canva file (use `canva-*`), a real multi-page website,
  a React/Vue component, or a screenshot of an existing page (use `agent-browser`).
allowed-tools: Write, Edit, Read, Grep, Glob, Bash, WebFetch
license: MIT
metadata:
  author: careerhackeralex
  version: 0.5.0
  category: document-creation
  tags: [visualization, html, slides, dashboard, infographic]
---

# Visualize

Turn any idea, data, or content into a stunning single-file HTML visualization.

This SKILL.md is the operational layer. Detailed patterns live in `references/` and are loaded
on demand — see the [Reference map](#reference-map) at the bottom. Never inline that detail here.

## When this triggers
- User says "visualize," "make a deck/slide/dashboard/infographic/poster/carousel," or any visualization-type keyword.
- User pastes data (CSV, JSON, table, numbers) and wants it shown visually.
- User shares a URL and asks for a visual summary.
- User wants a single self-contained `.html` artifact (no server, no build step).

## When this does not trigger
- Figma / mockup / design-system work → `figma-generate-design`, `figma-use`.
- Tally form / questionnaire → `tally`.
- Canva file → `canva-*` tools.
- Real multi-page website, React/Vue/Next app → not a skill match; build with framework.
- Screenshotting or scraping an existing webpage → `agent-browser`.
- Editing an existing HTML file the user already has → use plain `Edit`.

## Required inputs
- The content to visualize (conversation context, pasted data, URL, or explicit description).
- **If the visualization renders options for something that already exists** (a screen, component,
  loader, nav, card, color scheme, layout in the user's product) — the actual current implementation.
  Go read it before writing any HTML. See [Ground in the current implementation](#ground-in-the-current-implementation).
- Optional: visualization type (deck, dashboard, infographic, etc.) — infer if not given.
- Optional: output path (defaults to `~/Downloads/<kebab-case-name>.html`).

## Workflow
1. **Pick type.** Match the request to a row in the [Visualization Types](#visualization-types) table below. If ambiguous, pick the closest fit and proceed. Detailed per-type patterns: [references/types.md](references/types.md).
2. **Ground in the current implementation — before writing a single line of HTML.** If the thing being visualized already exists anywhere (a screen, component, loader, nav, card, palette, layout in the user's product), go find and read the real code, and render the options in *that* design — not an invented one. Mandatory, no exceptions: see [Ground in the current implementation](#ground-in-the-current-implementation).
3. **Gather content.** Use the conversation context, pasted data, or `WebFetch` for URLs (see [Source content](#source-content)). Never use placeholder/Lorem ipsum.
4. **Copy skeleton.** Start from [references/skeleton.md](references/skeleton.md) — never write HTML from scratch. It carries the theme system, exact CSS-property names, menu, animations, print styles, and accessibility hooks the evaluator checks.
5. **Add content** to the `<!-- YOUR CONTENT HERE -->` block. Follow the type-specific rules in [references/types.md](references/types.md).
6. **Wire libraries** as needed — [references/libraries.md](references/libraries.md) for CDN URLs + the Chart.js / Reveal.js / Leaflet patterns, [references/menu.md](references/menu.md) for the required hamburger menu, [references/animations.md](references/animations.md) for entrance/scroll animation, [references/design-system.md](references/design-system.md) for typography/color/spacing/sizing, [references/css-techniques.md](references/css-techniques.md) for advanced CSS.
7. **Run the Evaluation checklist** below before saving.
8. **Write the file** with `Write` to `~/Downloads/<name>.html` (kebab-case).
9. **Serve it over localhost, then open it.** See [Serving and verifying](#serving-and-verifying) — `file://` silently breaks the clipboard export and cannot be inspected with Playwright. Serve the containing directory, open the `http://127.0.0.1:<port>/<name>.html` URL, and return **that** URL in the response.
10. **Verify in the browser before claiming it works.** Load the localhost URL with Playwright, check console errors, confirm no horizontal overflow at 375px, and smoke-test the main interaction. Never report a visualization as done on the strength of the HTML source alone.

## Decision gates
- If the request matches a non-trigger (Figma, Tally, Canva, real website) → stop, suggest the correct skill.
- If the subject already exists in the user's product and you have **not** read its source → stop and read it before writing HTML ([Ground in the current implementation](#ground-in-the-current-implementation)).
- If a real token, string, or font exists and you used an approximation instead → replace it with the real value before writing.
- If skeleton.md required elements (menu, theme classes, exact CSS-property names) are missing from the output → evaluation will fail; fix before writing.
- If charts render blank → see the **Troubleshooting** checklist in [references/libraries.md](references/libraries.md#chartjs).
- If the layout overflows at 375px viewport → mandatory fix before write.
- If `Chart.defaults.animation = false` is not set immediately after the Chart.js CDN → must add.
- If the purpose is to let the user **choose between rendered options** → make options selectable and add a clipboard export ([Pick-and-Extract](#pick-and-extract)).

## Ground in the current implementation

**If the thing being visualized already exists, go look at it first. Then render the options in the
real design — same tokens, same markup shape, same direction.** An option card built from an invented
design is worthless: the user can't judge "does this fit my product" against something that isn't
their product, and a picked option can't be implemented as-is.

This applies to any visualization whose subject is an existing artifact — UI variants, loader styles,
nav layouts, card treatments, color schemes, empty states, copy in situ, before/after comparisons.
It does **not** apply to green-field concepts, abstract data stories, or charts of pure numbers.

### The four things to extract, in order

1. **Find the real source.** `Glob`/`Grep` for the component, screen, or stylesheet by the name the
   user used — the Hebrew or English label, the class name, the route. Read the actual file. Never
   reason from what the code probably looks like.
2. **Lift the design tokens verbatim.** Copy the real CSS custom properties, hex values, font stacks,
   radii, shadows, spacing scale, and breakpoints out of the source into your page's `:root`. Do not
   approximate a brand color by eye and do not substitute the skeleton's defaults where a real token
   exists. If the project has a design skill or token file (e.g. a `*-design` skill, `tokens.css`,
   `tailwind.config`), read that too and prefer it as the authority.
3. **Reproduce the real structure and state.** Same element hierarchy, same class names, same content
   — real strings from the app, real row counts, real number formats. Preserve `dir="rtl"`, the real
   font, and Hebrew/English mixing exactly as the product does it. Options must differ **only** on the
   axis being decided; everything else stays identical, or the comparison isn't a comparison.
4. **See it rendered when you can.** If the page is running (localhost, a preview URL, production),
   open it with Playwright and screenshot the real component before rebuilding it. Live pixels beat
   source-reading for spacing, density, and what the thing actually feels like.

### Always include the current state as a card

Render today's version as the **first** card, labeled "current" and marked `.no-pick` (not selectable
— see [pick-and-extract.md](references/pick-and-extract.md)). Every option is then judged as a delta
from a real baseline instead of in a vacuum, and "keep what we have" stays a visible answer.

### When you can't find it

Don't quietly invent one. Say in one line what you searched for and didn't find, state the design
assumption you're proceeding under, and build the options on that stated assumption — then flag it
above the fold on the page itself so the user reads the picks in the right light.

## Serving and verifying

**Always serve over localhost — never hand the user a bare `file://` link.** The file is still
self-contained and portable; localhost is only how it gets opened and checked.

```bash
cd ~/Downloads && (python3 -m http.server 8899 >/dev/null 2>&1 &) ; sleep 2
curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8899/<name>.html   # expect 200
open http://127.0.0.1:8899/<name>.html
```

Three concrete reasons, all of which have bitten real runs:

- **The clipboard export silently fails on `file://`.** `navigator.clipboard` is blocked on the
  file protocol, so a Pick-and-Extract page looks fine and then does nothing when the user clicks
  copy. Over `http://127.0.0.1` it just works.
- **Playwright refuses `file://`** (`Access to "file:" protocol is blocked`), so you cannot verify
  your own output — no console-error check, no computed-style check, no 375px overflow check.
- **Some CDN and font loads behave differently** under the file protocol.

Pick an uncommon port (8899, not 8000) so you don't collide with the user's dev server, and confirm
the 200 before opening. Mention the server is running when you hand over the link.

**Verify, don't assume.** When something looks wrong, read the computed styles rather than guessing
at the cause:

```javascript
// Is it a broken variable, or just a low-contrast palette? These answer it definitively.
getComputedStyle(document.documentElement).getPropertyValue('--negative')
getComputedStyle(document.querySelector('.card')).backgroundColor
```

A page that renders "unstyled" is far more often a **contrast** problem than a **CSS-not-loading**
problem — the light theme's `--surface: #FFFFFF` on `--bg: #FAFAF9` with 6%-opacity borders is
nearly invisible on a scaled-down screenshot. If cards and buttons disappear, add a stronger
component-level edge (a `--edge` variable at ~12–14% opacity plus a small shadow) rather than
rewriting the mandated `--border` / `--surface` names the evaluator checks for.

## Output format
- One `.html` file written to `~/Downloads/` (or user-specified path), kebab-case filename.
- File contains: inline CSS, inline JS, CDN libraries via `<script>`, no build step.
- Response to user: one short line + the `http://127.0.0.1:<port>/` URL (mention the local server). No additional summary unless asked.
- Evaluation rubric: [references/eval.md](references/eval.md) — the 8 scoring dimensions checked upstream.
- Background: [references/anthropic-skill-guide-notes.md](references/anthropic-skill-guide-notes.md) — the skill best-practices that shaped this structure.

## Gotchas
- **Skeleton is non-negotiable.** Starting from scratch loses the menu, theme classes, theme toggle, print styles, and accessibility hooks — all checked by the eval system.
- **Class-based themes only** (`html.theme-dark` / `html.theme-light`). Never `data-theme`, never `@media (prefers-color-scheme)` for the variables themselves.
- **`var` at top level, not `let`/`const`.** Function hoisting causes TDZ errors otherwise.
- **Chart.js: `Chart.defaults.animation = false` immediately after the CDN.** Required and auto-checked. Never disable tooltips.
- **No horizontal overflow at 375px.** Mandatory.
- **Single-screen posters use `overflow: hidden` + fixed dimensions** and no hamburger menu. A scrolling page screenshotted ≠ a poster.
- **Don't hand-draw SVG continents.** Use Leaflet for any geographic data.
- **Reveal.js needs numeric dimensions** (`width: 1280, height: 720`) — string `'100%'` causes blank slides.
- **Always use real content.** Never generate placeholder/Lorem ipsum data when real context exists.
- **Never mock up an option in a design you invented when the real one is on disk.** Read the component, lift its actual tokens, and render the options inside it — an option judged against a fake baseline is a wasted round-trip and can't be implemented as picked.
- **Options differ on one axis only.** If the variants also drift in font, spacing, or copy, the user is picking on noise instead of the actual decision.
- **Never hand over a `file://` link.** Serve over localhost — see [Serving and verifying](#serving-and-verifying). Clipboard export dies on `file://` and Playwright can't open it, so you'd be shipping unverified.

## Evaluation checklist
- If the subject already exists: was its real source read, and are its actual tokens/strings/direction reproduced in the output?
- If the subject already exists: is there a "current" reference card, rendered first and non-selectable?
- Do the options differ only on the axis being decided?
- Does the output start from skeleton.md verbatim?
- Are both `.theme-light` and `.theme-dark` classes defined with the full, exact custom-property set?
- Is the `.viz-menu` component present (toggle, theme, download PNG, print)? (Posters exempt.)
- Is Chart.js wired with `Chart.defaults.animation = false` + theme-aware colors + tooltips enabled?
- Does every chart canvas have `role="img"` and `aria-label`?
- Is there ≥1 entrance animation (`.animate` class, `data-reveal`, or scroll-driven `view()`)?
- Does the file render without horizontal scroll at 375px?
- Is there ≥1 meaningful interaction beyond menu + theme?
- Was the file served over localhost, opened, and the `http://127.0.0.1:<port>/` URL returned?
- Zero console errors on load (checked in a browser, not assumed)?

## Visualization Types

Pick the right format. Detailed per-type patterns and rules: [references/types.md](references/types.md).

| Type | When to Use | Key Feature |
|------|-------------|-------------|
| **Slide Deck** | Presentations, pitches | 16:9, keyboard nav, transitions |
| **Infographic** | Data summaries, visual stories | Long scroll, big numbers, sections |
| **Dashboard** | Metrics, KPIs | Grid of cards + charts |
| **Flowchart** | Processes, architecture | Mermaid or SVG diagrams |
| **Timeline** | Chronological events | Alternating left/right, scroll-triggered |
| **Comparison** | Side-by-side analysis | Feature matrix, pros/cons |
| **Data Viz** | Charts, data stories | Chart.js or D3 |
| **One-Pager** | Summaries, briefs | Single viewport, print-friendly |
| **Mind Map** | Concept relationships | Radial SVG layout |
| **Kanban** | Status tracking | Column-based cards |
| **Carousel Cards** | Social (IG/LinkedIn) | 1080×1080 per card, swipeable, bold text |
| **Event Poster** | Conferences, meetups | Portrait A4/letter, bold headline, date/venue |
| **Resume/CV** | Job applications | One-page, two-column, print-optimized |
| **Banner/Header** | Email, blog, social cover | 1200×630 or 1500×500, centered text on visual bg |
| **Quote Card** | Social proof, testimonials | Portrait/square, large quote, attribution |
| **Process Guide** | How-to, step-by-step | Numbered steps, icons, clear flow |
| **Status Report** | Executive updates | KPIs + progress bars + highlights, one-page |
| **Org Chart** | Team structure | Hierarchical tree, photos/avatars, roles |
| **Data Story** | Narrative + data | Scrollytelling, charts woven with text |
| **Product Card** | Feature highlight, launch | Hero image area, feature pills, CTA |

Every file MUST have ≥1 interaction beyond theme + menu — see the Type-Specific Interactivity table in
[references/types.md](references/types.md#type-specific-interactivity-mandatory).

## Source content

This skill runs mid-conversation. Leverage everything as source material — never invent placeholder data:
- **Conversation context** — summarize, structure, or visualize what's been discussed.
- **URLs** — `WebFetch` to crawl and extract content, then visualize as a summary.
- **Pasted data** — CSV (parse, auto-detect headers → chart), JSON (keys = labels, values = data, nested = series), tables (→ comparison/chart), numbers in text (→ stat cards).
- **Ideas / concepts / code** — turn abstract discussions or system designs into diagrams and data flows.

## Pick-and-Extract

When the visualization's purpose is to let the user **choose between rendered options** and hand the
decision back to you (UI variants, color schemes, layouts, copy choices), make the options selectable on
the page and add a clipboard export — a standalone `.html` can't post back, so the clipboard is the bridge.
Trigger phrases: "let me pick on the HTML," "make the options selectable," "I'll choose and send it back."
**Prerequisite:** if the options are variants of something that exists, complete
[Ground in the current implementation](#ground-in-the-current-implementation) first — the options must
be rendered in the product's real design, with today's version as the first, non-selectable card.
Full pattern, wiring code, and gotchas: [references/pick-and-extract.md](references/pick-and-extract.md).

## Reference map

Load only the file relevant to the current step (progressive disclosure):

| File | Load when |
|------|-----------|
| [references/skeleton.md](references/skeleton.md) | Always — the mandatory starting template + evaluation requirements. |
| [references/types.md](references/types.md) | Picking a type and applying its layout/interaction rules. |
| [references/libraries.md](references/libraries.md) | Wiring a CDN library (Chart.js, D3, Mermaid, Reveal.js, Leaflet, Tailwind) or debugging blank charts. |
| [references/design-system.md](references/design-system.md) | Typography, color, spacing, sizing, layout, accessibility, anti-patterns. |
| [references/menu.md](references/menu.md) | Building/verifying the required hamburger menu. |
| [references/animations.md](references/animations.md) | Entrance, scroll-driven, hover, and counter animation patterns. |
| [references/css-techniques.md](references/css-techniques.md) | Advanced CSS — container queries, `:has()`, view transitions, conic gradients, fluid type. |
| [references/pick-and-extract.md](references/pick-and-extract.md) | Decision visualizations the user picks on and exports back. |
| [references/eval.md](references/eval.md) | The 8-dimension scoring rubric the output is graded against. |
| [references/anthropic-skill-guide-notes.md](references/anthropic-skill-guide-notes.md) | Background on the skill-authoring best practices behind this structure. |

The quality bar: **"good, period"** — not "good for AI-generated."

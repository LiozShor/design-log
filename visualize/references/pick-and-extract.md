# Pick-and-Extract — interactive decision visualizations

When a visualization exists so the user can **choose between options** (UI variants, loader styles,
color schemes, layouts, copy options), don't make it read-only. Add an interactive picker so the user
selects ON the page, then exports their choices back as pasteable text. A standalone `.html` can't phone
home, so the bridge is the clipboard: **select → copy structured summary → user pastes into chat**.

Trigger this pattern when the user says any of: "let me pick on the HTML", "I want to choose and send it
back", "make the options selectable", "extract my choice to you", or whenever you're rendering candidate
options for a decision the user will hand back to you.

## Before any of this: render the options in the REAL design

If these options are variants of something that already exists in the user's product, read its actual
source first and build every card on the real thing — real CSS custom properties, real fonts, real
strings, real `dir`. See "Ground in the current implementation" in SKILL.md. Options rendered in an
invented design can't be judged for fit and can't be implemented as picked.

Concretely, that means the first card in each group is today's version — labeled "current", marked
`.no-pick`, not clickable — and every other card differs from it on the decision axis alone.

## The four required pieces

1. **Selectable cards** — each option card is clickable; selecting one in a group deselects its siblings
   (radio behavior per group). Mark each card with `data-group`, `data-id`, `data-name`.
2. **Visual selected state** — a clear ring + a checkmark badge on the chosen card. Show a "click to pick"
   hint on unselected cards so the interaction is discoverable.
3. **Sticky summary + export bar** — fixed bottom bar showing the live current choice per group, with a
   "copy selection" button that's disabled until every group has a pick.
4. **Export to clipboard** — build a compact, labeled text block (NOT JSON — plain labeled lines parse
   cleanly when pasted back), copy it to the clipboard, and also show it in a readonly `<textarea>` so the
   user can copy manually if the Clipboard API is blocked (common on `file://`).

Include a "none / keep current" option card in any group where opting out is valid — otherwise the user
can't express "I don't want this tier at all."

## Copy/clipboard robustness

**Serve the page over localhost** (`python3 -m http.server 8899` in the output directory, then open
`http://127.0.0.1:8899/<name>.html`). This is the single most effective fix: `navigator.clipboard` is
blocked outright on `file://`, so a Pick-and-Extract page opened as a file looks perfect and then does
nothing when the user clicks copy — the one interaction the whole page exists for. See the skill's
"Serving and verifying" section.

Localhost is the fix, but keep the fallbacks anyway — the user may re-open the saved file directly
later. Always provide BOTH copy paths and a manual fallback:

```javascript
function copyText(){
  var ta=document.getElementById('exportText'); ta.select();
  var ok=false;
  try{ ok=document.execCommand('copy') }catch(e){}                 // legacy path, works on file://
  if(navigator.clipboard){
    navigator.clipboard.writeText(ta.value).then(flashCopied).catch(function(){ if(ok) flashCopied() });
  } else if(ok){ flashCopied(); }
}
```

The readonly `<textarea>` (with `onclick="this.select()"`) is the guaranteed fallback — if both
programmatic copies fail, the user can still select-all and copy by hand.

## Selection + export wiring (copy-paste core)

```javascript
var sel = {};                                   // group -> {id,name}
document.querySelectorAll('.demo.pick').forEach(function(card){
  card.addEventListener('click', function(){
    var g = card.getAttribute('data-group');
    document.querySelectorAll('.demo.pick[data-group="'+g+'"]')
      .forEach(function(s){ s.classList.remove('selected') });
    card.classList.add('selected');
    sel[g] = { id: card.getAttribute('data-id'), name: card.getAttribute('data-name') };
    updateSummary();
  });
});
function updateSummary(){
  // reflect sel into the sticky bar pills; enable export button only when all groups chosen
}
function buildExport(){                          // labeled lines, not JSON
  return '=== <decision title> ===\n'
       + 'Group 1: ' + (sel['1'] ? sel['1'].id+' — '+sel['1'].name : '(none)') + '\n'
       + 'Group 2: ' + (sel['2'] ? sel['2'].id+' — '+sel['2'].name : '(none)') + '\n'
       + '====================';
}
```

## Card markup shape

```html
<article class="demo pick" data-group="1" data-id="A" data-name="Spinner overlay on a single row">
  <div class="check"><!-- checkmark svg --></div>
  <span class="pickhint">click to pick</span>
  <div class="stage"><!-- the LIVE rendered option --></div>
  <div class="meta"><h3>Title</h3><p>What it is + when to use it</p></div>
</article>
```

Key CSS states:
- `.demo.selected` → accent ring via `box-shadow:0 0 0 2px <accent>` + reveal `.check`, hide `.pickhint`.
- `.demo.no-pick` → a reference/"already exists" card that is NOT clickable (dim slightly, omit `pick`).

## Why labeled text, not JSON or a download

- The user pastes it straight into chat; you read it as plain text. JSON adds brace noise and invites
  copy errors. One `Label: id — name` line per group is unambiguous and human-checkable before sending.
- A downloaded file means the user must find it, open it, and copy — clipboard is one click.

## Gotchas

- Disable the export button until every group has a selection — half-finished picks waste a round-trip.
- Render each option **live** (real animation/layout), not a screenshot — the whole point is judging by eye.
- Keep the export block short. If there are free-text notes, add one optional `<textarea>` the user can
  type into and include its value in `buildExport()`.
- RTL decks: the export bar and modal must respect `dir="rtl"`; keep the copied text readable in RTL too.

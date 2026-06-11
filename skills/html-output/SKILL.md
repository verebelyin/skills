---
name: html-output
description: >
  Produce a single, self-contained HTML file for rich output instead of a long Markdown reply —
  plans, side-by-side comparisons, code reviews, research explainers, interactive prototypes,
  custom editors, dashboards. Reach for it whenever output is dense, visual, comparative,
  interactive, or worth sharing, or the user says "make an HTML file / one-pager / artifact /
  nice for humans".
---

# HTML Output

Prefer a self-contained `.html` file over a long Markdown reply when output is dense, visual,
comparative, interactive, or shareable. HTML carries tables, diagrams, code, color, and two-way
controls Markdown can't — and a file opens/links anywhere.

**Reach for it for:** implementation plans (mockups + data-flow diagram + key code snippets) · N
approaches in a side-by-side grid to compare · code review (render the diff, inline margin
annotations, color by severity) · research/explainer (diagram + 3–4 annotated snippets +
"gotchas") · prototypes (sliders/toggles to try options) · custom editors (draggable cards, forms
with dependency warnings). Use a separate file per stage rather than one giant page.

**Principles:** accurate (real facts/code pulled from the thread — never invent) · self-contained
(one file; deps from CDN) · clean (whitespace, one accent, prose ≤ `max-w-3xl`; tables/diagrams do
the work) · interactive where it earns its keep.

**Make it round-trip.** Any editor/prototype needs an export — a **"Copy as Markdown / JSON /
prompt"** button that turns whatever the user did in the UI back into pasteable text. That export
is the whole reason to build a UI instead of a static doc.

```html
<button onclick="navigator.clipboard.writeText(window.toExport())">Copy as Markdown</button>
<!-- define window.toExport() to serialize current UI state to text -->
```

**Boilerplate** (CDN; add the Mermaid block only if diagramming):

```html
<script src="https://cdn.tailwindcss.com"></script>
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&family=JetBrains+Mono&display=swap" rel="stylesheet">
<!-- body: class="bg-slate-50 text-slate-800"; wrap content in: max-w-6xl mx-auto px-6 (max-w-[1440px] if dense) -->
<script type="module">
  import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.esm.min.mjs';
  mermaid.initialize({startOnLoad:true,theme:'neutral',securityLevel:'loose',themeVariables:{fontFamily:'Inter,sans-serif'},flowchart:{useMaxWidth:true,curve:'basis'}});
</script>
```

**Look:** accent `indigo-600`; headings `text-slate-900`, muted `text-slate-500/600`; sections =
`<section id>` + `border-t border-slate-200 py-12` with a sticky top nav; cards `rounded-xl
bg-white border border-slate-200 p-5`; tables in `overflow-x-auto rounded-xl border`, header
`bg-slate-100`, rows `divide-y`; badges = small pills, one color per category used consistently;
`font-mono` for names/IDs/paths. For comparisons use a responsive grid (`grid md:grid-cols-3
gap-4`); for "what × where" a matrix table (✓/—/value cells, `colspan` group headers).

**Diagrams** (Mermaid, or hand-written SVG for custom visuals): keep flow one-directional — never a
node both read and written (split the hub; a backward arrow means a missing node); label every node
with its **type** and every arrow with an **action**; group nodes in a `subgraph` per boundary, and
split into one diagram **per scope** rather than one crowded one. (Quote labels with special chars;
use `<br>` for line breaks — never `<br/>`; thick labelled edge = `A ==>|text| B`.) Mermaid's
lexer tokenises globally before string context is established — never use `[`, `]`, or `@` inside
label *text* (node labels, subgraph titles, edge labels): they are grammar tokens that cause parse
errors regardless of quoting. Strip or reword rather than escape — there is no reliable escape.
For Mermaid init, prefer `theme: "neutral", securityLevel: "loose"` — `"loose"` enables HTML
labels (`<br>`) and avoids sanitiser-induced parse failures.

**Deliver:** write to the OS temp dir so nothing lands in the repo — resolve from `$TMPDIR`, falling
back to `/tmp` (or `%TEMP%` on Windows), to `<tmpdir>/<slug>-<timestamp>.html` (fresh each run).
Open it — `xdg-open <path>` (Linux), `open <path>` (macOS), `start <path>` (Windows) — and tell the
user the absolute path.

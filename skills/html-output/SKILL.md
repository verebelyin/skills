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
prompt"** button wired to a `toExport()` helper that serialises the current UI state back to
pasteable text. That export is the whole reason to build a UI instead of a static doc.

**Setup** (all CDN, no build): Tailwind from `cdn.tailwindcss.com`; Inter + JetBrains Mono from
Google Fonts; `<body class="bg-slate-950 text-slate-200">`, content in `max-w-6xl mx-auto px-6`
(`max-w-[1440px]` if dense). Add Mermaid only when diagramming — v11 ESM from jsDelivr, `theme:'dark'`
with slate `themeVariables` (background `#020617`, node `#1e293b`, border `#334155`, text `#e2e8f0`,
lines `#64748b`) so diagrams match the dark canvas.

**Look (dark theme):** background `bg-slate-950`, body text `text-slate-200`, font `Inter`; accent
`indigo-400` (brighter than light-mode indigo so it reads on dark); headings `text-slate-100`, muted
`text-slate-400/500`; sections = `<section id>` + `border-t border-slate-800 py-12` with a sticky
top nav (`bg-slate-950/80 backdrop-blur border-b border-slate-800`); cards `rounded-xl bg-slate-900
border border-slate-800 p-5`; code/`<pre>` blocks `bg-slate-900 border border-slate-800 rounded-lg
p-4 text-slate-200 overflow-x-auto font-mono` (hand-colour tokens with `text-rose-400` keyword,
`text-amber-300` string, `text-violet-300` fn, `text-sky-300` prop, `text-emerald-300` tag,
`text-slate-500` comment); tables in `overflow-x-auto rounded-xl border border-slate-800`, header
`bg-slate-800/60`, rows `divide-y divide-slate-800`; badges = small pills, one color per category
used consistently, tinted for dark (`bg-indigo-500/15 text-indigo-300`, swap hue per category);
`font-mono` for names/IDs/paths. For comparisons use a responsive grid (`grid md:grid-cols-3
gap-4`); for "what × where" a matrix table (✓/—/value cells, `colspan` group headers). Keep one
accent; let dark cards + borders carry the structure.

**Diagrams** (Mermaid, or hand-written SVG/HTML for custom visuals): render on the dark canvas —
Mermaid via `theme:'dark'` + the `themeVariables` above (slate node fills, light text, slate-500
lines); for hand-written SVG use the same palette (node fill `#1e293b`/`bg-slate-800`, stroke
`#334155`, text `#e2e8f0`, edges `#64748b`) and `fill="none"` backgrounds so it sits on
`bg-slate-950` — never black-on-white default strokes. For a lightweight tree/flow, plain
HTML boxes (`bg-slate-800 border border-slate-700 rounded px-2 font-mono text-sm`) often beat
Mermaid. Keep flow one-directional — never a
node both read and written (split the hub; a backward arrow means a missing node); label every node
with its **type** and every arrow with an **action**; group nodes in a `subgraph` per boundary, and
split into one diagram **per scope** rather than one crowded one. (Quote labels with special chars;
use `<br>` for line breaks — never `<br/>`; thick labelled edge = `A ==>|text| B`.) Mermaid's
lexer tokenises globally before string context is established — never use `[`, `]`, or `@` inside
label *text* (node labels, subgraph titles, edge labels): they are grammar tokens that cause parse
errors regardless of quoting. Strip or reword rather than escape — there is no reliable escape.
For Mermaid init, use `theme: "dark", securityLevel: "loose"` with the palette `themeVariables`
above — `"loose"` enables HTML labels (`<br>`) and avoids sanitiser-induced parse failures.

**Zoomable diagrams.** Inline diagrams are small — give each `.mermaid` a small icon button in its
top-right corner that opens the diagram in a viewport-filling dark overlay, **fit-to-screen** (not
zoomed in), with scroll-to-zoom and drag-to-pan, closeable via Esc / click-out / ✕. (Render with
`startOnLoad:false` then `await mermaid.run()` before wiring the buttons.)

**Deliver:** write to the OS temp dir so nothing lands in the repo — resolve from `$TMPDIR`, falling
back to `/tmp` (or `%TEMP%` on Windows), to `<tmpdir>/<slug>-<timestamp>.html` (fresh each run).
Open it — `xdg-open <path>` (Linux), `open <path>` (macOS), `start <path>` (Windows) — and tell the
user the absolute path.

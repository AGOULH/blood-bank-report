# Blood Bank Dammam — Year-over-Year Report

A single self-contained HTML file (`index.html`, originally
`blood_bank_yoy_report.html`) that renders an
editable, printable year-over-year statistics report for the **Central Blood
Bank, Dammam** (Saudi Ministry of Health). No build step, no package manager
— open the file directly in a browser. It loads one external script
(`html2pdf.js` via CDN, see "Save Report → automatic PDF" below) to generate
PDFs without a print dialog, but there's still no backend/server code of our
own. Everything else is local: the entry form starts blank every time it's
opened (see "Editing model" below), and anything the user chooses to keep
lives only in that browser's `localStorage` (see "Saved reports" below).

This is a separate project from the "blood-bank-inventory" app in
`~/Downloads/index.html` — that one tracks day-to-day inventory with an
in-app user guide; this one is a static-ish YoY statistics report with no
GUIDE array.

Deployed via GitHub Pages from `main`/root at
https://agoulh.github.io/blood-bank-report/ (repo is public so free-plan
Pages can serve it).

## What it shows

The report is a blank data-entry form: every field starts at `0`/empty on
load, the user types in raw counts for the two years, and every percentage,
donut, and trend arrow is computed live from what's typed — no numbers are
baked into the file. Three organizational sections (`BLOOD BANK CENTER`,
`DAMY`, `MOBILE`), each
comparing two years (2024 vs 2025) side by side across four metrics:

- **DONOR** — total donor count
- **FEMALE** — female donor count
- **RHNEG** — Rh-negative donor count
- **WASTE** — total wasted units, broken into 5 sub-reasons: QNS,
  Air infusion, Damaged, Clotted, Over Volume (WASTE is always the sum of
  these five — never edited directly, only derived)

Each metric renders as a donut showing its percentage:
- DONOR% = this section's donors ÷ total donors across **all sections** that year
- FEMALE/RHNEG/WASTE% = value ÷ this section's own DONOR count that year

A trend arrow between the two years is colored green ("good") or red ("bad").
Direction logic is inverted for WASTE (a decrease is "good"); for the other
three, an increase is "good". Each section also has a 4-quadrant "wheel"
donut (editable center label, defaults to a quarter like `Q1`) and an "RH
Details" line for O-negative count.

## Structure of `index.html`

- `<style>` — all CSS, CSS custom properties for the color palette at the top (`:root`)
- `<div class="header">` — MOH logo (inline base64 PNG) + Arabic/English branding
- `<div class="toolbar">` — Recalculate / Save Report / Clear All / Print-PDF /
  Export / Import / Saved Reports buttons
- `<div id="sections">` — the entry view; populated at runtime by `render()`
- `<div id="savedReportsView">` — the second "page": a list of previously-saved
  report snapshots, toggled with `#sections` by `showSavedView()` / `showEntryView()`
  (both live in the same `index.html`, not separate files — see "Working on this file")
- `<div class="autosave-note">` — reminds the user nothing is saved automatically; hidden in print
- `<div class="print-signature">` — signature line ("done bye : Hasan Alagoul");
  `display:none` on screen, forced `display:block` only inside `@media print`
- `<script src=".../html2pdf.bundle.min.js">` — the one external dependency (CDN,
  cdnjs); wraps `html2canvas` + `jsPDF` to generate a PDF from the DOM without a
  print dialog
- `<script>` — all app logic:
  - `blankYear()` / `BLANK` — the all-zero starting template (source of truth for
    both the initial `DATA` and "Clear All"); section names (`BLOOD BANK CENTER`,
    `DAMY`, `MOBILE`) and year labels (`2024`/`2025`) are pre-filled, all numbers are `0`
  - `REPORTS_KEY = "bloodbank_yoy_reports_v1"` — localStorage key holding the array
    of saved-report snapshots (see "Saved reports" below)
  - `DATA` — the working copy for the entry view; always starts as a fresh clone of
    `BLANK` on page load — nothing restores it from storage automatically
  - `ICONS`, `LABELS`, `COLORS`, `METRIC_KEYS`, `WHEEL_QUADS`, `WHEEL_DEFAULT_LABELS` — static config for the 4 metrics
  - `render()` — builds the DOM for all sections from `DATA`
  - `yearColHTML()` — builds one year's column (donuts, arrows, icons, values, waste/RH details)
  - `wheelSVG(sec)` — draws the 4-quadrant wheel per section; each quadrant's text label
    is also `contenteditable` (`data-wheel-label`) and syncs back to `sec.wheelLabels`
  - `attachListeners()` — wires `blur`/`Enter` on every `[contenteditable]` to re-read and recalc
  - `readDOMIntoData()` — reverse-syncs edited DOM values back into `DATA`
  - `recalcAll()` — recomputes WASTE totals, all percentages, donut fills, and trend arrows; the only place derived values are computed. Purely in-memory/DOM — no storage side effect.
  - `saveReport()` / `clearAll()` — snapshot the current entry into saved-report
    history (and trigger a PDF download, see below), and wipe the entry view
    back to `BLANK`, respectively
  - `generatePDF(filename)` / `arabicTextToImage(...)` / `overlayArabicImage(...)`
    — PDF export, called by `saveReport()`; see "Save Report → automatic PDF" below
  - `loadReports()` / `saveReports(list)` / `renderSavedList()` / `openReport(id)` /
    `deleteReport(id)` — read/write the saved-report history and render/act on
    `#savedReportsView`'s list
  - `showSavedView()` / `showEntryView()` — toggle which of `#sections` /
    `#savedReportsView` (and the toolbar) is visible
  - `exportData()` / `importData(event)` — download the current on-screen `DATA` as
    a JSON file, or load one back in (after a shape check and a confirm); a manual
    backup/restore path independent of the saved-reports history
  - `flash(msg)` — briefly swaps the toolbar hint text (e.g. "Saved ✓") after an action

## Editing model

Every number and label in the report is a `contenteditable` span/text node
(`data-field`, `data-metric`, `data-k`, `data-wheel-label`, `data-wheel-center`
attributes identify what each element maps to). On blur, `readDOMIntoData()`
pulls the edited DOM back into `DATA` and `recalcAll()` recomputes everything
derived — nothing is written to storage at this point. Reloading the page (or
just navigating away) silently discards whatever's on screen unless the user
explicitly pressed "Save Report" first. This is intentional: the report is
meant to be a disposable scratch form, not a document that auto-persists
edits in the background (that used to sync live across browsers via
Firestore; that mechanism was deliberately removed — data storage is now
purely local/per-browser via `localStorage`, no server or account involved).

The WASTE number under each donut is display-only (no `contenteditable`,
just a `title` tooltip) because `recalcAll()` always overwrites it with the
sum of the 5 waste sub-reason fields below it — it must never carry its own
`contenteditable`, or edits to it get silently discarded on blur.

## Saved reports (localStorage)

"Save Report" doesn't overwrite anything and doesn't sync anywhere — it
appends a timestamped, deep-cloned snapshot of the current `DATA` to a list
in `localStorage` under `REPORTS_KEY`. That list is what backs the
`#savedReportsView` second page:

- `saveReport()` reads the DOM, recalculates, then unshifts
  `{id, savedAt, data}` onto the list and writes it back — it does **not**
  clear or otherwise touch the live entry view.
- `showSavedView()` hides `#sections`/the toolbar and renders the list
  (newest first) via `renderSavedList()`; each row has **Open** and **Delete**.
- `openReport(id)` replaces `DATA` with a clone of that snapshot, re-renders,
  and switches back to the entry view — from there it's fully editable again,
  and pressing "Save Report" again creates a *new* history entry rather than
  overwriting the one just opened.
- `deleteReport(id)` removes one entry from the list (with a confirm).
- This history is per-browser only (plain `localStorage`, no server). Export/
  Import (see above) are the way to move a report's data to another machine
  or take an out-of-browser backup — the saved-reports list itself doesn't
  travel with the file.

## Save Report → automatic PDF

Pressing "Save Report" does two independent things: appends a snapshot to the
saved-reports history (above), and calls `generatePDF()` to download a PDF of
the report immediately — no print dialog, no manual "Save as PDF" step.

`generatePDF()` uses `html2pdf.js` (`html2canvas` + `jsPDF` under the hood) to
rasterize `.sheet` and wrap it in a PDF, after hiding `.toolbar` and
`.autosave-note` (restored in a `finally`).

**Known `html2canvas` limitation, and the workaround already in place:**
`html2canvas` does not correctly shape/reorder Arabic (or other complex-script)
text — the header's Arabic lines (`.brand-ar` "وزارة الصحة" and `.brand-sub`
"بنك الدم المركزي بالدمام") would come out with scrambled letter/word order if
captured as live DOM text. Neither `letterRendering` nor `foreignObjectRendering`
html2canvas options fix this reliably (the latter fixes the text but crops/
mispositions the rest of the page — do not re-enable it without re-verifying
the whole layout). The actual fix: `arabicTextToImage()` renders each Arabic
line to an offscreen `<canvas>` using the browser's own `fillText()` (which
*does* shape Arabic correctly, since it goes through the OS text engine, not
html2canvas's own layout code) and produces a PNG. `overlayArabicImage()`
makes the live text transparent (keeping its box/border/padding intact — the
divider line under "MINISTRY OF HEALTH" lives on `.brand-sub`'s `border-top`)
and absolutely-positions that PNG centered on top, only for the duration of
the capture; both are removed and the original text/color restored in
`generatePDF()`'s `finally` block. If you add more Arabic (or other RTL/
complex-script) text anywhere that might end up in a PDF, it needs the same
image-overlay treatment — plain text in that position will scramble.

## Print layout

`@media print` has its own compressed CSS (smaller donuts/icons/fonts, tighter
section padding, `.toolbar` and `.autosave-note` hidden) so the full report —
all three sections, both years — fits a single printed page, plus a forced-visible
signature line at the bottom. It also force-hides `#savedReportsView` and
force-shows `#sections`, so printing while the Saved Reports list happens to
be open still prints the entry data, not the list. This block has been
iterated on repeatedly and is fragile: the `@media (max-width:760px)` mobile
breakpoint can interact badly with print pagination (a browser printing at a
narrow viewport width picks up mobile rules too), so mobile and print rules
should be checked together, not assumed independent. When touching print CSS,
verify with an actual print preview, not just by reading the rules.

## Working on this file

- Keep edits confined to the existing three `<script>` blocks (style / data /
  logic) rather than introducing new files — the app is intentionally a single
  portable HTML file.
- If you add a new metric or section, update `METRIC_KEYS`/`LABELS`/`COLORS`/
  `ICONS`/`WHEEL_QUADS` together, and extend `BLANK` with matching shape
  (`years.<year>.metrics`, `years.<year>.waste`, `years.<year>.rhoneg`, all
  zeroed via `blankYear()`) so `clearAll()` and the initial blank load still work.
- `recalcAll()` is the single place percentages/derived values are computed —
  don't duplicate that math elsewhere.

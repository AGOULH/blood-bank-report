# Blood Bank Dammam — Year-over-Year Report

A single self-contained HTML file (`index.html`, originally
`blood_bank_yoy_report.html`) that renders an
editable, printable year-over-year statistics report for the **Central Blood
Bank, Dammam** (Saudi Ministry of Health). No build step, no package manager
— open the file directly in a browser. It does talk to external services:
Firestore (see "Saved reports" below) so saved reports show up the same in
any browser/device, and `html2pdf.js`/CDN so "Print / PDF" downloads a PDF
directly (see "PDF export" below) — but there's still no custom backend/
server code of our own. The entry form itself starts blank every time it's
opened (see "Editing model" below) and is never auto-saved — only an
explicit "Save Report" press persists anything.

This is a separate project from the "blood-bank-inventory" app in
`~/Downloads/index.html` — that one tracks day-to-day inventory with an
in-app user guide; this one is a static-ish YoY statistics report with no
GUIDE array.

Deployed via GitHub Pages from `main`/root at
https://agoulh.github.io/blood-bank-report/ (repo is public so free-plan
Pages can serve it).

**Caching gotcha:** GitHub Pages serves this file with `cache-control:
max-age=600` and there's no way to override that from a static Pages site
(no custom headers support). Combined with the browser's own disk/back-forward
cache, this means a real user's browser can keep showing an old version for
up to ~10 minutes after a push — and has repeatedly been mistaken for a code
bug mid-session (someone reports X is broken, the deployed source already has
the fix, a hard refresh / incognito load resolves it every time). `<meta
http-equiv="Cache-Control/Pragma/Expires">` tags are in `<head>` as a
best-effort mitigation, but they don't reliably beat a real HTTP
`cache-control` header in every browser. **Before debugging a "still broken"
report against the live site, fetch the live HTML directly (`curl` or a
fresh Playwright context, not the reporter's browser) and compare it to what
was actually pushed — if they already match, it's stale cache, not a bug.**

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
- `<script src=".../firebase-app-compat.js">` / `<script src=".../firebase-firestore-compat.js">`
  — CDN (gstatic.com), for the saved-reports Firestore sync
- `<script src=".../html2pdf.bundle.min.js">` — CDN (cdnjs); wraps `html2canvas` +
  `jsPDF` so "Print / PDF" downloads a PDF directly (see "PDF export" below)
- `<script>` — all app logic:
  - `blankYear()` / `BLANK` — the all-zero starting template (source of truth for
    both the initial `DATA` and "Clear All"); section names (`BLOOD BANK CENTER`,
    `DAMY`, `MOBILE`) and year labels (`2024`/`2025`) are pre-filled, all numbers are `0`
  - `firebaseConfig` / `reportsCol` — Firestore collection `bloodbank_yoy_reports`
    that backs the saved-report history (see "Saved reports" below)
  - `reportsCache` — local in-memory mirror of that collection, kept live by an
    `onSnapshot` listener set up right after `reportsCol` is created
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
  - `saveReport()` / `clearAll()` — snapshot the current entry into Firestore's
    saved-report history, and wipe the entry view back to `BLANK`, respectively
  - `generatePDF()` / `arabicTextToImage(...)` / `overlayArabicImage(...)` — PDF
    export, called by the "Print / PDF" button; see "PDF export" below
  - `renderSavedList()` / `openReport(id)` / `deleteReport(id)` — render
    `reportsCache` into `#savedReportsView`'s list, and act on one entry
    (Firestore doc IDs are strings — don't drop the quotes in the generated
    `onclick="openReport('${r.id}')"` handlers)
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
edits in the background. (An earlier version synced every keystroke live
across browsers via Firestore — that's gone; only an explicit "Save Report"
writes anywhere now, and only a full snapshot, never individual edits.)

The WASTE number under each donut is display-only (no `contenteditable`,
just a `title` tooltip) because `recalcAll()` always overwrites it with the
sum of the 5 waste sub-reason fields below it — it must never carry its own
`contenteditable`, or edits to it get silently discarded on blur.

## Saved reports (Firestore)

"Save Report" doesn't overwrite anything — it adds a new document to the
`bloodbank_yoy_reports` Firestore collection (project `lab-analytics-555c7`,
same free-tier project the old live-sync used), each holding a full
`{savedAt, data}` snapshot. Every open browser/device sees the same list,
live, because of the collection-wide `onSnapshot` listener set up at load
(`reportsCol.orderBy('savedAt','desc').onSnapshot(...)` → `reportsCache` →
`renderSavedList()`) — no manual refresh needed, a save in one tab shows up
in another automatically. This is deliberately scoped to *only* the saved-report
history; the live entry form (`DATA`) stays local/blank-on-load, as decided
in "Editing model" above — don't wire Firestore back into the typing/blur
path, that's the live-sync behavior that was intentionally removed.

- `saveReport()` reads the DOM, recalculates, then `reportsCol.add({savedAt,
  data})` — Firestore assigns the doc ID; on failure it alerts the user
  rather than silently losing the save (data loss here is a real risk since
  there's no local fallback copy).
- `showSavedView()` hides `#sections`/the toolbar and calls `renderSavedList()`
  (it's usually already fresh from the live listener, this just guarantees
  it); each row has **Open** and **Delete**.
- `openReport(id)` finds the doc in `reportsCache`, clones its `data` into
  `DATA`, re-renders, and switches back to the entry view — from there it's
  fully editable again, and pressing "Save Report" again creates a *new*
  document rather than overwriting the one just opened.
- `deleteReport(id)` deletes that one Firestore doc (with a confirm); the
  list updates via the live listener, no manual re-render needed.
- Firestore security rules are wide open (`allow read, write: if true`) —
  same as the old live-sync setup, no auth. Acceptable for this low-stakes
  internal tool; anyone with the page source can read/write the collection
  directly, bypassing the UI.
- Export/Import (above) still exist as a manual/offline fallback — e.g. an
  out-of-browser backup, or moving one report's data somewhere with no
  network access to Firestore.

## PDF export

The "Print / PDF" button calls `generatePDF()` directly — it downloads a PDF
immediately, no print dialog. (Earlier it called `window.print()`; that was
changed because a print dialog read as "it redirected me to the browser" —
users expect one click to produce a file.) `generatePDF()` rasterizes `.sheet`
via `html2pdf.js` (`html2canvas` + `jsPDF`), after hiding `.toolbar` and
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

`html2canvas` also scrolls the window while capturing and does not reliably
restore the original scroll position, which visibly jumped the page (header/
toolbar scrolled out of view) if left unhandled. `generatePDF()` captures
`window.scrollX/scrollY` up front and calls `window.scrollTo()` back to it in
the `finally` block — don't remove that, or the scroll-jump comes back.

This was tried, reverted (in favor of `window.print()`), then reinstated
after the print-dialog UX turned out to be the actual complaint — both
`html2canvas` bugs above were already solved before the revert, so bringing
it back was just re-adding the same verified code, not re-debugging from
scratch. If it needs to be reverted again, `git log` has the full history of
both the addition and the revert to compare against.

## Print layout

`@media print` has its own compressed CSS (smaller donuts/icons/fonts, tighter
section padding, `.toolbar` and `.autosave-note` hidden) so the full report —
all three sections, both years — fits a single printed page, plus a forced-visible
signature line at the bottom. It also force-hides `#savedReportsView` and
force-shows `#sections`, so printing while the Saved Reports list happens to
be open still prints the entry data, not the list. **Since the "Print / PDF"
button now calls `generatePDF()` instead of `window.print()`, this CSS is only
reached via the browser's own manual print action (Ctrl/Cmd+P, right-click →
Print) — it's still there and still worth keeping correct, just no longer the
primary path.** This block has been iterated on repeatedly and is fragile: the
`@media (max-width:760px)` mobile breakpoint can interact badly with print
pagination (a browser printing at a narrow viewport width picks up mobile
rules too), so mobile and print rules should be checked together, not assumed
independent. When touching print CSS, verify with an actual print preview,
not just by reading the rules.

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
- "Save Report" (Firestore) and "Print / PDF" (`generatePDF()`) are deliberately
  separate actions with separate failure modes — don't merge them back into one
  button's click handler. Keep PDF generation's Arabic-text/scroll workarounds
  (see "PDF export" above) intact if you touch `generatePDF()`.

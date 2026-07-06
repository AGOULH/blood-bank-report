# Blood Bank Dammam — Year-over-Year Report

A single self-contained HTML file (`index.html`, originally
`blood_bank_yoy_report.html`) that renders an
editable, printable year-over-year statistics report for the **Central Blood
Bank, Dammam** (Saudi Ministry of Health). No build step, no package manager
— open the file directly in a browser. It does talk to one external
service (Firestore, see "Cross-browser sync" below) for live data sync, but
there's still no custom backend/server code of our own.

This is a separate project from the "blood-bank-inventory" app in
`~/Downloads/index.html` — that one tracks day-to-day inventory with an
in-app user guide; this one is a static-ish YoY statistics report with no
GUIDE array.

Deployed via GitHub Pages from `main`/root at
https://agoulh.github.io/blood-bank-report/ (repo is public so free-plan
Pages can serve it).

## What it shows

Three organizational sections (`BLOOD BANK CENTER`, `DAMY`, `MOBILE`), each
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
- `<div class="toolbar">` — Recalculate / Save / Reset to original / Print-PDF /
  Export / Import buttons
- `<div id="sections">` — populated at runtime by `render()`
- `<div class="autosave-note">` — "edits are saved automatically" note; hidden in print
- `<div class="print-signature">` — signature line ("done bye : Hasan Alagoul");
  `display:none` on screen, forced `display:block` only inside `@media print`
- `<script>` — all app logic, no external JS dependencies:
  - `ORIGINAL` — the baked-in seed dataset (source of truth for "Reset to original")
  - `STORE_KEY = "bloodbank_yoy_report_v3"` — localStorage key holding live edits
  - `DATA` — the working copy loaded from localStorage (or cloned from `ORIGINAL`)
  - `ICONS`, `LABELS`, `COLORS`, `METRIC_KEYS`, `WHEEL_QUADS`, `WHEEL_DEFAULT_LABELS` — static config for the 4 metrics
  - `render()` — builds the DOM for all sections from `DATA`
  - `yearColHTML()` — builds one year's column (donuts, arrows, icons, values, waste/RH details)
  - `wheelSVG(sec)` — draws the 4-quadrant wheel per section; each quadrant's text label
    is also `contenteditable` (`data-wheel-label`) and syncs back to `sec.wheelLabels`
  - `attachListeners()` — wires `blur`/`Enter` on every `[contenteditable]` to re-read, recalc, persist
  - `readDOMIntoData()` — reverse-syncs edited DOM values back into `DATA`
  - `recalcAll()` — recomputes WASTE totals, all percentages, donut fills, and trend arrows; the only place derived values are computed
  - `persist()` / `saveData()` / `resetData()` — localStorage + Firestore write, and the
    "Reset to original" flow
  - `exportData()` / `importData(event)` — download `DATA` as a JSON file, or load one back
    in (after a shape check and a confirm); a manual/offline fallback for moving data around
    now that Firestore sync (below) handles the common case automatically
  - `flash(msg)` — briefly swaps the toolbar hint text (e.g. "Saved ✓") after Save/Reset

## Editing model

Every number and label in the report is a `contenteditable` span/text node
(`data-field`, `data-metric`, `data-k`, `data-wheel-label`, `data-wheel-center`
attributes identify what each element maps to). On blur, `readDOMIntoData()`
pulls the edited DOM back into `DATA`, `recalcAll()` recomputes everything
derived, and `persist()` saves the result.

## Cross-browser sync (Firestore)

`localStorage` alone is per-browser/per-device and never syncs on its own —
edits made in one browser were invisible everywhere else. `index.html` now
also loads the Firebase compat SDK (`firebase-app-compat.js` +
`firebase-firestore-compat.js`, via `<script src>` from the `gstatic.com` CDN
— the one external dependency this file has) and mirrors `DATA` to a single
Firestore document:

- Project: `lab-analytics-555c7` (Google Firebase, free tier), document path
  `bloodbank_yoy_report/data`.
- `persist()` writes to both `localStorage` (instant local cache) and that
  Firestore document (`remoteDoc.set(DATA)`), fire-and-forget.
- An `onSnapshot` listener on `remoteDoc` (set up right after `DATA` is first
  loaded) pushes any remote change into `DATA` + `render()` in real time —
  this is what makes edits show up automatically in every other open browser,
  no manual export/import needed for the common case. It skips snapshots
  where `metadata.hasPendingWrites` is true, so a browser doesn't re-render
  its own just-written change as if it came from elsewhere.
- `resetData()` calls `persist()` too, so "Reset to original" also propagates
  to every other browser instead of only resetting the local one.
- Firestore security rules are wide open (`allow read, write: if true`) —
  there's no auth. The `apiKey` in `firebaseConfig` is not a secret (Firebase
  web API keys are meant to be public; access control lives in the rules,
  not the key), but be aware anyone with the page's source can read/write
  this document directly, bypassing the UI entirely. Acceptable for this
  low-stakes internal report; revisit if that assumption ever stops holding.
- Export/Import (previous section) still exist as a manual/offline fallback
  — e.g. taking a point-in-time backup, or moving data somewhere with no
  network access to Firestore.

## Print layout

`@media print` has its own compressed CSS (smaller donuts/icons/fonts, tighter
section padding, `.toolbar` and `.autosave-note` hidden) so the full report —
all three sections, both years — fits a single printed page, plus a forced-visible
signature line at the bottom. This block has been iterated on repeatedly and is
fragile: the `@media (max-width:760px)` mobile breakpoint can interact badly
with print pagination (a browser printing at a narrow viewport width picks up
mobile rules too), so mobile and print rules should be checked together, not
assumed independent. When touching print CSS, verify with an actual print
preview, not just by reading the rules.

## Working on this file

- Keep edits confined to the existing three `<script>` blocks (style / data /
  logic) rather than introducing new files — the app is intentionally a single
  portable HTML file.
- If you add a new metric or section, update `METRIC_KEYS`/`LABELS`/`COLORS`/
  `ICONS`/`WHEEL_QUADS` together, and extend `ORIGINAL` with matching shape
  (`years.<year>.metrics`, `years.<year>.waste`, `years.<year>.rhoneg`) so
  `resetData()` still works.
- `recalcAll()` is the single place percentages/derived values are computed —
  don't duplicate that math elsewhere.

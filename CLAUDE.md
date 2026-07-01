# Blood Bank Dammam — Year-over-Year Report

A single self-contained HTML file (`blood_bank_yoy_report.html`) that renders an
editable, printable year-over-year statistics report for the **Central Blood
Bank, Dammam** (Saudi Ministry of Health). No build step, no backend, no
package manager — open the file directly in a browser.

This is a separate project from the "blood-bank-inventory" app in
`~/Downloads/index.html` — that one tracks day-to-day inventory with an
in-app user guide; this one is a static-ish YoY statistics report with no
GUIDE array.

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

## Structure of `blood_bank_yoy_report.html`

- `<style>` — all CSS, CSS custom properties for the color palette at the top (`:root`)
- `<div class="header">` — MOH logo (inline base64 PNG) + Arabic/English branding
- `<div class="toolbar">` — Recalculate / Save / Reset to original / Print-PDF buttons
- `<div id="sections">` — populated at runtime by `render()`
- `<script>` — all app logic, no external JS dependencies:
  - `ORIGINAL` — the baked-in seed dataset (source of truth for "Reset to original")
  - `STORE_KEY = "bloodbank_yoy_report_v3"` — localStorage key holding live edits
  - `DATA` — the working copy loaded from localStorage (or cloned from `ORIGINAL`)
  - `ICONS`, `LABELS`, `COLORS`, `METRIC_KEYS`, `WHEEL_QUADS` — static config for the 4 metrics
  - `render()` — builds the DOM for all sections from `DATA`
  - `yearColHTML()` — builds one year's column (donuts, arrows, icons, values, waste/RH details)
  - `wheelSVG(sec)` — draws the 4-quadrant wheel per section
  - `attachListeners()` — wires `blur`/`Enter` on every `[contenteditable]` to re-read, recalc, persist
  - `readDOMIntoData()` — reverse-syncs edited DOM values back into `DATA`
  - `recalcAll()` — recomputes WASTE totals, all percentages, donut fills, and trend arrows; the only place derived values are computed
  - `persist()` / `saveData()` / `resetData()` — localStorage read/write and the "Reset to original" flow

## Editing model

Every number and label in the report is a `contenteditable` span/text node
(`data-field`, `data-metric`, `data-k`, `data-wheel-label`, `data-wheel-center`
attributes identify what each element maps to). On blur, `readDOMIntoData()`
pulls the edited DOM back into `DATA`, `recalcAll()` recomputes everything
derived, and `persist()` saves to localStorage. Nothing is sent to a server —
all state lives in the browser (`localStorage`) until "Reset to original" is
clicked, which restores `ORIGINAL` and clears the stored key.

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

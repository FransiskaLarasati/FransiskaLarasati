# Mahima Staff Roster — handoff notes

Staff roster, leave, documents and payroll site for **Mahima Tennis, Padel & Gym** (Bali).
Owner: Fransiska (Operations). Built with Claude Code; this file is the project context for whoever continues it.

- **Live site (claude.ai Artifact):** https://claude.ai/artifact/WPvooAHnUSACqgzWrkKifK
- **Code:** `mahima-roster/index.html` — one self-contained page (HTML + CSS + vanilla JS, no build step).
- **Branch:** `claude/inspiring-cerf-ypx3q6`
- **Source spreadsheets (Fransiska's Google Drive):** `008. MAHIMA - Master` (roster, AL/EO/PH tracker, holidays, shift legend) and `005. Mahima Payroll 2026` (salary, PPh 21 tax, bank accounts). Both were imported once; the site is now the source of truth.

> ⚠️ This repository is **public**. Never commit salaries, bank accounts, service-charge amounts, contracts, medical documents or other personal HR data. All of that lives in the artifact's private database (see below), not in this repo.

## How to work on it

1. Edit `mahima-roster/index.html`.
2. Publish with the Claude Code **Artifact** tool, passing `url: https://claude.ai/artifact/WPvooAHnUSACqgzWrkKifK` and `file_path: mahima-roster/index.html`. From a new conversation you must `read` the artifact first, then publish with `url` (otherwise a new, separate artifact is created).
3. **Do not pass `capabilities`** on a normal redeploy — omitting it keeps the stored declaration. If you ever must restate it, restate all of it:
   ```json
   {"db": {"rules": [
     {"path": "sickdocs",      "read": "admin", "write": "admin"},
     {"path": "hrdocs",        "read": "admin", "write": "admin"},
     {"path": "servicecharge", "read": "admin", "write": "admin"},
     {"path": "payroll",       "read": "admin", "write": "admin"}]},
    "downloads": true, "assets": {}}
   ```
4. Read/write live data with the **ArtifactData** tool (same `url`). Pin writes with `if_version`.
5. You need **Editor** access on the artifact (Fransiska grants it in the page's Share menu) to publish or read the private collections.
6. Commit and push the HTML after each change. Before publishing, run `node --check` on the extracted `<script>` and do one browser check (see Testing).

## Architecture

- Runtime capabilities used: `db` (shared JSON document store), `assets` (file uploads: PDFs/photos), `downloads` (CSV/PDF export).
- Everything reaches capabilities through `await window.claude.use("db" | "assets" | "downloads")`; each can resolve `null` (e.g. a viewer without edit rights gets `assets = null`). The page must keep working when they are null.
- PDFs (payslips, THR slips) are generated in the browser with **jsPDF 2.5.1** from cdnjs.
- Rendering is plain template strings + `renderAll()`; state is the global `S` object; snapshots from `onSnapshot` refill `S` and trigger a debounced re-render.
- Week edits are coalesced per week document (`queueRow` → `flush`) to avoid write storms.

### Database collections

| Path | Contents | Who can read |
|---|---|---|
| `staff/<id>` | name, title, dept, contract (`PERM`/`TEMP`/`DW`), start, eoFwd (EO balance on 1 Oct 2026), phGroup (`Hindu`/`Catholic`/`None`), order, archived | everyone with access |
| `weeks/<monday ISO date>` | `cells{staffId:[7 codes]}`, `short{staffId:[7 0/1]}`, `remarks{staffId}` | everyone |
| `requests/<id>` | leave requests: staffId, type, from, to, note, status (`pending`/`approved`/`rejected`) | everyone |
| `config/holidays` | `items[{date,name,for,note}]` | everyone |
| `sickdocs/<id>` | proof for sick days: staffId, from, to, kind, assetId | Editors only |
| `hrdocs/<id>` | contracts, offer letters, payslips, THR slips: staffId, kind, month, from/until, assetId | Editors only |
| `servicecharge/grades` + `servicecharge/<YYYY-MM>` | service-charge grades and saved monthly splits | Editors only |
| `payroll/salaries`, `payroll/<YYYY-MM>`, `payroll/thr-<event>` | default pay per staff, monthly payroll drafts, THR runs | Editors only |

Access model: the artifact **owner (Fransiska) and Editors** see everything; **Contributors** (supervisors) can use roster and leave but the private collections read as empty. Salary data is meant to be visible to Fransiska and the owner Bjorn Yannick only — so only they should be Editors. Link sharing ("anyone with the link") strips Editor rights from people outside the organisation.

## Business rules (agreed with Fransiska)

**Roster**
- Shifts are stored as keys `M`, `M2`, `M3`, `A`, `N` but always **displayed as 8-hour ranges**: 06.30–14.30, 08.00–16.00, 12.00–20.00 (middle / manager on duty), 15.00–23.00, 22.30–06.30. Custom start times (`HH:MM`) also display as start + 8 h. Letter codes must not be shown to users.
- Other codes: `OFF` (1 per week, "No day off" warning otherwise), `EO`, `PH`, `AL`, `SICK` (Sick and "did not come" are the same code; typed `DC` becomes `SICK`), `X` (first day).
- Blue dot = short time (shorter shift).
- `MD`/`MOD` were removed (old cells converted to M3). `L` exists in old imported data with unknown meaning.

**Leave**
- Counting starts **1 Oct 2026**: every Permanent/Temporary staff member starts at **17 = 12 AL + 5 PH**. "Left of 17" = 17 − AL − EO − PH taken (EO also deducts, as in the original sheet).
- Everyone on staff by 1 Oct 2026 can use leave from 1 Oct 2026; later joiners after 1 year of service.
- EO balances were carried forward to 1 Oct (September EO usage subtracted; can be negative).
- Daily workers / probation (`DW`) get no EO, AL or PH.
- PH groups: Hindu = Galungan ×2, Kuningan ×2, Nyepi; Catholic (Fransiska) = Christmas ×2, Easter ×2, Nyepi.
- **One person per department on leave (AL/EO/PH) per Mon–Sun week.** Requests clashing with another pending/approved request or roster leave in the same department are blocked; approval re-checks; roster edits only warn.
- Every SICK day needs proof: doctor certificate, or manager approval letter, or hospital bill (uploaded to `sickdocs`).

**Contracts**
- `PERM` Permanent — managers, supervisors, and everyone after contract renewal.
- `TEMP` Temporary — less than one year of service; "Renewal due" shows 30 days before the 1-year date.
- `DW` Daily worker / probation.

**Pay**
- Salary follows the contract (defaults imported from the payroll sheet; each month carries over from the previous one). DW pay = days worked on the roster × daily rate.
- **Service charge** paid on the 15th, split by hidden grades: managers & supervisors 100%, pre-opening staff 75%, staff hired in 2026 50% (with one agreed exception, already set in the code's defaults). Share = total × grade ÷ sum of grades. **Grades must never be shown on payslips** (hidden in the UI behind "Edit grades").
- **THR** (holiday allowance) = 50% of basic salary; Hindu staff every Galungan, Catholic staff at Easter and Christmas; paid **2 days before** the holiday. Dates derive from `config/holidays`.
- "Generate all payslips" / "Generate THR slips" make one PDF per person, upload them to `assets` and file them in `hrdocs`.

## Testing

No test suite. The approach used so far:
1. Extract the last `<script>` block and run `node --check` on it.
2. Build a local `test.html` that defines a fake `window.claude.use()` returning an in-memory db (collections filled with *fake or locally exported* data — never commit it), prepend it to `index.html`, and open it with Playwright/Chromium (pre-installed at `/opt/pw-browsers`; global `playwright` module). Click through the four tabs, check for page errors, screenshot.
3. jsPDF can't load in the sandbox, so PDF generation must be checked on the live site.

## Open items

- Payroll ↔ roster mismatches: a few people appear in the payroll sheet but not on the roster (or the reverse), one resigned, and some salaries/bank accounts are missing — rows with Rp 0 on the Payroll page. Fransiska has the list; confirm with her before changing staff records.
- Unknown `L` code in imported roster data.
- THR assumptions to confirm: DW excluded, no tax deducted, no pro-rating under 12 months.
- Service charge: DW currently 50%; no reduction for days not worked.
- Branding: use the **TT Drugs** font and Mahima brand guidelines once the font files (web licence) and guideline PDF are provided; inline the font as an uploaded asset / `@font-face`.
- Staff self-service (own leave balance, leave requests, own payslips with a 4-digit PIN, no Claude account) is **not possible on this artifact** (every viewer must sign in to claude.ai). Proposed: a separate Google Apps Script web app reading from Drive/Sheets, with server-side PIN check and lockout.
- Consider moving this project to a **private repository**.

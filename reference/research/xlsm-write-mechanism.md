# Research #8 — How to write points into `MatchChart 0.3.2.xlsm` so the workbook logic still runs

- Ticket: https://github.com/lpridpath/tennis_MatchChartingProject_tagger/issues/8
- Retrieved / written: 2026-09-06
- Workbook inspected: `reference/MatchChart 0.3.2.xlsm` (MCP macro workbook v0.3.2)

---

## TL;DR (read this first — it reframes the ticket premise)

The ticket assumes the workbook auto-derives score / next row / stats via **`Worksheet_Change`-style VBA event macros**. **That assumption is wrong for this file.** Inspection of the actual `vbaProject.bin` shows:

- **There are NO event macros.** `ThisWorkbook`, `Sheet1`…`Sheet6` code modules are all **empty**. There is **no `Worksheet_Change`, no `Workbook_Open`, no auto-advance-row Sub, no score Sub.**
- The **only** VBA is a single **`Public Function isNetPoint(rally As String) As String`** in `Module1` — a **user-defined function (UDF)** called from worksheet formulas, not an event handler.
- Everything the ticket attributes to "macros" is actually done by **native worksheet formulas**: score derivation, running game/set counters, next-point rows, and stats are ~**8,900 formulas** already filled down the sheets (MATCH 1,019 / Points 7,710 / MatchStats 146). The "next row" is not *created* by a macro — every row down to **row 505 is pre-built with formulas** that simply light up once you type the point.

**Consequence for the whole design:** the hard requirement is **formula recalculation**, not "keep VBA events firing." Formulas recalc **automatically whenever Excel (or LibreOffice) opens/recalcs the file** — you do *not* need a live event loop for them. The one thing that *does* need a real Office app is the **`isNetPoint` UDF** (a JS library cannot execute it). This makes **file-level writes far more viable than the ticket feared**, and means we most likely **do NOT have to reimplement the macro logic** — see §2 and Implications.

---

## What the workbook actually does (from inspection)

Extraction method: `.xlsm` is a zip → `xl/vbaProject.bin` is an OLE container → decoded with **oletools / `olevba` 0.60.2** (`python3 -m oletools.olevba <file>`; `pcodedmp` for p-code). Sheet XML parsed directly for formulas.

Sheets (tab order): `MATCH`, `MatchStats`, `Tables`, `Points`, `Instructions`, `Version info`, `License_Assignment`.

Input layout on **`MATCH`** (header row 17, data from **row 18** down to ~row 505):

| Col | Header | Role |
|-----|--------|------|
| `A` | `Pt` | point number (formula) |
| `Y` | **`1st`** | **charter types 1st-serve notation here (INPUT)** |
| `Z` | **`2nd`** | **charter types 2nd-serve notation here (INPUT)** |
| `AC` | `Rally` | full rally string (INPUT / derived) |

The surrounding columns are formulas, e.g. score/counters `=IF(B14="W",8,IF(B14="N",3,6))`, `=I14+1`, `=IF(B14="T",12,I14)`; and on `Points` `=isNetPoint(MATCH!AC18)`, `=AND(MATCH!K18=2, COUNTIF(Tables!L$1:L$4, MATCH!F18))`; `MatchStats` aggregates via `VLOOKUP`/`COUNTIF` over MATCH.

`isNetPoint` itself is pure string logic (`Replace`/`InStr`/`Mid`/`Left`/`Array`) that classifies a rally as a net point (returns "0/1/2/3"). No I/O, no Excel object model calls — relevant because it is portable to LibreOffice Basic *and* trivial to reimplement in TS if ever needed.

**Bottom line:** write the notation string into `MATCH!Y{row}` (1st) / `MATCH!Z{row}` (2nd) [and `AC{row}` rally], then **cause a recalc**. That is the entire integration.

---

## Approach 1 — Drive a running Office app so formulas (and the UDF) execute

Writing a cell through a live Office app gives a **full, correct recalc including `isNetPoint`**. (It would also fire `Worksheet_Change` — but this file has none, so that is not a factor here.)

### Windows — COM automation (`winax` / PowerShell / edge-js)
- **Feasible and the strongest option.** Node/Electron can drive Excel's `Excel.Application` COM object via **`winax`** (`ActiveXObject('Excel.Application')`; NodeJS 10–23; must `electron-rebuild` against your Electron's ABI). Alternative: shell out to **PowerShell** `New-Object -ComObject Excel.Application` (no native module to rebuild — more robust across Electron upgrades).
- **Hard constraints:** requires **desktop Excel installed + licensed** (Microsoft 365 / Office 2016+; **not** web/store UWP Excel). Set `app.Visible=false`, `app.DisplayAlerts=false`. Setting `Range("Y"&r).Value` recalcs and runs the UDF. Save with `xlOpenXMLWorkbookMacroEnabled` (=52) to keep it `.xlsm`.
- **Gotchas:** COM servers can be left orphaned if the process crashes (must `Quit()` + release); first-run "a program is trying to automate Excel" is generally silent for classic Office but AV/EDR may flag COM automation; no prompt if Excel already trusted.

### macOS — AppleScript / JXA to Microsoft Excel
- **Feasible** on the dev machine. `osascript` (AppleScript or JXA) → `tell application "Microsoft Excel" to set value of range "Y18" of active sheet to "..."`. On a real Excel-for-Mac process this **recalcs and runs `isNetPoint`**. (It would also fire `Worksheet_Change` if one existed — confirmed Excel-for-Mac behavior — but again none here.)
- **Hard constraints:** requires **Excel for Mac installed + licensed**. First scripted access triggers a **macOS TCC "Automation" consent prompt** ("<App> wants to control Microsoft Excel") — one-time per app, but must be granted; in a packaged Electron app you need the `NSAppleEventsUsageDescription` Info.plist key and the app must not be blocked by hardened-runtime without the `com.apple.security.automation.apple-events` entitlement.

### Linux — LibreOffice Calc (UNO)
- **Weak / nice-to-have.** You can drive LibreOffice via **UNO** (`python-uno`, or `soffice --headless`) and write the cell; Calc recalcs formulas fine. The risk is the **`isNetPoint` UDF**: LibreOffice loads VBA under `Option VBASupport 1` (an *emulation* layer, not real VBA) and support is **partial**. `isNetPoint` uses only basic string ops that VBASupport generally handles, so it has a decent chance of running — **but it is not guaranteed**, macro security must be lowered to allow it, and any cell whose formula calls a non-resolving UDF returns `#NAME?`. Treat as unverified until tested on the real file.

---

## Approach 2 — File-level writes with a JS library (`exceljs` / SheetJS `xlsx`)

Write `Y/Z/AC` cells directly into the `.xlsm` zip, **no Office process**.

- **Can macros be preserved in the file?** **Yes.** The VBA project is one opaque blob (`xl/vbaProject.bin`). Both libraries can carry it through:
  - **SheetJS (`xlsx`)**: read with `bookVBA:true` → blob on `workbook.vbaraw`; write `.xlsm` (`bookType:'xlsm'`) with `bookVBA:true` re-embeds it. Community edition; latest is distributed from the SheetJS CDN (`cdn.sheetjs.com`), not npm.
  - **`exceljs`** (npm `exceljs@^4.4.0`): read/write preserves an existing `vbaProject.bin` when present.
- **The catch — recalc, not macros.** A JS library **does not run Excel**, so it writes **cached values only**: it will **not** recompute the ~8,900 formulas and **cannot** execute the `isNetPoint` UDF. Immediately after a library write, score/stats cells hold stale cached values (and any dependent UDF cell is stale).
- **But this is largely self-healing for THIS workbook**, precisely because the logic is formulas + one UDF rather than event macros: **the moment the user opens the file in Excel, Excel recalculates everything** (force it by setting the workbook's `fullCalcOnLoad`/`calcPr forceFullCalc="1"` flag, which both libraries can emit, or the charter presses recalc). At that point score, next-row formulas, stats, and `isNetPoint` all populate correctly. So we do **NOT** have to reimplement the macro logic — we just accept that derived columns are stale until the next Excel open/recalc.
- **When this is fine vs. not:** perfectly fine if the app's model is "write points during charting, Excel is opened afterward for the authoritative recalc/CSV export." **Not fine** if the charter needs the *live* Excel score to update on-screen point-by-point while the JS app owns the write — because nothing recalcs between writes. If live on-screen score is required and we still don't want a live Excel, the only way is to **reimplement score/counter logic in TS** for display (the formulas are simple and readable; `isNetPoint` is ~1 pure function to port) — but the `.xlsm` remains the system of record and Excel still does the authoritative recalc on open.

---

## Approach 3 — Keystroke / UI injection into a focused Excel window (last resort)

Type the string + Enter into the focused cell via OS automation (Windows SendKeys / UI Automation; macOS System Events keystrokes). This genuinely enters the cell as if a human did, so **formulas + UDF recalc naturally**.
- **Fragile:** depends on exact window focus, active cell being the right one, no modal dialogs, correct selection/navigation, and locale (decimal/list separators). Any focus steal corrupts data silently. High maintenance, no error signal. Use only as an emergency fallback, never the primary path.

---

## Per-OS / per-approach verdict

Legend: **GREEN** = recommended / reliable · **YELLOW** = works with caveats · **RED** = avoid.

| Approach | Windows (majority) | macOS (dev) | Linux (nice-to-have) | Does it recalc formulas + run `isNetPoint`? |
|---|---|---|---|---|
| **1. Drive live Office** | **GREEN** — `winax` or PowerShell COM; needs Excel installed+licensed | **GREEN** — AppleScript/JXA; needs Excel + one-time Automation consent | **RED/YELLOW** — LibreOffice UNO writes fine but UDF under VBASupport unverified | **Yes** (Win/Mac full; Linux formulas yes, UDF risky) |
| **2. File-level write (`exceljs`/SheetJS)** | **GREEN** (with `forceFullCalc` + Excel opened later) | **GREEN** (same) | **GREEN** for the write itself (LibreOffice/Excel recalcs on open) | **Not at write time**; **yes on next Excel open** (cached/stale until then) |
| **3. Keystroke/UI injection** | **YELLOW** (fragile) | **YELLOW** (fragile) | **YELLOW** (fragile) | Yes (real entry) |

---

## RECOMMENDATION

**Primary, cross-platform: Approach 2 (file-level write) as the default writer, with `forceFullCalc` set so Excel does the authoritative recalc on open** — because this workbook's logic is formulas + one UDF (no event macros), a library write that preserves `vbaProject.bin` loses nothing permanent; Excel/LibreOffice reconstitutes every derived value on the next open. This is the only path that is truly cross-platform and needs no Office installed to *write*.

Layer a **live-Office writer on top where available**, when point-by-point on-screen score is desired:
- **Windows (majority): Approach 1 via COM** (prefer PowerShell `-ComObject` for Electron-upgrade resilience, or `winax` if you accept `electron-rebuild`). Full live recalc incl. UDF. **GREEN.**
- **macOS (dev): Approach 1 via AppleScript/JXA.** Full live recalc incl. UDF; budget for the one-time Automation consent + Info.plist/entitlement. **GREEN.**
- **Linux: Approach 2 only** (write + open in LibreOffice/Excel). Do **not** rely on the UDF running under LibreOffice VBASupport without testing. **YELLOW.**
- **Approach 3: emergency fallback only.**

Suggested architecture: a `WorkbookWriter` interface with two implementations — `LiveOfficeWriter` (COM on Win, AppleScript on Mac) and `FileWriter` (exceljs/SheetJS, `forceFullCalc=1`, `bookVBA`/vbaProject preserved) — selected by platform + whether Excel is detected. Both target `MATCH!Y/Z/AC{row}`; row navigation is just an incrementing row index (18→505), since formulas for every row already exist.

---

## Hard constraints

- **Approach 1 requires desktop Excel installed + licensed** (Win: Office 2016+/M365, classic desktop not UWP; Mac: Excel for Mac). No live path without it.
- **macOS Automation consent** (TCC) is mandatory for AppleScript control; packaged app needs `NSAppleEventsUsageDescription` and the apple-events entitlement under hardened runtime.
- **`winax`** is a native addon → must be rebuilt per Electron version (`electron-rebuild`); PowerShell COM avoids this.
- **Keep the file `.xlsm`** and always **preserve `xl/vbaProject.bin`** on every write (or `isNetPoint` disappears and `Points` formulas break with `#NAME?`). SheetJS: `bookVBA:true` on read+write. exceljs: preserves existing project.
- **Set full-recalc-on-load** (`calcPr forceFullCalc="1"` / `fullCalcOnLoad`) for the file-write path so stale cached derived cells refresh on open.
- **LibreOffice VBA support is emulation** (`Option VBASupport 1`), partial — the UDF running is unverified; formulas themselves are fine.
- Locale matters for keystroke injection (Approach 3) only.

---

## Implications for architecture (issues #3 / #6 / #7)

1. **The premise that "the workbook's `Worksheet_Change` macros derive score / advance rows / build stats" is false** — it is all **worksheet formulas** (~8,900), pre-filled to row 505, plus **one UDF `isNetPoint`**. Update PRD/CONTEXT wording accordingly ("formula-driven workbook," not "event-macro-driven").
2. **We almost certainly do NOT need to reimplement the macro logic.** There is no event logic to reproduce; formulas recalc on open. The *only* code that can't run outside Office is `isNetPoint` — and Excel runs it on open. Reimplementing anything in TS is required **only if** we want a **live on-screen score without a live Excel process** (then port the handful of simple score/counter formulas + the one `isNetPoint` function — all trivial, pure string/number logic).
3. **Write target is fixed and simple:** `MATCH!Y{row}` (1st), `MATCH!Z{row}` (2nd), `MATCH!AC{row}` (rally); row = 18 + pointIndex, up to 505. "Row navigation" is an integer increment, not a macro-driven insert. If matches can exceed ~488 points, we may need to extend the pre-filled formula range (a template concern).
4. **Writer abstraction** (`LiveOfficeWriter` vs `FileWriter`) lets #6/#7 pick per-platform without changing the input-bridge core; the validated MCP string is the single payload both consume.
5. **System of record integrity:** always round-trip through the real `.xlsm` preserving VBA + styles; never regenerate the workbook from scratch (would drop `vbaProject.bin`, formulas, and formatting).

---

## Sources

- oletools / olevba (extraction tool): https://github.com/decalage2/oletools
- SheetJS — VBA and Macros (`bookVBA`, `vbaraw`): https://docs.sheetjs.com/docs/csf/features/vba/
- SheetJS parse options (`bookVBA`): https://docs.sheetjs.com/docs/api/parse-options/
- ExcelJS (npm): https://www.npmjs.com/package/exceljs
- winax (Node ActiveX/COM): https://www.npmjs.com/package/winax · https://github.com/durs/node-activex · Electron issue: https://github.com/durs/node-activex/issues/98
- LibreOffice — Support for VBA Macros (`Option VBASupport 1`, partial): https://help.libreoffice.org/latest/en-US/text/sbasic/shared/vbasupport.html
- How LibreOffice interprets Excel VBA: https://salivity.github.io/libre-office/article/how-libreoffice-interprets-excel-vba-macros

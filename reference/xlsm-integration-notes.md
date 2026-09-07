# .xlsm Integration Notes (distilled)

Operational facts about `MatchChart 0.3.2.xlsm` that the build depends on. Distilled from the #8
research (full report on branch `research/xlsm-write` → `reference/research/xlsm-write-mechanism.md`).

## Key finding — no macros drive the workbook

- `ThisWorkbook` and the sheet code modules are **empty**. There is **no `Worksheet_Change`, no
  auto-advance-row macro, no score macro**.
- Score / stats are **~8,900 native worksheet formulas**, pre-filled down to **row 505**.
- The only VBA is one user-defined function, **`isNetPoint()`** (Module1), called from formulas.
- ⇒ The requirement is **formula recalculation**, not "keep VBA events firing." **We never
  reimplement the scorer** — the workbook's formulas do it; Excel recalcs them on open.

## Input cell map (sheet `MATCH`, per point, from row 18)

| Column | Holds |
|---|---|
| `Y{row}`  | 1st serve code string |
| `Z{row}`  | 2nd serve code string |
| `AC{row}` | rally code string |

Rows are pre-populated with formulas; the app writes into Y/Z/AC of the current point's row and
moves down. It does **not** need to create rows or compute score.

## Write mechanism — DECISION (ticket #8)

**Hybrid:** v1 = **headless file-level write**; live-Excel drive = **post-MVP**.

- **v1 · headless file-level write** (`exceljs` / SheetJS): write cells directly into the `.xlsm`,
  **preserving the VBA project blob**, with `forceFullCalc` set. Fully cross-platform, **no Excel
  required at charting time**. Formulas do **not** recalc at write time; **Excel recalcs everything
  on next open** — so there is **no live on-screen score during charting** in v1. Our own grammar
  validation still catches malformed strings live.
- **post-MVP · drive live Excel** for a real-time score: Windows COM (PowerShell `-ComObject` /
  `winax`), macOS AppleScript/JXA. Requires desktop Excel installed + licensed. Linux has no live
  path (LibreOffice UDF behavior unverified).

## Implications
- **#3** (builder/validator) unaffected — still produces validated strings; no scorer.
- **#6** (session/workbook targeting) — writes to Y/Z/AC by row; autosave = write the file; unblocked
  now that #8 is resolved.
- **#7** — architecture assumes a `WorkbookWriter` interface with a headless (file-level) impl for
  v1 and a live-Excel impl behind the same interface later.

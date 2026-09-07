# CONTEXT — glossary

Ubiquitous language for this project. Terms only; no implementation detail. Keep it sharp — challenge
any use that conflicts with a definition here.

- **The workbook** — the MatchChart `.xlsm` (`reference/MatchChart 0.3.2.xlsm`). The **system of
  record**: it holds every point, derives score, generates stats and CSVs, holds match metadata, and
  is what gets submitted. We drive it; we do not replace it.

- **Input bridge** — what this app *is*: it captures input, builds a validated MCP code string, and
  writes it into the workbook. It is not a standalone charting app.

- **Input source** — a device/means that produces charting input. **Pluggable**, priority-ordered:
  **Stream Deck XL** (default/primary) → **keyboard** (built-in alternative) → other (later).

- **Token** — one atomic unit of MCP notation (a serve direction, a shot-type letter, a direction
  number, a depth, an error type, a modifier, a point-ender). See `reference/mcp-shorthand-grammar.md`.

- **Code string** — the assembled sequence of tokens for a single **point** (e.g. `5f2f1b3n@`). What
  the workbook stores in the `1st`/`2nd` cell. Our builder produces it; our validator checks it.

- **Point** — one charted rally, entered into one workbook row (cells `1st`, optional `2nd`, `Notes`).
  The unit of navigation.

- **Point cell** — the workbook cell a code string is written into. Concretely, on sheet `MATCH`
  from row 18: `Y` = 1st serve, `Z` = 2nd serve, `AC` = rally.

- **Recalc-on-open** — the workbook has no macros; score/stats are native formulas (pre-filled to row
  505). v1 writes cells headlessly (Excel not running), and Excel recalculates them when the file is
  next opened. So there is no live score during charting in v1.

- **Workbook writer** — the app component that writes code strings into the workbook. v1 = a headless
  file-level writer (`exceljs`); a post-MVP live-Excel writer (COM/AppleScript) can sit behind the
  same interface for a real-time score.

- **Next / previous point** — navigation between workbook rows. There is **no separate "commit"** —
  moving to the next point *is* the commit, and it autosaves.

- **Charting tier** — how much detail a point is charted with: shot-types only (beginner) →
  + direction → + return depth → + court-position modifiers (expert). Every tier must be valid.

- **Validate-as-you-type** — checking a code string is well-formed against the grammar *before* it is
  written to a cell, so malformed points never land.

- **Operator** — the person charting. Novice (wants approachability) or expert (wants efficiency +
  power tools). One operator per match; no real-time collaboration.

- **The coordinator** — Jeff Sackmann, who receives submitted workbooks into the MCP dataset. His
  format is our constraint.

- **Layers** — layer 1 = MCP output vocabulary (locked, the grammar) · layer 3 = input mapping
  (pluggable, customizable). (There is no layer 2 — the workbook is the data model.)

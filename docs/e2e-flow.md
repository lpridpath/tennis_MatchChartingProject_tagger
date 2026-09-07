# E2E User Flow — v0.2 (DRAFT · your answers folded in)

> **Status:** v0.2 DRAFT — updated with Landon's answers to the v0 open questions and the **workbook-
> driving reframe** (the app is a permanent input bridge into the MatchChart `.xlsm`). Landon may
> still write a fresh authoritative version; if so, re-read it as gospel and reconcile.

Scope (PRD v0.2): single operator · post-hoc · **video in a separate app** · **the `.xlsm` is the
system of record** · pluggable input (Stream Deck primary, keyboard next) · validate-as-you-type ·
next/previous-point navigation · autosave.

---

## Actors & preconditions

- **Operator** — one person, novice or expert.
- **Input source** — a **Stream Deck XL** (default; Elgato app quit so our app owns it) *or* the
  **keyboard**. The app detects what's available and picks by priority (deck → keyboard → other).
- **The workbook** — a MatchChart `.xlsm` the operator will chart into; it stays the system of record.
- **Video** — open in the operator's own player (VLC/YouTube/…), driven manually and separately.

## Session states

`IDLE → PICK_WORKBOOK → CHARTING → (PAUSED ⇄ CHARTING) → operator marks MATCH_DONE`

Within **CHARTING**, each point walks a rally sub-state:
`AWAIT_SERVE → (SERVE_FAULT → AWAIT_2ND)? → AWAIT_RETURN → RALLY → POINT_END → (next point)`

There is **no explicit commit** — advancing to the next point *is* the commit, and it autosaves.

---

## The flow, step by step

### 0. One-time setup
1. Install the app. Preferred: plug in the Stream Deck XL and quit the Elgato Stream Deck app. If no
   deck is present, the app falls back to **keyboard** input automatically.
2. Launch the app → it reports the active input source (deck or keyboard).

### 1. Metadata (in the workbook, outside our app)
3. The operator opens their MatchChart `.xlsm` and fills in match metadata there — players
   (server-first), hands, tournament/round/date/surface, final-set rule. **Our app does not do
   metadata**; the workbook does.
4. Queue up the match video in the separate player.

### 2. Point the app at the workbook
5. In the app, the operator selects the `.xlsm` to drive (and confirms the first point cell).
6. App enters **CHARTING**, positioned on the first point's `1st` cell.

### 3. The charting loop (repeats per point) — the core
For each point:
7. Operator watches the point in their video player (pausing/rewinding there as needed).
8. **Serve:** press serve direction (`4/5/6`, or `0`). The deck/keyboard shows serve-context tokens
   (fault types, ace `*`, serve-and-volley `+`, let `c`).
   - **Fault** → enter fault type → context shifts to the second serve (`2nd`); repeat.
   - **Point ends on serve** (ace/unreturnable) → jump to point end.
9. **Return + rally:** context shifts to shot entry — shot-type keys (hand-side modifier, direction
   `1/2/3`, return depth `7/8/9`). Each press **appends a token to the point's code string**, shown as
   it builds.
10. Optional per-shot detail (direction, depth, court-position modifiers) is surfaced as optional keys
    — novices skip, experts add.
11. **Point end:** enter the ending — winner `*`, error (`n/w/d/x/!/e`) + `#`/`@`, or a whole-point
    special (`S/R/P/Q`).
12. **Validate + write:** the assembled string is validated against the grammar; if well-formed it's
    **written into the workbook cell**. If malformed, the app flags it and won't advance.
13. **Next point:** the operator presses **Next point** → the app writes/confirms the cell,
    **autosaves the workbook**, and moves to the next row's `1st` cell. The workbook's own macros
    derive the score and populate the next row. (No separate commit; Next *is* the commit.)

### 4. Correct a mistake (P0)
14. **Undo last shot** — drop the last token from the in-progress point before moving on.
15. **Previous point** — press **Previous point** to navigate back to an earlier point's **cell**; the
    app shows that cell, and the operator simply **re-inputs the syntax** for that point, overwriting
    it. Then **Next point** forward again.

### 5. Pause / resume
16. **Autosave happens on every next-point**, so the workbook is always current. The operator can stop
    anytime and close the app; the workbook holds all progress.
17. **Resume:** reopen the app, re-select the `.xlsm`, and navigate to the next un-charted point.

### 6. Match end
18. When done, the **operator marks the match complete** (the app does not infer it from the score).

### 7. Submission (unchanged, outside our app)
19. The operator submits the finished `.xlsm` to the MCP coordinator exactly as today — the workbook is
    already the accepted format. Nothing to export or convert.

---

## Branch points & edge cases
- **No Stream Deck** → automatic keyboard fallback (P0); "other" sources later.
- **Missed a point** on video → whole-point `S`/`R` with a note in the workbook's Notes column.
- **Unknown detail** mid-rally → `0`/`q`/`e` escapes; keep moving.
- **Let / time violation / point penalty / incorrect challenge** → their special codes mid-loop.
- **Deck disconnects mid-match** → fall back to keyboard, or pause; the workbook is already saved.
- **Derived score looks wrong** → signal of an earlier mis-entry → Previous point → re-input.

## Resolved from v0 (Landon's answers)
1. **Deck not found** → pluggable input; auto-fallback to keyboard (deck → keyboard → other).
2. **Metadata** → entered in the `.xlsm`, not our app.
3. **Commit** → none; **Next / Previous point** navigation instead.
4. **Correction** → Previous point navigates to that cell; operator re-inputs the point's syntax.
5. **Save** → autosave on every next-point.
6. **Match end** → operator marks it; not auto-detected.

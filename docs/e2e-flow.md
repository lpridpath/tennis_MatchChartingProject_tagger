# E2E User Flow — v0 (DRAFT · to be superseded)

> **Status:** v0 DRAFT — Claude's *inference* from PRD v0.1 + the reference docs. **Not gospel.**
> Landon intends to write a fresh authoritative version; when that lands, re-read it as the source of
> truth and reconcile the PRD, map, and tickets to it.
> **🟡** marks a genuine open question the gospel version should settle.

Scope reminder (PRD v0.1): single operator · post-hoc · **video plays in a separate app** (no in-app
video, no timestamps in v1) · Stream Deck XL drives input · validate-as-you-go · export to MCP CSVs.

---

## Actors & preconditions

- **Operator** — one person, novice *or* expert, charting one match.
- **Stream Deck XL** — connected; the Elgato app is quit so our app owns the device.
- **Video** — the operator has the match open in their own player (VLC/YouTube/etc.), controlled
  **manually and separately**. Our app never touches it in v1.

## Session states (the state machine this flow implies)

`IDLE → MATCH_SETUP → CHARTING → (PAUSED ⇄ CHARTING) → MATCH_COMPLETE → EXPORTED`

Within **CHARTING**, each point walks a rally sub-state:
`AWAIT_SERVE → (SERVE_FAULT → AWAIT_2ND)? → AWAIT_RETURN → RALLY → POINT_END → commit`

---

## The flow, step by step

### 0. One-time setup
1. Install the app; plug in the Stream Deck XL; quit the Elgato Stream Deck app.
2. Launch our app — it detects the deck and lights the keys.
   🟡 What if the deck isn't found / Elgato app is still running? (Show a "connect your deck" state.)

### 1. Start or resume a match
3. App opens to a **home/IDLE** screen: *New match* or *Resume in-progress match*.
4. **New match** → go to Match setup. **Resume** → load saved session, jump back into CHARTING at the
   next un-charted point.

### 2. Match setup (metadata)
5. Operator enters match metadata (per the `.xlsm` header): both players (server-first), dominant
   hands, tournament, round, date, surface, and the final-set/tiebreak rule.
   🟡 Entered how — on-screen form (keyboard/mouse), or partly on the deck? Likely on-screen; the deck
   is for the charting loop, not form-filling.
6. Operator queues up the video in their separate player, ready at the first point.
7. Confirm → app enters **CHARTING** at game 1, point 1. Score display shows `0–0`.

### 3. The charting loop (repeats per point) — the core
For each point:
8. Operator watches the point play out in their video player (pausing/rewinding there as needed).
9. **Serve:** press the serve direction key (`4/5/6`, or `0` unknown). The deck now shows serve-context
   keys (fault types, ace `*`, serve-and-volley `+`, let `c`).
   - If **fault**, enter fault type → deck shifts to **second-serve** context; repeat.
   - If **ace / unreturnable / point ends on serve**, mark it → **POINT_END**.
10. **Return + rally:** deck shifts to shot-entry context — shot-type keys (context-aware: hand-side
    modifier, direction `1/2/3`, return depth `7/8/9`). Each shot appends to the running code string,
    shown on screen as it builds.
11. Optional detail per shot (direction, depth, court-position modifiers) — surfaced as optional
    modifier keys so novices can skip and experts can add (PRD tiers).
12. **Point end:** press the ending — winner `*`, or error (`n/w/d/x/!/e`) + forced `#` / unforced `@`;
    or a whole-point special (`S/R/P/Q`).
13. **Commit:** app validates the assembled string against the grammar. If **valid**, it commits the
    point, **derives the new score**, and advances to the next point (AWAIT_SERVE). If **malformed**,
    it blocks the commit and flags what's wrong (validate-as-you-go).
    🟡 Is commit an explicit key press, or automatic when the point is grammatically complete?
14. The running score updates. If the derived score looks wrong to the operator, that's the signal an
    earlier point was mis-entered → they undo/correct (below).

### 4. Correct a mistake (available throughout — P0)
15. **Undo last shot** — remove the last token from the in-progress point.
16. **Undo/redo last point** — step back to a committed point, re-open it, re-enter.
    🟡 How far back can you correct — only the last point, or jump to any earlier point? Suggest: last
    shot instantly; any prior point via a point list/scrubber.

### 5. Pause / save / resume
17. Operator can **pause** anytime → session state (all committed points + the in-progress one) is
    saved to disk. App → PAUSED (or can be closed entirely).
    🟡 Auto-save every commit, or explicit save? Suggest auto-save each committed point (PRD NFR:
    lose ≤ the current point on crash).
18. Later, **Resume** (step 4) drops them back at the next un-charted point.

### 6. Match ends
19. When the final point is charted, the scorer recognizes the match is won → app → MATCH_COMPLETE.
    🟡 Does the app *know* the match is over from the score + format rules (recommended), or does the
    operator mark "match done"?
20. App shows a summary (final score, points charted, any points flagged/guessed).

### 7. Export
21. Operator triggers **Export** → app writes the MCP CSVs (`-matches`, `-points`, `-stats`) from the
    captured model, via the validated exporter. (#6)
22. App confirms the export location and that every point is well-formed / submittable.

### 8. Submit (outside the app)
23. Operator sends the output onward for inclusion in the MCP dataset (the Jeff-Sackmann handoff).
    Out of app scope in v1 (no automated submission).

---

## Branch points & edge cases (v0 guesses)
- **Missed a point** on video → whole-point `S`/`R` (award server/returner) with a note.
- **Unknown detail** mid-rally → `0`/`q`/`e` escapes; keep moving.
- **Let, time violation, point penalty, incorrect challenge** → their special codes mid-loop.
- **Deck disconnects mid-match** → pause + reconnect prompt; don't lose state.

## Open questions collected (🟡)
1. Deck-not-found / Elgato-app-running handling (step 2).
2. Metadata entry surface — on-screen vs deck (step 5).
3. Commit = explicit key vs auto-on-complete (step 13).
4. Correction reach — last point only vs any prior point (step 16).
5. Save model — auto per commit vs explicit (step 17).
6. Match-end detection — derived vs operator-marked (step 19).

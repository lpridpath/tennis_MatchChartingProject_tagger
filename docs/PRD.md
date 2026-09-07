# PRD — MCP Tagger (working title)

> **Status:** v0 DRAFT — for review/red-line · **Owner:** Landon · **Date:** 2026-09-06
> **🟡 ASSUMPTION** markers flag values Claude filled in; confirm or overwrite each.
> This PRD is the product-level driver. It sits *above* the wayfinder map
> ([#1](https://github.com/lpridpath/tennis_MatchChartingProject_tagger/issues/1)); the map's
> decision tickets resolve the *how*. Domain facts live in `reference/` — linked, not restated.

---

## 1. Problem & background

The [Match Charting Project](https://github.com/JeffSackmann/tennis_MatchChartingProject) (MCP) is a
crowdsourced dataset of shot-by-shot tennis data. Contributors chart matches **by hand** into a
macro Excel workbook (`MatchChart 0.3.2.xlsm`), typing a terse alphanumeric code per point while
scrubbing video in a *separate* player app. The flow is documented in
`reference/mcp-charting-workflow.md`; the notation in `reference/mcp-shorthand-grammar.md`.

That flow is slow and high-friction: rapid code typing, constant pause/rewind, tool-switching
between the video app and Excel, and a steep notation-learning curve. **The efficiency ceiling of
the manual flow is the core problem this project attacks.**

## 2. The thesis (product overview)

A desktop **companion/sidecar** — *not* a fork of MCP — that makes charting dramatically faster by:

1. Replacing **keyboard code-typing** with an **Elgato Stream Deck XL** (32 labeled, context-aware
   keys — one key per notation token instead of memorized codes).
2. Replacing the **video-app + Excel split** with **one surface**: video playback and tagging in the
   same app, playback controllable from the deck.
3. **Validating as you go**, so malformed points are caught immediately instead of surfacing later
   as a wrong auto-score.

Output stays **100% MCP-compatible** — the tool produces submittable data, it does not invent a
competing format.

## 3. Users / personas

- **Primary: the expert operator (you).** A single, experienced charter who knows the MCP notation
  and wants to chart faster and more comfortably. Owns a Stream Deck XL.
  **🟡 ASSUMPTION:** v1 targets *one* proficient local user, not novices and not a crowd.
- **Designed-around (not v1):** novice charters (progressive-tier onboarding), multi-user/crowd
  contribution. The data model must not preclude these; the UX won't build for them yet.

## 4. Goals & success metrics

**Primary goal:** cut the time and cognitive load to chart a match at MCP's *intermediate* tier
(serve dir + shot type + shot direction), with **zero loss of output fidelity**.

**🟡 ASSUMPTION — success metrics (please set real targets):**
- **Speed:** chart a match in **≤ 1.25× real match length** at intermediate tier (baseline: the
  Excel flow is commonly ~2–3× match length for newer charters). *Set your own target.*
- **Accuracy:** **0 malformed points** reach export (blocked by live validation); output parses
  cleanly into the MCP `-points` format.
- **Ergonomics/adoption:** you *prefer* this over the Excel flow for your next real chart (the
  honest single-user adoption test).

**Non-metric bar:** every match charted in the tool must round-trip to a valid MCP submission.

## 5. Non-goals (v1)

Ruled out for this version (from the map's out-of-scope list):
- Machine-generated tag suggestions / auto-charting (human does the charting).
- Multi-user / crowd charting or collaboration.
- **Live-mode implementation** (v1 is post-hoc video review; live is designed-around, not built).
- Streaming-video ingest (v1 = **local files only**).
- Custom notation tokens beyond the MCP vocabulary.
- Automated match selection, duplicate-avoidance, or email submission to the coordinator.

## 6. Scope & requirements

Priorities: **P0** = MVP (must ship in v1) · **P1** = v1 if time · **P2** = later/fog.

### 6.1 Functional — charting core
- **P0** One key per MCP token via the Stream Deck XL, with on-key labels.
- **P0** Context-sensitive key layouts that follow rally state (serve → return → rally → ending) and
  surface valid next tokens.
- **P0** Support charting at **any complexity tier** (beginner→expert); direction/depth optional, not
  forced.
- **P0** Reach "unknown" escapes (`0`,`q`,`e`,`R`/`S`) and point-enders quickly.
- **P0** Live validation: reject/flag a malformed point before it's committed.
- **P1** Undo / correct the last shot or point. **🟡 ASSUMPTION:** P1, not P0 — confirm; correction
  may be too core to defer.
- **P2** Keyboard fallback for coding (deck optional).

### 6.2 Functional — video
- **P0** Load and play a **local** video file; play/pause; precise seek; **single-frame step**.
- **P0** Playback controllable from the Stream Deck.
- **P0** Bind each charted point to a **video timestamp**.

### 6.3 Functional — data & export
- **P0** Capture into an internal model that is a **superset** of MCP (adds timestamps, raw
  keystrokes) and exports **losslessly** to canonical MCP code strings. *(Ticket
  [#3](https://github.com/lpridpath/tennis_MatchChartingProject_tagger/issues/3))*
- **P0** Export to MCP CSVs (`-matches`, `-points`, `-stats`). *(Ticket
  [#6](https://github.com/lpridpath/tennis_MatchChartingProject_tagger/issues/6))*
- **P0** Derive score / set / game boundaries (not stored) — makes mis-entry detectable.
- **P0** Enter match metadata (players server-first, hands, tournament/round/date/surface, final-set
  rule).
- **P1** Save / resume an in-progress chart.

### 6.4 Non-functional
- **P0** Per-key press → app response and video frame-step at interactive latency (no perceptible lag
  during a rally).
- **P0** macOS desktop (Apple Silicon). **🟡 ASSUMPTION:** Mac-first; Windows/Linux later. Electron
  keeps cross-platform open.
- **P0** Runs fully **offline** (local files, local device).
- **P1** Losing power / crashing mid-chart doesn't lose more than the current point.

## 7. Solution constraints (locked decisions)

From the wayfinder destination round — see map [#1](https://github.com/lpridpath/tennis_MatchChartingProject_tagger/issues/1):
- **Stack:** Electron + TypeScript. *(feasibility GREEN — ticket #2)*
- **Device:** direct Stream Deck XL control (`@elgato-stream-deck/node`, main process + IPC), Elgato
  app closed.
- **Video:** local files; Electron ships proprietary codecs so H.264/mp4 + VP9/webm play natively.
- **Output:** strict, validated MCP shorthand (layer 1, locked); superset capture model (layer 2);
  fully customizable input mapping (layer 3).

## 8. Open decisions (in flight)

Tracked as wayfinder tickets, blocking the final spec ([#7](https://github.com/lpridpath/tennis_MatchChartingProject_tagger/issues/7)):
- **#3** Capture data model + MCP exporter *(claimed, paused for this PRD)*
- **#4** Stream Deck key-mapping & paging model
- **#5** Video-playback ↔ tagging interaction
- **#6** Match/session lifecycle & CSV export

## 9. Assumptions to validate (all 🟡 above, collected)

1. Single expert operator (you), not novices/crowd — §3.
2. Success-metric targets (speed ratio, accuracy bar, adoption test) — §4.
3. Undo/correct is P1 not P0 — §6.1.
4. Mac-first, other OSes later — §6.4.

## 10. Milestones (indicative, non-binding)

1. **Spec complete** — all decision tickets resolved, #7 assembled. *(this planning effort)*
2. Charting core prototype (deck + validation, no video).
3. Video surface + timestamp binding.
4. Export to MCP CSVs + round-trip validated against real dataset.
5. First real match charted end-to-end in the tool (the adoption test).

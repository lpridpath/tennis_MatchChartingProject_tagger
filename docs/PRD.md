# PRD — MCP Tagger (working title)

> **Status:** v0.1 DRAFT — for review/red-line · **Owner:** Landon · **Date:** 2026-09-06
> **🟡 ASSUMPTION** markers flag values still needing your confirmation.
> This PRD is the product-level driver. It sits *above* the wayfinder map
> ([#1](https://github.com/lpridpath/tennis_MatchChartingProject_tagger/issues/1)); the map's
> decision tickets resolve the *how*. Domain facts live in `reference/` — linked, not restated.
>
> **v0.1 changes:** video controls pulled out of MVP (now a future enhancement); Jeff Sackmann added
> as a downstream persona; personas reframed to serve *both* novices and experts; success metrics
> set; undo/correct promoted to P0; cross-platform made a hard requirement.

---

## 1. Problem & background

The [Match Charting Project](https://github.com/JeffSackmann/tennis_MatchChartingProject) (MCP) is a
crowdsourced dataset of shot-by-shot tennis data. Contributors chart matches **by hand** into a
macro Excel workbook (`MatchChart 0.3.2.xlsm`), typing a terse alphanumeric code per point while
scrubbing video in a *separate* player app. The flow is documented in
`reference/mcp-charting-workflow.md`; the notation in `reference/mcp-shorthand-grammar.md`.

That flow is slow and high-friction: rapid code typing and a steep notation-learning curve make it
both intimidating for newcomers and tedious for veterans. **The core problem: charting is hard to
start and slow to do.**

## 2. The thesis (product overview)

A desktop **companion/sidecar** — *not* a fork of MCP — built around an **Elgato Stream Deck XL**
(32 labeled, context-aware keys). Two MVP pillars:

1. **Replace keyboard code-typing with the deck** — one key per notation token instead of memorized
   codes; context-aware layouts show what's valid next.
2. **Validate as you go** — malformed points are caught immediately, not surfaced later as a wrong
   auto-score.

Output stays **100% MCP-compatible** — submittable data, not a competing format.

**Future vision (post-MVP):** fold video *into* the app — playback controllable from the deck, each
point bound to a video timestamp — collapsing today's video-app + Excel split into one surface. See
§6.2; deliberately **out of the MVP**.

## 3. Users / personas

The tool is **for everyone who charts** — it should do *different things* for two tiers:

- **Novice charters** — value = **approachability**. Lower the barrier to entry: guided,
  context-sensitive layouts that surface valid next tokens so you can chart without memorizing the
  notation or constantly checking instructions.
- **Expert charters** — value = **efficiency + power tools**. Fast paths, modifiers, minimal
  prompting; get out of the way and let a proficient charter fly.

- **Downstream consumer — Jeff Sackmann (MCP coordinator).** The charted output is ultimately sent to
  him for inclusion in the dataset. He is a persona because **his acceptance defines "valid output"**:
  the tool must emit well-formed, submittable MCP data with no manual clean-up on his end.

- **Initial operator / dogfooder — Landon.** First real user and test-user recruiter; develops and
  validates on macOS.

**Single-operator, not multi-user:** one person charts a given match at a time (no real-time
collaboration). The *user base* is broad; concurrent collaborative charting is a non-goal (§5).

## 4. Goals & success metrics

**Primary goal:** make charting **approachable for novices** and **fast for experts**, at MCP's
intermediate tier (serve dir + shot type + shot direction), with **zero loss of output fidelity**.

**Success metrics:**
- **Speed:** chart a match in **~2–3× real match length** at intermediate tier (comparable wall-clock
  to a proficient Excel charter — the win is far lower *cognitive load* and learning curve, plus
  expert power-tools, not just raw time).
- **Accuracy:** **0 malformed points** reach export (blocked by live validation); output parses
  cleanly into MCP `-points`.
- **Ergonomics / adoption:** the tool is **preferred over the Excel flow**, and reaches **some real
  adoption** — i.e., Landon can recruit test users who chart with it.

**Non-metric bar:** every match charted must round-trip to a valid MCP submission.

## 5. Non-goals (v1)

Ruled out for this version:
- **Integrated video playback / control and per-point timestamp binding** — v1 assumes video runs in
  a *separate* app; deferred to the post-MVP "one surface" enhancement (§2, §6.2).
- Machine-generated tag suggestions / auto-charting.
- **Real-time multi-user / crowd** charting or collaboration.
- **Live-mode implementation** (v1 is post-hoc; live designed-around, not built).
- Custom notation tokens beyond the MCP vocabulary.
- Automated match selection, duplicate-avoidance, or email submission.

## 6. Scope & requirements

Priorities: **P0** = MVP (must ship in v1) · **P1** = v1 if time · **P2** = later/fog.

### 6.1 Functional — charting core
- **P0** One key per MCP token via the Stream Deck XL, with on-key labels.
- **P0** Context-sensitive key layouts that follow rally state (serve → return → rally → ending) and
  surface valid next tokens — the primary lever for **novice approachability**.
- **P0** Support charting at **any complexity tier** (beginner→expert); direction/depth optional.
- **P0** Reach "unknown" escapes (`0`,`q`,`e`,`R`/`S`) and point-enders quickly.
- **P0** Live validation: reject/flag a malformed point before it's committed.
- **P0** **Undo / correct the last shot or point** — too central to defer; corrections are constant
  in real charting.
- **P1** Expert power-tools: modifier keys, shortcuts, layout customization.
- **P2** Keyboard fallback for coding (deck optional).

### 6.2 Video — DEFERRED (post-MVP)
- **P2 / future** Load, play, precisely seek, and **frame-step** local video *inside* the app.
- **P2 / future** Control playback from the Stream Deck.
- **P2 / future** Bind each charted point to a **video timestamp**.

> **MVP assumption:** the operator watches the match in whatever player they like (VLC, YouTube,
> etc.), *separately*. The tagger does **not** control video or capture timestamps in v1. This keeps
> the MVP small and gets end-to-end validation faster; integrated video is the first major follow-on.

### 6.3 Functional — data & export
- **P0** Capture into an internal model that is a **superset** of MCP (adds raw keystrokes; timestamps
  when video lands) and exports **losslessly** to canonical MCP code strings. *(Ticket
  [#3](https://github.com/lpridpath/tennis_MatchChartingProject_tagger/issues/3))*
- **P0** Export to MCP CSVs (`-matches`, `-points`, `-stats`). *(Ticket
  [#6](https://github.com/lpridpath/tennis_MatchChartingProject_tagger/issues/6))*
- **P0** Derive score / set / game boundaries (not stored) — makes mis-entry detectable.
- **P0** Enter match metadata (players server-first, hands, tournament/round/date/surface, final-set
  rule).
- **P1** Save / resume an in-progress chart.

### 6.4 Non-functional
- **P0** Per-key press → app response at interactive latency (no perceptible lag during a rally).
- **P0** **Cross-platform is a hard requirement.** Windows, macOS, and Linux must all be supported;
  **Windows is expected to be the majority of users.** Mac-first is only the *development* sequence
  (Landon's machine), not a scope limit.
  **🟡 ASSUMPTION / decision to confirm:** Electron gives cross-platform (incl. Windows) from one
  codebase, but is *not* OS-native UI. If "native application on each OS" is a hard requirement
  (rather than "runs well on each OS"), that **reopens the stack decision** (#2/#7). Claude's read:
  Electron satisfies "must support other OSes" and keeps one codebase — recommend treating cross-
  platform-from-one-codebase as the requirement, not per-OS native widgets.
- **P0** Runs fully **offline** (local device; local files).
- **P1** Crash / power loss mid-chart doesn't lose more than the current point.

## 7. Solution constraints (locked decisions)

From the wayfinder destination round — see map [#1](https://github.com/lpridpath/tennis_MatchChartingProject_tagger/issues/1):
- **Stack:** Electron + TypeScript. *(feasibility GREEN — ticket #2; also the cross-platform vehicle,
  §6.4.)*
- **Device:** direct Stream Deck XL control (`@elgato-stream-deck/node`, main process + IPC), Elgato
  app closed.
- **Output:** strict, validated MCP shorthand (layer 1, locked); superset capture model (layer 2);
  fully customizable input mapping (layer 3).
- **Video:** *(deferred)* — when built, Electron ships proprietary codecs so H.264/mp4 + VP9/webm play
  natively (ticket #2 findings), so the future enhancement has no codec blocker.

## 8. Open decisions (in flight)

Tracked as wayfinder tickets, blocking the final spec ([#7](https://github.com/lpridpath/tennis_MatchChartingProject_tagger/issues/7)):
- **#3** Capture data model + MCP exporter *(claimed, paused for this PRD)*
- **#4** Stream Deck key-mapping & paging model
- **#6** Match/session lifecycle & CSV export
- **#5** Video-playback ↔ tagging interaction — **moved to post-MVP** per §6.2; removed from the v1
  destination.

## 9. Assumptions — status

1. ✅ **Single operator** per session (broad user base, no real-time collaboration) — §3.
2. ✅ **Speed target ~2–3× match length**; primary win is cognitive load + expert power — §4.
3. ✅ **Adoption metric**: preferred over Excel + recruits real test users — §4.
4. 🟡 **Cross-platform via Electron (one codebase) vs. per-OS native** — confirm Electron satisfies
   "must support other OSes" (§6.4). Only open assumption.

## 10. Milestones (indicative, non-binding)

1. **Spec complete** — decision tickets resolved, #7 assembled. *(this planning effort)*
2. **Charting-core MVP** — deck input + context layouts + live validation + undo, **no video**.
3. **Export** — MCP CSVs + round-trip validated against the real MCP dataset.
4. **First real match charted end-to-end** (video watched separately) — the E2E validation + adoption
   test.
5. *(Post-MVP)* Integrated video + timestamp binding — the "one surface" enhancement.

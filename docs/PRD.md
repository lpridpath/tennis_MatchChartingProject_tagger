# PRD — MCP Tagger (working title)

> **Status:** v0.2 — for review/red-line · **Owner:** Landon · **Date:** 2026-09-06
> This PRD is the product-level driver. It sits *above* the wayfinder map
> ([#1](https://github.com/lpridpath/tennis_MatchChartingProject_tagger/issues/1)); the map's
> decision tickets resolve the *how*. Domain facts live in `reference/`; the user journey in
> `docs/e2e-flow.md`.
>
> **v0.2 — major reframe:** the tool is a permanent **input bridge that drives the existing
> MatchChart `.xlsm`**, not a standalone app that replaces it. The workbook stays the system of
> record (score, stats, CSV, metadata, submission); we replace the *code-typing* with a Stream Deck
> (and other input sources). Consequences: no in-app capture model, scorer, or CSV exporter — those
> are the workbook's job. Input is pluggable (deck → keyboard → other). See §2, §5, §6.

---

## 1. Problem & background

The [Match Charting Project](https://github.com/JeffSackmann/tennis_MatchChartingProject) (MCP) is a
crowdsourced dataset of shot-by-shot tennis data. Contributors chart matches **by hand** into a macro
Excel workbook (`MatchChart 0.3.2.xlsm`), typing a terse alphanumeric code per point while scrubbing
video in a *separate* player. The flow is in `reference/mcp-charting-workflow.md`; the notation in
`reference/mcp-shorthand-grammar.md`.

The workbook is proven and is what the parent project accepts — it auto-derives score, generates the
stats and CSVs, and defines the submission format. The pain is **not** the workbook; it's the
**data entry**: typing terse codes fast, and a steep notation-learning curve that's intimidating for
newcomers and tedious for veterans. **That input step is the whole problem this project attacks.**

## 2. The thesis (product overview)

A desktop **input bridge** that sits in front of the real MatchChart `.xlsm` and makes charting fast
and approachable — **without replacing the workbook**. Core pillars:

1. **Replace code-typing with a Stream Deck XL** — one key per notation token, on-key labels,
   context-aware layouts that show what's valid next. Input is **pluggable**: Stream Deck is the
   default/primary; keyboard is the built-in alternative; other sources come later.
2. **Drive the workbook directly** — assembled MCP code strings are written into the workbook's point
   cells (`1st`/`2nd`); *next / previous point* navigates its rows; changes autosave.
3. **Validate as you type** — a point's code string is checked for well-formedness before it lands in
   the cell, so the workbook's own "wrong auto-score" signal fires far less often.

The workbook remains the **system of record**: it does score derivation, stats, CSV generation,
metadata, and submission. We are bound to it because the parent MCP project is bound to it.

## 3. Users / personas

The tool is **for everyone who charts** — doing *different things* for two tiers:

- **Novice charters** — value = **approachability**: guided, context-sensitive layouts surface valid
  next tokens so you can chart without memorizing the notation.
- **Expert charters** — value = **efficiency + power tools**: fast paths, modifiers, minimal prompting.
- **Downstream consumer — Jeff Sackmann (MCP coordinator).** Output is his `.xlsm`, submitted as
  today. His format *is* our constraint; because we write into his workbook, compatibility is
  structural, not just a hoped-for export. **No clean-up on his end.**
- **Initial operator / dogfooder — Landon.** First user + test-user recruiter; develops on macOS.

**Single-operator, not multi-user:** one person charts one match at a time. Broad user base;
real-time collaboration is a non-goal (§5).

## 4. Goals & success metrics

**Primary goal:** make charting **approachable for novices** and **fast for experts** at MCP's
intermediate tier, with **zero loss of fidelity** — because we write straight into the accepted
workbook.

**Success metrics:**
- **Speed:** chart a match in **~2–3× real match length** at intermediate tier — the win is lower
  cognitive load + expert power-tools, not just wall-clock.
- **Accuracy:** **0 malformed code strings** written to the workbook (blocked by live validation).
- **Ergonomics / adoption:** **preferred over hand-typing into the workbook**, and reaches **some real
  adoption** (Landon can recruit test users).

**Non-metric bar:** a workbook charted via the tool is byte-for-byte a valid MCP submission — because
it *is* a MatchChart workbook.

## 5. Non-goals (v1)

- **A standalone data model / our own CSV export / our own scorer** — delegated entirely to the
  workbook. We never re-implement what the `.xlsm` already does.
- **Integrated video playback / control / timestamps** — video runs in a separate app; deferred to a
  post-MVP enhancement (§6.5).
- Machine-generated tag suggestions / auto-charting.
- **Real-time multi-user / crowd** charting.
- **Live-mode implementation** (v1 is post-hoc; designed-around, not built).
- Custom notation tokens beyond the MCP vocabulary.
- Modifying the workbook's format, macros, or metadata layout.

## 6. Scope & requirements

Priorities: **P0** = MVP · **P1** = v1 if time · **P2** = later/fog.

### 6.1 Input & mapping
- **P0** **Pluggable input sources**, priority-ordered: **Stream Deck XL** (default/primary) →
  **keyboard** (built-in alternative) → other (later). An input-source abstraction, not a hard-wired
  device. *(Ticket [#4](https://github.com/lpridpath/tennis_MatchChartingProject_tagger/issues/4))*
- **P0** One key/keystroke per MCP token, on-key labels (deck).
- **P0** Context-sensitive layouts that follow rally state (serve → return → rally → ending) and
  surface valid next tokens — the main lever for **novice approachability**.
- **P0** Support charting at **any tier** (beginner→expert); direction/depth optional.
- **P0** Fast reach to "unknown" escapes (`0`,`q`,`e`,`R`/`S`) and point-enders.
- **P0** Undo last shot; re-edit a prior point (navigate to its cell and re-enter its syntax — §6.2).

### 6.2 Workbook integration (the core)
- **P0** Target/open a MatchChart `.xlsm`; write assembled MCP code strings into the point cells
  (`1st`/`2nd`). *(Ticket [#3](https://github.com/lpridpath/tennis_MatchChartingProject_tagger/issues/3) — re-scoped to the code-string builder + validator + cell write.)*
- **P0** **Next / previous point** navigation — move to the corresponding workbook row/cell (there is
  no separate "commit"; moving to the next point is the commit).
- **P0** **Autosave** on every next-point.
- **P0** **Operator marks match end** (not auto-detected).
- **P0** Preserve the workbook's own behavior — the macro-driven score/next-row/stats must still fire;
  don't corrupt the file.
- **CRITICAL / OPEN:** *how* the app writes into a macro `.xlsm` while its macros still run, on
  Windows (majority) and macOS — Excel automation vs. driving a focused Excel vs. file-level writes.
  This is the linchpin technical decision. *(New ticket — see §8.)*

### 6.3 Validation
- **P0** Assemble and **validate** a well-formed MCP string (per `reference/mcp-shorthand-grammar.md`)
  before it's written to a cell — validate-as-you-type.

### 6.4 Delegated to the workbook (explicitly NOT built)
Score/game derivation · stats · CSV export · match metadata entry (done in the `.xlsm`, §Q2) ·
submission. The tool must not duplicate these.

### 6.5 Video — DEFERRED (post-MVP)
- **P2 / future** In-app local video with frame-step, deck-controlled playback, per-point timestamps.
  v1 assumes the operator watches in a separate player.

### 6.6 Non-functional
- **P0** Input press → response at interactive latency (no lag during a rally).
- **P0** **Cross-platform from one Electron codebase** (Windows/Mac/Linux); **Windows is the expected
  majority**. Mac-first only for development. *(The workbook-write mechanism, §6.2, is the main
  cross-platform risk.)*
- **P0** Runs **offline** (local device, local workbook).
- **P0** Autosave each next-point → crash/power-loss loses ≤ the current point.

## 7. Solution constraints (locked decisions)

- **Stack:** Electron + TypeScript. *(feasibility GREEN — ticket #2; cross-platform vehicle.)*
- **Primary input:** direct Stream Deck XL control (`@elgato-stream-deck/node`, main process + IPC),
  Elgato app closed; behind a **pluggable input-source abstraction** (keyboard next).
- **System of record:** the MatchChart `.xlsm` — we drive it, we don't replace it.
- **Output vocabulary:** strict MCP shorthand (layer 1, locked) — structural, since we write into the
  workbook. Input mapping (layer 3) is customizable. *(The former layer-2 "superset capture model" is
  dropped — the workbook is the model.)*

## 8. Open decisions (in flight)

Tracked as wayfinder tickets, blocking the final spec ([#7](https://github.com/lpridpath/tennis_MatchChartingProject_tagger/issues/7)):
- **NEW · `.xlsm` write mechanism** — the linchpin: how to write cells while macros run, cross-platform.
- **#3** *(re-scoped)* In-app MCP code-string builder + validator + cell write.
- **#4** *(re-scoped)* Input mapping across sources (Stream Deck + keyboard).
- **#6** *(re-scoped)* Session & workbook targeting: open/select `.xlsm`, row navigation, autosave,
  operator-marked match end.

## 9. Assumptions — resolved

1. ✅ Single operator per session; broad user base; no real-time collaboration.
2. ✅ Speed target ~2–3× match length; primary win = cognitive load + expert power.
3. ✅ Adoption = preferred over hand-typing + recruits real test users.
4. ✅ Cross-platform via Electron (one codebase, Win/Mac/Linux).
5. ✅ Input is pluggable: Stream Deck → keyboard → other.
6. ✅ Workbook is the permanent system of record (metadata, score, stats, CSV, submission).

## 10. Roadmap (indicative)

1. **Spec complete** — decision tickets resolved, #7 assembled. *(this planning effort)*
2. **Write-mechanism spike** — prove we can drive the `.xlsm` on the target OS(es).
3. **Charting-core MVP** — deck + keyboard input, context layouts, live validation, undo, writing into
   the workbook with next/prev + autosave.
4. **First real match charted end-to-end** into a submittable workbook (video watched separately) — the
   E2E + adoption test.
5. *(Post-MVP)* Visualization tools + quality-of-life improvements; later, integrated video.

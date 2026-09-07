# Diagrams — source of truth

Two views that center the project and index the handoffs. Version-controlled here; mirrored on the
tldraw canvas (`MCP | Match Charting Project.tldraw` → **Diagram** page) for whiteboarding. Keep this
file authoritative; update it when components or tickets change.

Legend: **✅ locked/done** · **▶ in flight** · **⛭ frontier (takeable)** · **⏸ blocked** · **↝ deferred/post-MVP**.

---

## View 1 — System components (product data flow)

What the tool *is* and how data flows from the operator to the MCP dataset. Each box names the
ticket/doc that owns it.

```mermaid
flowchart LR
  SD["🎛 Stream Deck XL<br/>32 context-aware keys"]

  subgraph APP["Electron + TypeScript app  (✅ stack, #2)"]
    direction TB
    IM["Input mapping<br/>layer 3 · ⛭ #4"]
    CE["Capture engine<br/>Point/Shot model · ▶ #3"]
    VAL["Live validator<br/>▶ #3"]
    SC["Scorer<br/>derives score/games · ▶ #3"]
    EX["Exporter<br/>→ MCP shorthand · ▶ #3"]
    IM --> CE
    CE --> VAL
    CE --> SC
    CE --> EX
  end

  CSV["MCP CSVs<br/>-matches · -points · -stats<br/>⏸ #6"]
  JS["👤 Jeff Sackmann<br/>MCP dataset (accepts output)"]
  VID["📺 External video player<br/>VLC / YouTube — separate in v1"]

  SD -->|key presses| IM
  EX --> CSV
  CSV --> JS
  VID -.->|watched alongside; integration ↝ #5 post-MVP| APP

  GRAMMAR["📘 reference/mcp-shorthand-grammar.md<br/>locked output vocab (layer 1)"]
  GRAMMAR -.->|constrains| VAL
  GRAMMAR -.->|constrains| EX
```

**Layers (schema spine):** layer 1 = MCP shorthand (locked, the grammar doc) · layer 2 = superset
capture model (#3) · layer 3 = customizable input mapping (#4).

---

## View 2 — Artifact & handoff map

Where everything lives and how the planning hands off, ending in a build-ready spec.

```mermaid
flowchart TD
  subgraph REPO["repo: tennis_MatchChartingProject_tagger"]
    REF["📂 reference/<br/>grammar · workflow · MatchChart.xlsm"]
    PRD["📄 docs/PRD.md — v0.1 ✅"]
    DIAG["📄 docs/diagrams.md (this file)"]
    TLD["🖼 MCP | Match Charting Project.tldraw<br/>Stream Deck layout + Diagram page"]
  end

  MAP["🗺 Wayfinder map · issue #1"]

  R2["#2 Electron feasibility<br/>✅ closed → research branch"]
  T3["#3 Capture model + exporter<br/>▶ claimed"]
  T4["#4 Key-mapping & paging<br/>⛭ frontier"]
  T6["#6 Lifecycle & CSV export<br/>⏸ blocked by #3"]
  T5["#5 Video ↔ tagging<br/>✅ closed · ↝ post-MVP"]
  T7["#7 Assemble v1 spec<br/>⏸ blocked by #3,#4,#6"]
  BUILD["🔨 Build effort<br/>(separate, after spec)"]

  REF --> PRD
  PRD --> MAP
  DIAG -.orients.-> MAP
  MAP --> T3
  MAP --> T4
  MAP --> T6
  R2 -.informs.-> MAP
  T5 -.deferred.-> MAP
  T3 --> T7
  T4 --> T7
  T6 --> T7
  T3 --> T6
  T7 --> BUILD
```

**Handoff rule (wayfinder):** decisions resolve one ticket at a time; #7 is assembled *last* from the
resolved tickets, then handed to a build effort. The PRD sits above the map as product context.

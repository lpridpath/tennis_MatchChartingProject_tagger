# Diagrams — source of truth

Two views that center the project and index the handoffs. Version-controlled here; mirrored on the
tldraw canvas (`MCP | Match Charting Project.tldraw` → **Diagram** page) for whiteboarding. This file
is authoritative — update it when components or tickets change.

**Model (PRD v0.2):** the app is an **input bridge** — it captures pluggable input, builds a validated
MCP code string, and **writes it into the MatchChart `.xlsm`**, which stays the system of record.

Legend: **✅ locked/done** · **▶ in flight** · **⛭ frontier (takeable)** · **⏸ blocked** · **↝ deferred**.

---

## View 1 — System components (data flow)

```mermaid
flowchart LR
  subgraph SRC["Input sources — pluggable (#4)"]
    direction TB
    SD["🎛 Stream Deck XL<br/>primary"]
    KB["⌨️ Keyboard<br/>alternative"]
    OTH["… other (later)"]
  end

  subgraph APP["Electron + TS app  ·  INPUT BRIDGE  (✅ stack #2)"]
    direction TB
    IN["Input abstraction<br/>#4"]
    BLD["Code-string builder<br/>+ validator · ▶ #3"]
    WR["Workbook writer<br/>file-level · ✅ #8"]
    IN --> BLD --> WR
  end

  XLSM["📗 MatchChart .xlsm<br/><b>SYSTEM OF RECORD</b><br/>score · stats · CSV · metadata"]
  SUB["submit workbook (unchanged)"]
  JS["👤 Jeff Sackmann<br/>MCP dataset"]
  GR["📘 grammar.md<br/>MCP vocab (layer 1)"]

  SD --> IN
  KB --> IN
  OTH -.-> IN
  WR -->|"writes Y/Z/AC (file-level); Excel recalcs on open"| XLSM
  XLSM --> SUB --> JS
  GR -.constrains.-> BLD
```

**Layers:** layer 1 = MCP vocab (locked, grammar doc) · layer 3 = input mapping (#4, pluggable). The
old layer-2 superset model is dropped — the workbook is the model.

---

## View 2 — Artifact & handoff map

```mermaid
flowchart TD
  subgraph REPO["repo: tennis_MatchChartingProject_tagger"]
    REF["📂 reference/<br/>grammar · workflow · MatchChart.xlsm"]
    PRD["📄 docs/PRD.md — v0.2 ✅"]
    E2E["📄 docs/e2e-flow.md — v0.2"]
    CTX["📄 CONTEXT.md — glossary"]
    DIAG["📄 docs/diagrams.md (this file)"]
  end

  MAP["🗺 Wayfinder map · #1"]
  R2["#2 Electron feasibility<br/>✅ closed"]
  T8["#8 .xlsm write mechanism<br/>✅ closed · hybrid (headless v1)"]
  T3["#3 Code-string builder+validator<br/>▶ claimed"]
  T4["#4 Input mapping (deck+kbd)<br/>⛭ frontier"]
  T6["#6 Session & workbook targeting<br/>⛭ frontier"]
  T5["#5 Video ↔ tagging<br/>✅ closed · ↝ post-MVP"]
  T7["#7 Assemble v1 spec<br/>⏸ blocked by #3,#4,#6"]
  BUILD["🔨 Build effort"]

  REF --> PRD --> MAP
  E2E -.orients.-> MAP
  CTX -.orients.-> MAP
  DIAG -.orients.-> MAP
  R2 -.informs.-> MAP
  T5 -.deferred.-> MAP
  MAP --> T8
  MAP --> T3
  MAP --> T4
  MAP --> T6
  T8 --> T6
  T8 --> T7
  T3 --> T7
  T4 --> T7
  T6 --> T7
  T7 --> BUILD
```

**Frontier (takeable now):** #3 (claimed), #4, #6.

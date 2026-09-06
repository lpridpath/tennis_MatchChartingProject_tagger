# MCP Charting Workflow (Quick-Start flow)

The human workflow the Match Charting Project prescribes today. **This is the driver for this
project**: everything the tagger builds is an add-on/companion that makes *this flow* more
efficient — it does not replace the flow or the data format.

> Source: Jeff Sackmann, "The Match Charting Project Quick-Start Guide,"
> tennisabstract.com, 2015-09-23. Retrieved 2026-09-06.
> URL: https://www.tennisabstract.com/blog/2015/09/23/the-match-charting-project-quick-start-guide/
> Companion to `reference/mcp-shorthand-grammar.md` (the notation) and the `.xlsm` template.

## The flow today

### 1. Setup
- **Pick a match** from the project database (by date/player). Start with a *short* match; avoid
  lefties at first. Verify it isn't already claimed (avoid duplicate work).
- **Get video.** YouTube / streaming / DVR. Use a player with **frame-by-frame keyboard control**
  (SMPlayer, VLC) — essential for reviewing fast sequences.
- **Prep template.** Download the `.xlsm`, read the Instructions tab.

### 2. Point-by-point charting loop (the core)
For every point: watch → type the alphanumeric code string for the rally → the sheet auto-fills
score and advances to the next row. Heavy **pause / rewind / re-watch**, and frequent
**instruction-checking**, especially early on.

### 3. Progressive complexity (the charter levels up over many matches)
1. **Beginner:** serve direction + shot-type letters only.
2. **Intermediate:** add shot direction (`1`/`2`/`3`).
3. **Advanced:** add return depth (`7`/`8`/`9`) — "the hardest step to add."
4. **Expert:** add court-position modifiers (`+` `-` `=`).

### 4. Submission
Email the finished spreadsheet to the coordinator for DB integration.

---

## Friction → companion opportunity

The north-star: attack these friction points. Each maps to a ticket area. (Analysis, not scope
expansion — machine-assist / live / streaming ingest remain out of scope for the v1 map.)

| Friction in the manual flow | Where it hurts | Companion opportunity | Ticket area |
|---|---|---|---|
| **Typing terse code strings** while the rally moves fast | Core loop; error-prone | One Stream Deck key per token; visual, labeled, no typing | Key-mapping |
| **Constant pause/rewind**, tool-switching between video app and Excel | Every point | Video + tagging in one surface; playback on the deck | Playback interaction |
| **Instruction-checking** the notation | Early matches, and rare codes forever | Context-sensitive key layouts that *show* valid next tokens | Key-mapping |
| **Progressive-complexity cliff** (depth/direction hard to add live) | Levels 2–4 | Layered layouts: expose direction/depth as optional modifier keys, not mandatory typing | Key-mapping |
| **Score-keeping / mis-entry detection** (wrong auto-score = you erred upstream) | Long rallies | Live validation of the code string as you build it; reject malformed points | Capture model / exporter |
| **Grammar ordering rules** easy to violate | Anytime | Parser/serializer enforces well-formed output before it's ever saved | Capture model / exporter |
| **Match selection / dup-avoidance / submission** are manual, out-of-band | Setup + end | (Noted; largely out of scope v1 — export produces submittable CSVs) | Session lifecycle |

## Design principles this implies
- **The deck replaces the keyboard for coding, and the app replaces the video-app + Excel split.**
  Collapsing those two context-switches is the core efficiency thesis.
- **Optionality is a first-class feature**, not an afterthought — the flow is explicitly designed to
  be charted at 4 different depths. The tagger must let a user work at *any* tier and add detail
  progressively, mirroring the beginner→expert path.
- **Validation-as-you-go** turns the sheet's implicit "wrong score = you messed up" signal into an
  explicit, immediate one.

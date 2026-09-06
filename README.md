# tennis_MatchChartingProject_tagger

A companion / sidecar application for tagging and classifying tennis matches, feeding the
[Match Charting Project](https://github.com/JeffSackmann/tennis_MatchChartingProject) (MCP) data format.

## What it is

A human + machine control interface for reviewing and tagging tennis matches. The goal is to
make manual charting faster and more ergonomic by pairing an operator with a hardware control
surface — an **Elgato Stream Deck XL** (32 keys, 8×4 grid) — where each key maps to a shot,
outcome, or navigation action.

This is a **sidecar**, not a fork: it produces/annotates data compatible with the upstream
Match Charting Project rather than replacing it.

## Status

🚧 Early scaffolding. Wireframing and interface design in progress.

## Planned components

- **Control surface** — Stream Deck XL key layout mapping keys to charting actions
- **Tagging engine** — capture, validate, and serialize tags to MCP-compatible output
- **Review UI** — match playback + tag review/correction

## Related

- Upstream data project: [JeffSackmann/tennis_MatchChartingProject](https://github.com/JeffSackmann/tennis_MatchChartingProject)

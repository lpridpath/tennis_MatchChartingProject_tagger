# MCP Shorthand Grammar (Match Charting Project notation)

Formal token spec for the Match Charting Project point-coding language, extracted from the
**Instructions** tab of `MatchChart 0.3.2.xlsm` (Jeff Sackmann). This is the **locked output
vocabulary** (layer 1) the tagger must emit and validate against.

> Source of truth: `reference/MatchChart 0.3.2.xlsm` → Instructions sheet. If upstream revises
> the format, re-extract and bump this doc.

## Data-entry model

- One **point per row**. Up to three cells filled: **`1st`**, **`2nd`**, **`Notes`**.
- If the **1st serve lands in**, the *entire point* is coded in `1st`; `2nd` stays blank.
- If the **1st serve is a fault**, `1st` holds only that serve (direction + fault type); the
  `2nd` serve **and the whole rally** go in `2nd`.
- The workbook auto-populates score for the next row (score is derived, not charted).

A coded point is a single string: `serve [outcome] [rally...] [ending]`.

---

## 1. Serves

**Direction** (number, same in both courts): `4` = out wide · `5` = body · `6` = down the T · `0` = unknown.

**Fault types** (lowercase letter after direction, when serve is out):
`n` = net · `w` = wide · `d` = deep · `x` = wide **and** deep · `g` = foot fault · `e` = unknown fault.

**Serve modifiers:**
- `!` = shank (rare; used instead of a fault-type letter)
- `V` (uppercase) = time violation, server loses first serve
- `c` = let (repeat per let, e.g. `cc`)
- `+` = serve-and-volley attempt (in or out; placed after direction, e.g. `4+w`)

**Examples:** `6` T-serve in · `4` wide in · `6x` T serve wide+deep (fault) · `5d` body serve deep (fault) · `cc4e` two lets then a wide serve out, direction-of-out unknown.

## 2. Serve outcomes (points ending at/around the serve)

- **Ace:** add `*` → `5*` = body-serve ace.
- **Unreturnable:** add `#` → `6#` = T serve touched but not returned.
- **Forced error (return):** code the return shot + `#` → `6f#`.
- **Unforced error (return):** code the return shot + `@` → `6f2d@`.

Rule of thumb: *unreturnable* = returner gets no full racquet on it / can't reach the net / wild miss.
Everything else is a forced/unforced return error.

---

## 3. Rally shots

Each rally shot (except returns & point-ending forced errors) = **shot-type letter + direction number**.

**Shot-type letters:**

| Code | Shot | Code | Shot |
|---|---|---|---|
| `f` | forehand groundstroke | `b` | backhand groundstroke |
| `r` | forehand slice | `s` | backhand slice |
| `v` | forehand volley | `z` | backhand volley |
| `o` | overhead/smash | `p` | "backhand" overhead/smash |
| `u` | forehand drop shot | `y` | backhand drop shot |
| `l` | forehand lob | `m` | backhand lob |
| `h` | forehand half-volley | `i` | backhand half-volley |
| `j` | forehand swinging volley | `k` | backhand swinging volley |
| `t` | trick shot (tweener, behind-back, etc.) | `q` | unknown shot |

**Direction** (relative to a right-hander; mirror for lefties): `1` = to righty's FH side ·
`2` = down the middle · `3` = to righty's BH side · `0` = unknown. Direction is **optional**.

**Examples:** `f1` crosscourt FH · `s2` BH slice up middle · `u3` FH drop to BH side ·
`fbh` FH, BH, half-volley (no directions — allowed).

## 4. Return depth (optional)

When a service return is in, append a **third** char for depth:
`7` = within the service boxes · `8` = behind service line (nearer service line) ·
`9` = nearer the baseline · `0` = unknown. → returns are 3 keystrokes: type + direction + depth
(e.g. `f37`).

## 5. Rally endings

- **Winner:** append `*` → `f3*` = FH winner down the line.
- **Error:** code the *attempted* final shot, then an **error type**, then **forced/unforced**:
  - error types (same set as serves): `n` net · `w` wide · `d` deep · `x` wide+deep · `!` shank · `e` unknown
  - `@` = unforced error · `#` = forced error
- Error shot = up to 4 keystrokes: `type / direction / error-type / (@|#)`.
  - Unforced **requires** type + error-type + `@`; direction optional.
  - Forced **requires** type + `#` only (e.g. `b#`); more detail allowed (`b3d#`).

**Full example:** `5f2f1f1v2n@` = body serve in, FH mid, FH cc, FH cc, volley mid into net (unforced).

## 6. Court-position / shot modifiers (optional, lowest priority)

Placed immediately after the shot code:
- `+` = approach shot (also serve-and-volley on serve) — e.g. `b+2` BH approach up middle
- `-` = shot taken at the **net** (override default) · `=` = shot taken at the **baseline**
- `;` = clipped the net cord — e.g. `f;1*`
- `^` = stop volley / drop volley — e.g. `z^2*` BH drop-volley winner up middle

## 7. Special / whole-point codes (entered in `1st`)

- `S` = missed point, awarded to **server** · `R` = missed point, awarded to **returner**
- `P` = point penalty against server · `Q` = point penalty against returner
- `C` = player stopped play for an **incorrect** challenge — e.g. `6b29C`

## 8. Match-level fields (header, not per-point)

Players (server-first), dominant hands, tournament/round/date/surface (match TennisAbstract.com
formats). **Final-set / tiebreak rule** field:
`1` = final-set tiebreak (default) · `0` = advantage set · `S` = super-tiebreak-only · `W` = TB at 8-all ·
`V` = no tiebreaks · `A` = 10-pt super-TB at 6-all · `T` = TB at 12-all · `N` = NextGen format.

---

## Implications for the tagger (informational)

- **"Unknown" escape codes exist at every layer** (`0`, `q`, `e`, `R`/`S`) — the input model must
  make these fast to reach, not buried.
- **Ordering & optionality are grammar rules**, not free text — a validating exporter (layer-1)
  can enforce well-formed strings. Good target for a parser/serializer with a test suite.
- **The alphabet is far larger than 32 keys** (≈20 shot types × directions × depths × modifiers +
  serves + endings) → the key-mapping model needs context-sensitive pages/modifiers, not a flat map.
- Distinct FH/BH letters per shot family suggest a **hand-side modifier** could halve the visible
  shot-type keys.

# ROLL & SHOOT — World Cup Penalty Tournament
*Design doc + Claude Code briefing · last updated June 2026*

---

## What this is
A browser penalty shootout game with a full World Cup tournament wrapper. You draft a cross-era squad from real historical World Cup teams, then play through a group stage and knockout bracket where every match is a live penalty shootout you both **take** and **defend**.

---

## File structure
```
Roll&Shot/
├── index.html   ← entire game, ~1100 lines, single file
└── assets/
    └── football.png                 ← ball sprite (referenced locally)
```

Everything lives in one HTML file: HTML structure, CSS (custom properties + utility classes), and a single `<script>` IIFE. No build step, no dependencies, no framework. Canvas for the pitch/ball/keeper; DOM cards for all menus and tournament screens.

---

## Current build status — what's DONE

### Build screen
- Roll button animates through random squads (11 ticks × 65ms)
- Each roll reveals 3 takers from that squad → player picks one
- After 5 takers, 3 keeper options are revealed → player picks one
- Squad identity = keeper's code + year (e.g. "TUR '02")
- Progress bar shows SHOOTER 1–5 → KEEPER

### Penalty shootout (canvas)
**Attack sequence (3 inputs, one at a time):**
1. Aim — move reticle in the goal with pointer/keyboard
2. Power — oscillating bar, tap/click to lock
3. Curve — left/right bar, tap/click to lock

Ball flies arc; AI keeper dives based on aim read. Result: GOAL / SAVE / OUT.

**Defend sequence:**
- AI taker has a chosen target zone (3 thirds of goal)
- A directional cue appears briefly; player picks dive direction + timing
- Keeper's radius determines save range
- Two keeper styles toggle-able: Dive (anticipate) vs React (wait for cue)

Stats wire directly into mechanics: Power widens the strong zone on the power bar; Accuracy controls aim drift; Technique affects curve sweet spot; Composure slows meters under pressure. Keeper rating scales the save circle radius.

**Shootout flow:** 5 rounds alternating attack/defend. Tied after 5 → sudden death. Score tracked live in the scorebox.

### Tournament flow
```
BUILD SQUAD
  ↓
GROUP STAGE [random letter A–H]
  - You + 3 opponents (always real squads from SQUADS array)
  - Play 3 live penalty shootouts
  - Each matchday: your match plays live → paired other match auto-sims
  - Group table: P / W / L / Pts, top 2 qualify (green Q badge)
  - Tiebreaker: points → goals scored → alphabetical (no player bias)
  ↓
QUALIFICATION CHECK (top 2 only → otherwise eliminated screen)
  ↓
KNOCKOUT BRACKET (16 teams, single elimination)
  - All 15 opponents are real squads (never ghosts)
  - R16 → QF → SF → FINAL
  - Before you play: bracket shown unresolved (clean view)
  - You play your match live → return to bracket → rest of round auto-sims and reveals
  - If eliminated: bracket shows + "SIMULATE TO END" button → sims remaining rounds → shows who won
  ↓
CHAMPION screen or ELIMINATED screen (with tournament winner shown)
```

### Screens / states
| State | Description |
|---|---|
| `build` | Roll & draft screen |
| `group` | Group stage hub — next match box, matchday results, table |
| `shootout` | Live penalty match (canvas) — reused for all live matches |
| `summary` | Full time result + round-by-round log → CONTINUE routes back |
| `bracket` | 16-team knockout bracket, flag + era slots |
| `champion` | Winner screen with run history |
| `eliminated` | Knocked out screen with round + tournament winner |

### Data
**16 real squads** (always available for draft and bracket opponents):
BRA 2002, FRA 1998, ARG 1986, ARG 2022, GER 2014, ITA 2006, ESP 2010, ENG 1990, TUR 2002, NED 1998, CRO 2018, POR 2006, BRA 1970, URU 2010, ENG 2018, GER 1990

Each squad: `{ code, year, col[3], takers[{n, p, a, t, c}], keeper:{n, rating} }`

**16 ghost teams** (real 2026 WC nations — bracket filler for the group stage pool only, never played live):
MAR, USA, JPN, MEX, BEL, SEN, COL, KOR, SUI, DEN, AUS, GHA, ECU, CAN, NGA, CMR

Ghost structure: `{ code, year:2026, col[3], rating:6–9, ghost:true }` — no takers/keeper.

---

## Architecture — key patterns for Claude Code

### State variables (top of IIFE)
```js
let screen = 'build'           // current screen name
let tournament = null          // full tournament state object (null until squad is built)
let lineup = []                // drafted takers [{code, year, col, player}]
let chosenKeeper = null        // {code, year, col, keeper}
let opponent = null            // squad currently being played against (live match)
let round, half, yourScore, oppScore  // shootout state
let atkRes = [], defRes = []   // per-round results for summary
```

### Tournament object shape
```js
tournament = {
  letter: 'D',                 // group letter (cosmetic)
  you: {code, year, col, you:true},  // bare identity (no takers/keeper)
  group: [{team, you, P, W, L, Pts, gf}, ...],  // 4 entries, index 0 = player
  yourFix: [{a, b, played, you:true, aScore, bScore, youWon}, ...],  // 3 fixtures
  otherFix: [{a, b, played, aScore, bScore}, ...],  // 3 fixtures (non-you)
  stage: 'r16',                // group | r16 | qf | sf | final | done
  bracket: {r16, qf, sf, final},  // arrays of matchup objects
  run: [{label, opp, score, won}],  // history for end screens
  returnTo: 'group',           // where to go after a live match ends
  curMatchRef: null,           // {type, fixIdx} or {type, round, idx, youIsA}
}
```

### Screen routing
`setScreen(s)` shows/hides all screens. After a live match ends → `showSummary()` → player clicks CONTINUE → `routeAfterMatch()` → routes back to `group` or `bracket` based on `tournament.returnTo`.

### Key functions
- `startTournament()` — builds group, sets tournament object, calls `renderGroup()`
- `playLiveMatch(squad, returnTo, ref)` — sets `opponent`, resets shootout state, calls `setScreen('shootout')`
- `enterBracket()` — checks qualification, calls `buildBracket()` + `proceedBracket()`
- `proceedBracket()` — shows bracket unresolved, wires PLAY button
- `afterYourBracketMatch(won)` — finalizes round, seeds next, advances stage or shows eliminated
- `simulateRest(round)` — seeds and sims all remaining rounds after elimination, shows winner
- `simMatch(a, b)` — weighted coin flip using `squadStrength()`; returns `{winner, aScore, bScore}`
- `squadStrength(s)` — returns 1–10 strength; handles ghost (uses `s.rating`), bare identity (`!s.takers` → returns 8), and real squad (averages taker stats + keeper)

### Flag rendering
SVG flags are embedded as base64 data URLs in a `FLAG_SVG` map keyed by ISO2 code. `CODE2ISO` maps squad codes to ISO2. `flagStyle(code, col)` returns an inline style string — uses SVG if available, falls back to CSS gradient from `col[3]`.

---

## Visual design system
```css
--ink:    #15110d   /* near-black, borders and text */
--paper:  #f0e9d6   /* warm cream, card backgrounds */
--pink:   #ff2e63   /* accent, your matches, warnings */
--blue:   #2b50e2   /* accent, shadows */
--yellow: #ffc42e   /* current/live highlight */
--green:  #3e9b4f   /* wins, qualification, advance */
```

Cards: `border: 4px solid var(--ink); box-shadow: 8px 8px 0 var(--blue)`. Impact font for all headings and labels. Riso/punk-zine aesthetic — bold, flat, high-contrast.

---

## What's NOT done yet (next up)

- **Pressure scaling by round** — composure stat exists in data but pressure only toggles on/off in practice mode; it doesn't auto-scale across group → knockouts → final
- **Sound design** — Web Audio SFX exist (kick, goal, save) but crowd reactions, ambient noise, music not implemented
- **Champion/eliminated screen polish** — functional but sparse; no animation, no confetti
- **assets/football.png integration** — football.png is in the assets folder locally but not yet wired into the canvas draw (currently using a procedural circle)
- **Mobile layout** — works but not optimized; bracket overflows on narrow screens
- **Persistent stats / run history** — no save/resume across sessions (by design for v1)
- **Share-your-squad** — not implemented

---

## How to work on this in Claude Code

```bash
cd /path/to/Roll\&Shot
claude
```

The game is fully self-contained in `index.html`. Open it directly in a browser to test (or `python3 -m http.server 8080` if you need asset loading). All JS is in one `<script>` tag as an IIFE — no modules.

When asking Claude Code to edit: reference function names from the architecture section above. The file is ~1100 lines; key sections are clearly commented with `// ===== SECTION NAME =====` headers.

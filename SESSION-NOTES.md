# Matchstick Game — Full Project Status

## Overview
A Theory of Constraints simulation based on "The Goal" by Eli Goldratt. Players observe how statistical fluctuations and dependent events degrade throughput in a production line. Three pages, all single-file HTML/CSS/JS with Chart.js — no build tools.

**Location**: `C:\Users\marcl\matchstick-game\`

> **WARNING**: The `index.html` at `C:\Users\marcl\index.html` (parent directory) is a completely unrelated GSD WebSocket dashboard. The matchstick game lives only in the `matchstick-game\` folder.

---

## Files

| File | Lines | Purpose |
|------|-------|---------|
| `index.html` | ~1400 | **Observe** mode — watch the line run with no interventions |
| `sandbox.html` | ~2800 | **Sandbox** mode — full intervention controls |
| `game.html` | ~86 | **Gam-ba** mode — placeholder "Coming Soon" page |

All three pages share the same visual style (dark blue theme, gold accents) and nav tabs: **Observe | Sandbox | Gam-ba**

---

## Architecture

### Observe Page (`index.html`)
- Single simulation engine, no interventions
- `BUFFER_CAP = 4` (hard-coded) — explained in rules as rounding up from 3.5 average
- Stations: 3-10, configurable
- Chart: Cumulative actual vs expected, lost throughput, WIP, overproduction, per-round bars, flow score, bottleneck markers
- Pause button during auto-play
- Bottleneck detection via rolling window of 10 rounds
- Stat label: "Avg Throughput / Round"

### Sandbox Page (`sandbox.html`)
- **Dual-engine architecture**: `playerState` (with interventions) + `controlState` (vanilla, same dice rolls)
- `DEFAULT_BUFFER_CAP = 4`
- The control engine runs silently in the background for comparison

#### Global State Variables
```js
let playerState = null;       // user's line — rendered visually
let controlState = null;      // vanilla line — runs silently
let stationConfigs = [];      // per-station intervention configs
let releaseRate = 6;
let budgetUsed = 0;
let budgetEnabled = false;
let budgetDifficulty = 'easy';
let chart = null;
let autoPlaying = false;
let paused = false;
let animating = false;
let comparisonVisible = false;
let showControl = true;
let defectsEnabled = false;
let defectDifficulty = 'easy';
let diceControlEnabled = false;
let randomReducerEnabled = false;
```

#### Constants
```js
const NAMES = ['Herbie','Davey','Ron','Chuck','Evan','Andy','Alex','Ben','Jake','Hank'];
const DEFAULT_BUFFER_CAP = 4;
const BUDGET_LEVELS = { easy: 30, medium: 20, hard: 14, extreme: 8 };
const DEFECT_RANGES = { easy: [0.05,0.10], medium: [0.10,0.20], hard: [0.20,0.30], extreme: [0.30,0.50] };
const REDUCER_PRESETS = { none: [1,6], slight: [2,5], moderate: [3,4] };
```

#### Station Names
All male names from "The Goal" (boy scouts): Herbie, Davey, Ron, Chuck, Evan, Andy, Alex — then Ben, Jake, Hank for stations 8-10.

#### Per-Station Config (`stationConfigs[i]`)
```js
{ bufferCap: 4, diceMin: 1, diceMax: 6, overtime: false, inspectBefore: false }
```

#### Controls Layout

**Top Row** (action controls): Roll Dice, Auto Play, Pause, Speed slider, Stations dropdown, Show Control toggle, Reset button

**Interventions Bar** (mode toggles, each in a `.mode-group` styled box):
1. **Release Rate** — dropdown 1-16, default 6. Free.
2. **Budget Mode** — checkbox toggle, with difficulty dropdown + budget bar display. Contains nested Dice Mode.
   - **Dice Mode** (tri-toggle, nested inside Budget Mode, disabled when budget off):
     - Normal = standard 1-6 dice
     - Reducer = per-station preset dropdown (1-6 / 2-5 / 3-4)
     - Dice Control = per-station min/max selects
3. **Buffer Control** — checkbox toggle, reveals per-station buffer dropdowns (2-16). Free (but increases cycle time).
4. **Defects Mode** — checkbox toggle with difficulty dropdown. Reveals per-station "inspect" checkboxes.

#### Budget System
- **Budget gating**: Dice Mode requires Budget Mode to be enabled. When budget is off, dice radios are disabled and locked to Normal. OT buttons are also hidden when budget is off.
- **Budget costs**: Dice narrowing (1pt per step from 1-6), Overtime (3pts per station), Inspection (1pt per station)
- **Free interventions**: Release Rate, Buffer Size
- **Budget levels**: Easy 30, Medium 20, Hard 14, Extreme 8
- When budget is toggled OFF: dice resets to Normal, overtime cleared, all radios disabled
- `enforcebudget()` disables controls that would exceed budget (OT buttons, inspect checkboxes, dice min/max options, reducer presets)
- Buffer options are NOT budget-enforced (buffers are free)

#### Key Functions
| Function | Purpose |
|----------|---------|
| `createInitialState(n)` | Creates fresh state object with all history arrays |
| `createDefaultConfigs(n)` | Creates per-station configs with defaults |
| `calculateBudget()` | Sums budget cost: dice narrowing + overtime + inspection |
| `enforcebudget()` | Disables controls that would exceed budget |
| `executeRound()` | Orchestrator: generates shared dice rolls, runs both engines |
| `executeVanillaRound(st, rolls)` | Control engine: no interventions, same dice |
| `executePlayerRoundClean(st, rolls)` | Player engine: applies interventions, defects, inspection |
| `updateBottleneckDetection(st, stations)` | Rolling window bottleneck detection |
| `buildDOM()` | Builds station HTML with all controls |
| `renderStats()` | Updates all stat cards with values, trend arrows, control comparison |
| `initChart()` | Creates Chart.js chart with 8 datasets |
| `updateChart()` | Pushes new data to chart |
| `resetGame()` | Full reset of state, UI, chart |
| `onBudgetToggle()` | Toggles budget, enables/disables dice radios, shows/hides OT |
| `onDefectsToggle()` | Toggles defects, shows/hides inspect checkboxes, calls updateBudgetDisplay |
| `onDiceModeChange()` | Switches dice mode, resets previous mode's values, updates UI |

#### Chart Datasets (8 total, in order)
0. Your Actual (green line, borderWidth: 4)
1. Expected (gold dashed line, borderWidth: 4)
2. Lost Throughput (red filled area)
3. WIP Inventory (teal #20b0b0 line, y2 axis)
4. Overproduction (purple filled area)
5. Per-Round Throughput (light green bars)
6. Flow Score (hot pink #e840a0, y4 axis, 0-100%)
7. Bottleneck Station (bottle icons on hidden y5 axis — orange stable, red moving)

Control lines were removed from chart; control values shown under stat cards instead.

#### Stat Cards
**Row 1**: Round, Flow Score (pink, green/yellow/red glowing border), Cycle Time (teal, 1 decimal place e.g. "5.3 min"), Performance (sky blue), Bottleneck (fixed-size card, min-width 160px, min-height 2.2em)
**Row 2**: Total Throughput, Expected Throughput, Avg Throughput / Round, WIP Inventory, Total Lost Throughput, Total Overproduction, Scrap (hidden unless Defects Mode on)

#### Cycle Time Display
- Shows 1 decimal place everywhere: stat card ("5.3 min"), control line ("ctrl: 6.7 min"), comparison table
- Green threshold: ≤ 1.5x theoretical min (was 1.35x)
- Yellow threshold: ≤ 2x theoretical min
- Red threshold: > 2x theoretical min

#### Trend Arrows
- Flow Score: up = green (good), down = red (bad)
- Cycle Time: down = green (good), up = red (bad)
- WIP: up = red (bad), down = green (good)
- Avg Throughput / Round: up = green, down = red
- Lost Throughput: % of total throughput, increasing = red, decreasing = green
- Overproduction: % of total throughput, increasing = red, decreasing = green
- Scrap: up = red, down = green
- Performance: up = green, down = red

#### Flow Score Card Border
- Green glow: score > 80
- Yellow glow: score 50-80
- Red glow: score < 50

#### Animation Speeds (all halved from original)
| Animation | Duration | Where |
|-----------|----------|-------|
| Stat green pulse | 8s | Flow/CT/Perf/BN cards |
| Stat yellow pulse | 4s | Flow/CT/Perf/BN cards |
| Stat red pulse | 1.6s | Flow/CT/Perf/BN cards |
| Stat siren (critical) | 1s | BN card |
| Bottle glow red pulse | 1.6s | Station bottles (multi) |
| Bottle siren (critical) | 1s | Station bottles (critical) |
| Siren text (critical) | 1s | Station names (critical) |

#### Bottleneck Card Stability
- `min-width: 160px` on `#bottleneckCard`
- `.stat-value` inside has `min-height: 2.2em` with flexbox centering
- Prevents layout shifts when text changes between "None", station names, and "CRITICAL (...)"

#### Bottleneck Visualization
**On chart**: Custom canvas-drawn beer bottle icons as Chart.js pointStyle
- `createBottleIcon(color)` — 14x28 canvas with cap, lip ring, narrow neck, shoulder curves, wide body, label band
- Orange (`#e08030`) for stable, red (`#d04040`) for moving

**On stations**: SVG beer bottle silhouette (`.bottle-glow`) behind each station
- Hidden by default, shown via CSS when station has `bottleneck-stable`, `bottleneck-multi`, or `bottleneck-critical` class
- Orange glow + drop-shadow for stable, red glow + pulsing for multi, red/blue siren for critical
- Tilted 30 degrees (`rotate(-30deg)`), 75x160px, centered on station

#### Disabled Dice Mode Styling
```css
.tri-toggle input[type="radio"]:disabled + label {
  opacity: 0.35;
  cursor: not-allowed;
  pointer-events: none;
}
```

#### Comparison Dashboard
- Toggled via "Show Comparison" button below stats
- Side-by-side table: Your Line vs Control for throughput, lost, WIP, overproduction, flow score, avg throughput/round, cycle time (1 decimal), performance, scrap

---

## Rules & Strategy Section (sandbox.html)
- **Sandbox Mode** explanation
- **Interventions**: Release Rate (free, 1-16), Overtime (3pts, requires budget), Inspection (1pt, requires defects)
- **Modes**: Budget Mode (with nested Dice Mode explanation including Normal/Reducer/Dice Control costs and example), Defects Mode (can toggle mid-game), Buffer Control (free, 2-16, increases cycle time)
- **Control Line & Comparison** explanation
- **Metrics Explained**: Flow Score, Cycle Time (Little's Law, 1.5x/2x thresholds), Performance Score (geometric mean)
- **Strategy Tips** — 6 bullet points
- **Bottleneck Severity Levels** — None, Single/Stable, Multiple/Moving, CRITICAL

---

## Pending / Future Work
1. **Gam-ba mode** (`game.html`) — preset rounds, levels with increasing difficulty. Currently just a "Coming Soon" placeholder. No implementation yet.
2. The user may want additional features/tweaks — they iterate frequently with visual and UX changes.

---

## Style Conventions
- Dark blue theme: `#0a1628` bg, `#0f1f3a` panels, `#1a3050` borders
- Gold accents: `#f0c040` for headers, active tabs, highlights
- Stat value colors: green `#30a060`, red `#d04040`, orange `#e08030`, teal `#20b0b0`/`#20d0d0`, pink `#e840a0`, blue `#4090d0`, purple `#a060d0`, sky `#60c0f0`
- Control values (`.stat-ctrl`): bright white `#ffffff`, 0.75rem, bold
- Mode toggle groups: `.mode-group` with dark bg, border, rounded corners
- All pages use same nav-tab style, header gradient, font stack ('Segoe UI', system-ui)

---

## Key Lessons Learned
- When writing complex game logic with multiple interdependent state mutations (buffers, inspection, defects, dice clamping), write the clean version first rather than trying to iterate mid-function.
- The dual-engine architecture requires shared dice rolls but separate state — always generate rolls first, then pass to both engines.
- Chart.js custom pointStyle via canvas works well for bottle icons but canvas dimensions matter for visual clarity (current: 14x28).
- `enforcebudget()` must handle all control types including reducer selects when budget mode is on.
- When toggling modes mid-game (e.g. defects on/off), always call `updateBudgetDisplay()` to ensure budget enforcement stays in sync — not just on toggle-off.
- Nesting related controls (Dice Mode inside Budget Mode) is cleaner than having standalone toggles with hidden dependencies.

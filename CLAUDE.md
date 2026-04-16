# CLAUDE.md — Transformation System / OperAxis

## Project Overview

This is a **personal fitness and habit-tracking web app** called "Transformation System" (personalized) or "OperAxis" (generic/template). It tracks daily habits across six life domains, logs body metrics (calories, steps, weight), and generates weekly coach reports.

The app is **entirely self-contained in a single HTML file** — no server, no build step, no package manager. Open the file in a browser and it works.

Two variants exist in the repo:
- `index.html` — personalized deployment (`COACH_EMAIL = "wnut3080@gmail.com"`, title "Transformation System")
- `System` — generic template (`COACH_EMAIL = "coach@example.com"`, title "OperAxis System")

Both files are functionally identical except for those two strings. When making feature changes, apply them to **both files** unless the change is deployment-specific.

---

## Repository Structure

```
System/
├── index.html   # Personalized deployment (primary)
├── System       # Generic template variant (no .html extension)
└── README.md    # Minimal placeholder
```

No subdirectories. No package files. No CI/CD. No tests.

---

## Architecture

Each HTML file has three script blocks:

### Block 1: `<script>` — Vanilla JS globals
Defined before React loads. Contains:
- **Flywheel SVG constants** (`FW_NODES`, `FW_RING`, `FW_COL`, `FW_DET`) — node positions and colors for the System tab SVG
- **`fwEdge()`** — geometry helper for drawing arrows between flywheel nodes
- **`buildFlywheel()`** — imperatively constructs SVG arc/spoke paths at runtime
- **`fwSel(key)`** — handles flywheel node click, shows detail panel, triggers React callback via `window._fwOnSelect`
- **`_fwObs`** — MutationObserver that re-builds flywheel when the SVG enters/leaves the DOM
- **`PROTOCOLS`** array — static content for each domain's protocol detail view

### Block 2: `<script type="text/babel">` — React app (JSX)
Babel compiles this at runtime. Contains:
- Global constants and utility functions
- React component definitions
- `ReactDOM.createRoot(...).render(...)` bootstrap call

### CSS
Minimal global resets and two named rules (`#detail-panel`, `.nc`, `@keyframes shake`) inline in `<style>`.

---

## Key Configuration Constants

All are defined at the top of the Babel script block (easy to find and edit):

| Constant | Value | Purpose |
|---|---|---|
| `APP_PIN` | `"1234"` | Lock screen PIN |
| `COACH_EMAIL` | varies by file | Weekly report destination |
| `BMR` | `1969` | Basal metabolic rate (kcal) |
| `BASE` | `Math.round(BMR * 1.2)` = `2363` | Daily calorie base (sedentary TDEE) |
| `WBURN` | `{Push:408, Pull:408, Lower:476, Calisthenics:544, Rest:0}` | Extra kcal burned per workout type |
| `KPS` | `0.045` | Calories burned per step |

**Storage key:** `"operaxis_v4"` — all data is stored under this key in `localStorage`. Changing this key wipes all existing user data.

**Calorie burn formula:** `calcBurn(workoutType, steps) = BASE + WBURN[workoutType] + round(steps * KPS)`

---

## Data Model

All data lives in `localStorage` under the key `"operaxis_v4"` as a JSON object keyed by ISO date strings (`"YYYY-MM-DD"`):

```js
{
  "2024-01-15": {
    checks: {
      mental:         [true, false, true],     // one bool per habit, indexed by position
      fitness:        [true, true, false, true],
      nutrition:      [false, true, true, false],
      recovery:       [true, false, true],
      grooming:       [true, true, false, true],
      accountability: [true, false, true]
    },
    calIn:        2100,       // calories eaten (number)
    calBurn:      2800,       // estimated calories burned (calculated)
    stepsNum:     9200,       // steps walked (number)
    weight:       198.5,      // body weight in lbs (number, optional)
    workoutType:  "Push",     // one of WTYPES: Push|Pull|Lower|Calisthenics|Rest
    workoutNotes: "Bench 3x8 @ 185lbs"  // free text (string, optional)
  }
}
```

`ld()` loads data, `sv(data)` saves it. Both wrap `localStorage` in try/catch.

---

## Domain System

Six tracking domains form a ring around a central Identity node:

| Domain | Color | Habits (count) |
|---|---|---|
| Mental | `#7F77DD` | 3 |
| Fitness | `#378ADD` | 4 |
| Nutrition | `#639922` | 4 |
| Recovery | `#e8e8e8` | 3 |
| Grooming | `#D4537E` | 4 |
| Accountability | `#EF9F27` | 3 |

Each habit has a weight (`w`). Domain score = `sum(earned weights) / sum(total weights) * 100`. Day score = average of all six domain scores.

**Score colors:** ≥75% → green (`#34d399`), ≥40% → blue (`#60a5fa`), >0% → amber (`#f59e0b`), 0% → dark grey (`#333`).

---

## Component Reference

| Component | Purpose |
|---|---|
| `LockScreen` | PIN entry screen shown before `unlocked === true`. 4-digit keypad with shake animation on wrong PIN. |
| `CalChart` | Chart.js bar+line chart for calories eaten, burned, and deficit over a date range. |
| `WtChart` | Chart.js line chart for bodyweight trend over 30 days. |
| `Flywheel` | SVG visualization of the 7-node flywheel. Clicks imperatively trigger `fwSel()` (vanilla JS) which calls back into React via `window._fwOnSelect`. Uses MutationObserver to rebuild arcs after React re-renders. |
| `ProtocolDetail` | Renders protocol content for a selected domain from the `PROTOCOLS` array. Supports section types: `mission`, `nonneg`, `layers`, `priorities`. |
| `App` | Root component. Manages all state (tabs, daily log, inputs). Renders tab navigation and all tab content inline. |

**Component tree:**
```
App
├── LockScreen (shown when !unlocked)
├── Tab nav bar
└── Tab content (inline JSX, not separate components)
    ├── "daily"   → domain checklists + nutrition/body inputs
    ├── "dashboard" → 7-day table + domain averages
    ├── "monthly" → month calendar grid + averages
    ├── "weekly"  → domain bars + overall score + report button
    ├── "body"    → CalChart, WtChart, workout summary
    ├── "system"  → Flywheel or ProtocolDetail
    └── "settings" → coach email display, export/import, clear data
```

---

## Tabs / Navigation

| Tab ID | Label | Content |
|---|---|---|
| `daily` | Daily | Habit checklists for each domain + nutrition/body inputs + Save Day button |
| `dashboard` | Dashboard | Streak, today %, 7d avg stats; 7-day domain table; 7-day domain averages |
| `monthly` | Monthly | Calendar grid for any month (navigate with arrows); month averages |
| `weekly` | Weekly | Domain 7d averages; overall weekly score; reflection textarea; email report button |
| `body` | Body | Calorie chart (7d/month toggle); workout log; weight chart (30d) |
| `system` | System | Flywheel SVG → click node → ProtocolDetail for that domain |
| `settings` | Settings | Coach email display; export/import JSON backup; clear all data |

---

## Scoring System

```
habit score    = checks[i] ? habit.w : 0
domain score   = round(sum(habit scores) / sum(habit weights) * 100)
day score      = round(average of all 6 domain scores)
```

Key functions:
- `dPct(checks, habits)` — domain score for one domain
- `dayPct(dayData)` — overall day score
- `pc(percent)` — returns color hex for a score value

---

## Development Workflow

**No build system.** To run the app:
1. Open `index.html` in any modern browser (double-click the file, or `python3 -m http.server` and navigate to it)
2. Enter PIN `1234` to unlock

**To test changes:**
- Edit the file, save, hard-refresh the browser (`Cmd+Shift+R` / `Ctrl+Shift+R`)
- Open DevTools console to catch JS errors
- Babel compiles the JSX inline — syntax errors appear in the console

**Data persistence:** All data is in `localStorage`. Open DevTools → Application → Local Storage to inspect or clear it. Use the Settings tab's Export/Import to backup and restore.

**No automated tests.** Manual testing is the only verification path.

---

## Two-File Pattern

`index.html` and `System` are near-identical. Differences:
- `<title>`: "Transformation System" vs "OperAxis System"
- `COACH_EMAIL`: `"wnut3080@gmail.com"` vs `"coach@example.com"`

**When making feature changes:** Edit both files unless the change is deployment-specific (e.g., changing the coach email for one user only). Diff the files to confirm they stay in sync on everything else.

---

## Code Conventions

- **Function names:** camelCase (`calcBurn`, `fmtDate`, `dayPct`, `exportD`, `importD`)
- **Variable names:** short abbreviations are the norm (`cis`, `sts`, `wts`, `lds`, `drs`, `aps`)
- **Constants:** ALL_CAPS for static data arrays/objects (`DOMAINS`, `WTYPES`, `WBURN`, `FW_NODES`)
- **Styles:** all inline React style objects with camelCase properties; no CSS classes except `.nc` and `#detail-panel`
- **State pattern:** `var xs = useState(...)` then `var x = xs[0], setX = xs[1]` (pre-destructuring style)
- **Comments:** none by default (consistent with the existing code)
- **No TypeScript**, no prop-types, no linting config

---

## Common Change Patterns

### Change the PIN
Find `var APP_PIN="1234"` in the Babel script block and update the string. Do this in both files if needed.

### Change calorie targets
Update `BMR` and/or `BASE`. `BASE` is auto-derived from `BMR * 1.2`. Update `WBURN` values to change per-workout burn estimates.

### Add a habit to a domain
Find the domain object in the `DOMAINS` array and add a `{label: "...", w: N}` entry. The weight `w` is relative — pick a value consistent with existing habits (1–4). The `checks` array for that domain in localStorage will auto-extend (missing indices are treated as `false`).

### Change domain colors
Update both `FW_COL` (flywheel SVG colors) and `DOM_COL` (dashboard table colors) — they must stay in sync.

### Add a new workout type
Add the type string to `WTYPES` and add its calorie burn to `WBURN`.

### Update protocol content
Edit the `PROTOCOLS` array in the vanilla JS block. Each protocol entry supports section types: `mission`, `nonneg`, `layers`, `priorities` — follow the existing shape exactly.

---

## What NOT to Do

- **Do not change the localStorage key** (`"operaxis_v4"`) — it will silently wipe all user data
- **Do not add npm/build tooling** — the zero-dependency, open-in-browser design is intentional
- **Do not add external CDN libraries** without verifying the app still works fully offline/locally
- **Do not split into multiple files** — the single-file design is a feature (easy to share, deploy anywhere)
- **Do not add a backend/server** — all data is intentionally client-side only
- **Do not change the `type="text/babel"` script attribute** — it is required for JSX compilation

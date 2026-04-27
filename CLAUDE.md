# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Office on-call scheduling SPA delivered as a single, self-contained HTML file (`index.html`). No build tooling, no npm, no server — open the file in a browser and it runs.

## Running the App

```bash
# Open directly in browser (no server needed)
xdg-open index.html        # Linux
open index.html            # macOS
```

There are no build, test, or lint commands — the entire application is in `index.html`.

## Architecture

`index.html` (~911 lines) is divided into:

1. **CDN imports** — React 18.2 and ReactDOM loaded from CDN via `<script>` tags; no bundler.
2. **Constants** (top of `<script>`) — Theme colors (`DARK`/`LIGHT`), config (`CFG`), holiday colors (`HCOL`), default timeframe (`DEFAULT_TF`), and the default employee CSV (`DCSV`).
3. **Utility functions** (lines ~94–303) — Pure functions for dates, employee ID generation, CSV parsing/building, holiday resolution, and the scheduling algorithm.
4. **React components** (lines ~313–911):
   - `TypeInput` — Small controlled input for part-time cap values.
   - `App` — Single monolithic component that owns all application state and renders tabs: `calendar`, `summary`, `settings`, `guide`.

## Key State (inside `App`)

| Variable | Purpose |
|----------|---------|
| `tf` | Timeframe `{startMonth, startYear, endMonth, endYear}` |
| `sched` | `{ "YYYY-MM-DD": empId }` mapping |
| `locked` | Set of date strings that are holiday pre-assignments |
| `lm` | Set of `"YYYY-MM"` strings for locked (finalized) months |
| `meta` | Derived metadata: targets, allDates, tfDates, postTfDates |
| `emps` | Array of employee objects with availability flags |
| `nmap` | `{ empId: displayName }` override map |
| `hist` | Undo/redo stack managed by `hRed` reducer |

## Scheduling Algorithm (`genSched`)

- Target = `Math.ceil(totalTimeframeDays / poolSize)`; part-time employees get scaled caps via `scaleCap`.
- Pass 1: pre-assign employee-specific holidays from `locked`.
- Pass 2: for each remaining Mon–Thu date, pick the most under-target available employee (random tiebreak); relaxes same-month deduplication and availability constraints only when necessary.
- `calcTgts(emps, tfCount, mo)` computes per-employee targets accounting for availability and caps.

## Holiday System

`buildHmap(by)` returns a map of `YYYY-MM-DD → {type, label}` covering:
- **Federal**: fixed-date and computed (e.g., "last Monday of May" for Memorial Day via `lWD`).
- **Religious/Cultural**: Diwali, Hanukkah, Eid, Passover, Halloween.
- **Assigned**: per-employee holidays stored in `emps[i].holidays`.

## Data Persistence

- No automatic saving. Users manually export/import via:
  - **Save JSON / Load JSON** — full app state round-trip.
  - **Export CSV / Import CSV** — employee availability table only.
- The archive covers 2 years back (`CFG.archiveDepth`); those months are read-only in the UI but editable in the data.

## Patterns to Follow

- All date keys use `fd(d)` → `"YYYY-MM-DD"`.
- Employee IDs are strings like `"e1"`, `"e2"`, …; `empNum(id)` extracts the integer.
- When adding features, keep everything inside `index.html` — do not introduce a build step or external files.
- State mutations happen through React `set*` calls or the `dispatch` from `hRed`; never mutate state objects directly (use `deepCopy` for clones).

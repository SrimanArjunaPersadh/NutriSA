# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

SRIMAN.CUTS is a single-page nutrition & weight management web app. The entire app lives in **`index.html`** — one file containing all HTML, CSS, and JavaScript (~2000+ lines). There is no build step, no bundler, no framework.

## Development

Open `index.html` directly in a browser, or serve it with any static file server:

```powershell
# Simplest option - just open the file
start index.html

# Or serve via Python
python -m http.server 8080

# Or via Node
npx serve .
```

There are no lint, build, or test commands — the project has no `package.json`.

## Architecture

### Single-File Structure

All code is in `index.html`. Key sections within the file:
- **CSS** — dark theme, blue (`#0066FF`) accents, Barlow/Barlow Condensed fonts, mobile-responsive
- **Food Database (`foodDB`)** — 75+ foods with per-100g or per-unit macros; `flagged` foods trigger an off-plan warning
- **Supabase init** — credentials at the top of the script block
- **localStorage helpers** — `loadLocal(key)` / `saveLocal(key, data)` — keys: `sc_w`, `sc_m`, `sc_cu`, `sc_wa`
- **Supabase sync functions** — `cloudAddWeight`, `cloudAddMeal`, `cloudAddWater`, `cloudSyncAll`
- **View renderers** — `renderDashboard()`, `renderNutrition()`, `renderWeight()`, `renderLibrary()`
- **Navigation** — single `showView(v)` toggles between views (`vD`, `vN`, `vW`, `vLib`)

### Data Flow

```
User action
  → update in-memory arrays (wLog, mLog, cuMeals, waLog)
  → saveLocal(key, data)          ← immediate, offline-safe
  → cloudAdd*(data)               ← async, fires and forgets
  → re-render affected view
```

On app load: `cloudSyncAll()` fetches all four Supabase tables and merges with localStorage (cloud wins on conflict by ID).

### Four Supabase Tables

| Table | Local cache key | In-memory var |
|-------|----------------|---------------|
| `weight_logs` | `sc_w` | `wLog` |
| `meal_logs` | `sc_m` | `mLog` |
| `custom_meals` | `sc_cu` | `cuMeals` |
| `water_logs` | `sc_wa` | `waLog` |

### Meal Object Shape

Logged meals (`mLog` entries) carry a `_ings` array for ingredient-level macro editing. The `_libId` field links back to a `cuMeals` entry. Both fields must be preserved through sync — race conditions around `_ings` have been fixed before (see commits).

### Weight Trending

Trend weight uses exponential smoothing (α = 0.1, MacroFactor algorithm). The dashboard computes a 7-day rate of change and projects an ETA to goal weight. Weekly averages follow ISO calendar weeks.

### Drag-and-Drop (Nutrition view)

Library meals can be dragged onto the meal log. The drop handler calls `cloudAddMeal` and must include both `_libId` and `_ings` — omitting them causes ingredient data loss.

## Git / GitHub

**Never automatically commit or push to GitHub.** Always wait for explicit instruction from Sriman before running any `git commit`, `git push`, or related commands.

## Key Conventions

- **No frameworks** — plain `document.getElementById`, `innerHTML`, event listeners.
- **Debounce live macro inputs** — use the existing debounce wrapper before firing recalculations on ingredient weight changes.
- **Flagged foods** — set `flagged: true` in `foodDB` to show a warning; used for off-plan fats (olive oil, coconut oil).
- **Seitan-day detection** — water logging has a special path when seitan is logged that day.
- **Sync status indicator** — update the sync badge (`synced` / `syncing` / `offline` / `error`) on every cloud operation.
- **iOS zoom prevention** — meta viewport and font-size rules are intentional; don't remove them.

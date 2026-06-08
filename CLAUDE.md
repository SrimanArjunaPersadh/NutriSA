# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**NutriSA** (formerly SRIMAN.CUTS) is a single-page nutrition & weight management web app. The entire app lives in **`index.html`** — one file containing all HTML, CSS, and JavaScript (~2500+ lines). There is no build step, no bundler, no framework.

GitHub repo: `https://github.com/SrimanArjunaPersadh/NutriSA`

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
- **Math.round() at display time** — always round macros to whole numbers when rendering (not in `calcIng`/`sumIngs`) to avoid cumulative rounding errors.

## UI Components & Patterns

### `renderDatePicker(date, cfg)`
Reusable popup calendar. Accepts a `cfg` object to configure which state keys it uses:
```javascript
renderDatePicker(S.wdDate, {
  dateExpr: 'S.wdDate',
  openKey: 'wdOpen',
  vmKey: 'wdViewMonth',
  dirKey: 'wdDir',
  wrapId: 'wd-wrap'
})
```
Used on both the Nutrition page (`S.mdate`) and the Weight log page (`S.wdDate`). The boot click-outside handler must close both: check `#dp-wrap` and `#wd-wrap`.

### `mac(t)` — Macro Progress Bars
Renders Calories + Protein/Carbs/Fat as labelled progress bars using `.mbar` / `.mbhdr` / `.mbname` / `.mbnums` / `.track` / `.fill` CSS classes. Colors come from CSS vars: `var(--protein)`, `var(--carbs)`, `var(--fats)`. Do not redesign without Sriman's approval — multiple alternatives (ring charts, stat cards) were tried and rejected in favour of this original design.

### Dashboard — Today's Meals
- **Collapsed**: shows meal name + `kcal` only (no macro sub-line)
- **Expanded**: 4-tile macro grid (kcal / protein / carbs / fat) above an ingredient list

### Saved Meals (Library page)
- `.meal-card-name` — font-size 20px, Barlow Condensed 800 italic
- `.mcm` — 14px, `var(--text3)` — macro summary chips (rounded whole numbers)
- "Saved Meals" heading — Barlow Condensed 19px 800 italic

### Delete buttons
`.delbtn`, `.mb-del`, `.ql-del` are all red (`var(--red)`) by default with a subtle border. Do not make them grey.

### CSS Variables (key ones)
```
--blue: #0066FF
--red: #FF3B30
--protein: (blue-ish)
--carbs: (yellow-ish)
--fats: (orange-ish)
--text, --text2, --text3
--bg4, --border
--fh: 'Barlow Condensed' (headings/labels)
```

## Known Issues / Historical Fixes

- **SVG CSS variables**: `stroke="var(--blue)"` does NOT work as an SVG presentation attribute — CSS custom properties don't resolve there. Use `style="stroke:var(--blue)"` inline instead.
- **`_ings` sync race condition**: fixed previously — always include `_libId` and `_ings` in drag-and-drop drop handler.
- **Duplicate string in debounce blocks**: `_debouncedUpdateGCustom` and `resetGCustom` had identical surrounding patterns; always target with enough context to be unique when editing.

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

**steady** is a nutrition / body-recomposition tracker published as a Claude **artifact**. The entire app is one self-contained file, [steady.html](steady.html) (~5100 lines of plain HTML/CSS/JS). There is no build step, package manager, linter, or test suite — so there are no build/lint/test commands. To try a change, open `steady.html` in a browser (storage/AI/download features only fully work when published as an artifact, see "Artifact runtime bridge"), or republish it with the Artifact tool.

Other contents: [docs/](docs/) (README, deep technical doc, score documentation, case study, license) and `aino_avatar/` (avatar image iterations for the AI nutritionist "Aino"). All rights reserved; source is view-only (license notice also sits in a comment atop `steady.html` — keep it).

Read [docs/steady-technical-documentation.md](docs/steady-technical-documentation.md) before non-trivial changes: it has the full storage schema (§3), calculations (§4), per-feature design rationale (§5), and a function map (§8). [docs/meal_and_day_ratings_documentation.md](docs/meal_and_day_ratings_documentation.md) covers the 1–10 score prompts.

## Architecture

- **One IIFE, banner-commented sections** top to bottom: constants → `state` → storage I/O → domain calculations → UI chrome + `render()` dispatcher → screen renderers → AI nutritionist → Trends → data management → utilities → `init()`.
- **No framework / no virtual DOM.** Each `render*()` builds an HTML string, assigns `.innerHTML`, then re-attaches listeners. A single mutable `state` object holds everything; nothing is reactive, so every mutation must be followed by an explicit `render()` or targeted `render<Section>()`.
- **Routing** is only `state.view` (`profile`, `trends`, `favorites`, default Today/Log) in the `render()` dispatcher; no URL/history.
- **No charting lib, no icon fonts/emoji:** charts and all icons are hand-drawn inline SVG. For circular icon buttons, bake the circle into the SVG itself rather than CSS `background`/`border-radius` (off-center rendering bugs; the loading spinner is a separate self-contained element).

### Artifact runtime bridge
The platform gives no `window.storage`, direct Claude API, or `<a download>`. A block near the top of the script (`getBridge()`, `kvGet/kvSet/kvList/kvDelete`, `aiFetch`, `saveCsv`) maps these onto artifact capabilities `db`, `user`, `sample` (with images), `downloads`. The rest of the app uses `storageGet/Set/List/Delete`, `aiFetch`, `saveCsv`.
- Each storage key is one doc at `data/users/<userId>/<escaped key>` with body `{k, v}` (`v` = JSON string); private per viewer.
- `aiFetch` returns a fetch-shaped object; the `system` prompt is sent as a leading user turn since the platform has no system prompt.
- Native `confirm()` doesn't work in the viewer — use the app's `confirmDialog`.

### Storage keys (see doc §3)
Per-day keys `meals:`, `activity:`, `context:`, `note:`, `chat:<YYYY-MM-DD>`; global single keys `profile`, `dailyTotals` (cache of per-day totals/targets, maintained by `saveMeals`/`saveActivities`, backfilled by Trends/CSV via `ensureDailyTotalsCached`), `favorites`, `measurements`, `customActivities`, `customContextTags`, `__healthcheck`. Non-today `chat:` keys are deleted on every load. Legacy keys (`daily-index*`, `weekchat:*`, `weeksummary`) are no longer written but still wiped by "Delete all data".

## Conventions that matter when editing

- **Changing a data shape:** add a normalizing branch in the matching `load*()` (e.g. `activity:` has had three shapes, `note:` two) — never assume one shape, never rewrite on read.
- **New per-day data:** follow the `meals:<date>` pattern and load it in both `switchDate()` and `init()`. **New global data:** one key, loaded in `init()`.
- **New AI features:** go through `callCoachText` / `callCoachChat` (consistent error surfacing), not `aiFetch` directly. For scores append `SCORE_INSTRUCTION` and parse with `parseScoreFromResponse`.
- **Async generation that writes to storage:** capture target date/view/array at call start, not when the response arrives (race condition). Track loading state by ownership id/date (see `tipLoadingId`, `chatRequestId`, `dailyNoteLoadingDate`), cleared in `finally` gated on ownership — not a bare boolean or "still on same view".
- **Demo data** (`loadDemoData`, `getDemoOffsetData`) is flagged via `profile.isDemo`; cache updates are demo-aware.
- Meal photos are never persisted; chat photos live in memory for one message only. Keep it that way.
- Browser caching: no cache-busting, so hard-refresh to verify changes.

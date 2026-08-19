# steady — Technical Documentation

A single-file, client-side nutrition and body-recomposition tracker built as a Claude artifact. No build step, no framework, no backend beyond Anthropic's artifact storage and API proxy.

---

## 1. Overview

- **File**: one self-contained `.html` file (~2,440 lines) — inline `<style>`, inline `<script>`, no external dependencies except Google Fonts (Fraunces, Inter, JetBrains Mono).
- **Runtime**: vanilla JavaScript, IIFE-wrapped, manual DOM string-templating (`innerHTML` re-renders, no virtual DOM).
- **Persistence**: Anthropic's `window.storage` API — key/value, text/JSON only, personal (per-user, non-shared), active only once the artifact is **published**.
- **AI**: calls `https://api.anthropic.com/v1/messages` directly from the browser (no API key required — handled by the platform), model `claude-sonnet-4-6`. Billed against the user's own Claude usage limits, not a separate cost.
- **No backend, no database** — `window.storage` *is* the persistence layer.

---

## 2. Architecture

The whole app is one IIFE:

```js
(function(){
  // constants (ACTIVITY_KCAL_PER_MIN, MACRO_META, GOAL_LABELS, ...)
  // state object (single mutable object, no reactivity framework)
  // storage helper functions (load*/save* per data type)
  // pure calculation functions (computeTargets, goalAdjustment, ...)
  // render functions (one per screen/section)
  // event handler / action functions (mutate state → persist → re-render)
  // init()
})();
```

### Rendering model
No virtual DOM, no diffing. Each screen has a `render*()` function that builds an HTML string and assigns it to `.innerHTML`, then re-attaches event listeners via `document.getElementById`/`querySelectorAll`. The top-level dispatcher:

```js
function render(){
  updateStorageBanner();
  if (!state.profile) { renderSetup(); return; }
  if (state.view === 'profile')   { renderProfileForm(state.profile); return; }
  if (state.view === 'trends')    { renderTrends(); return; }
  if (state.view === 'favorites') { renderFavoritesPage(); return; }
  renderLog(); // default: Today view
}
```

`state.view` is the only router; there's no URL/history integration (artifacts can't manage browser history reliably).

### State object
A single mutable `state` object holds everything in memory: current profile, current date's meals/activities/chat, favorites, measurements, UI flags (modal open, editing IDs, loading flags). Nothing is reactive — every mutation is followed by an explicit `render()` or a targeted `render<Section>()` call.

---

## 3. Storage schema

All keys use `shared: false` (personal, private per user). None are shared.

| Key | Shape | Notes |
|---|---|---|
| `profile` | `{ nickname?, unit, sex, age, weight, height, activity, trainingsPerWeek, goal }` | One record, no history. `nickname` is optional — when set, it's woven into AI prompts (§5.4) so the nutritionist can address the person by name |
| `meals:<YYYY-MM-DD>` | `[{ id, name, calories, protein_g, carbs_g, fat_g, time, source, coachNote?, favorited?, favoriteRef? }]` | One key per day |
| `activity:<YYYY-MM-DD>` | `[{ type, minutes }]` | Current format. See §3.1 for legacy formats still read for backward compatibility |
| `note:<YYYY-MM-DD>` | `{ text, mealCount, calories }` | Daily AI summary + a snapshot signature used to detect staleness (§5.4) |
| `chat:<YYYY-MM-DD>` | `[{ role: 'user'\|'coach', text, time }]` | Today's AI nutritionist conversation only — past days never get a chat thread |
| `weekchat:<YYYY-MM-DD>` | `[{ role, text, time }]` | Keyed by *today's* date; effectively resets each new day since the 7-day window rolls |
| `favorites` | `[{ id, name, calories, protein_g, carbs_g, fat_g, ingredients }]` | Single key, all favorites |
| `measurements` | `{ "<date>": { waist?, bust?, hips? } }` | Single key, all entries, values in cm |
| `__healthcheck` | probe string | Written/read once on load to detect whether storage is actually working (§6) |

**Legacy keys** (`daily-index`, `daily-index-built`, `weeknote:<date>`) are no longer written but are still explicitly deleted by "Delete all data" for users who had them from earlier versions of the app.

### 3.1 Backward compatibility
`activity:<date>` has gone through three shapes over the app's development:
1. Oldest: single string (`'rest' | 'light' | 'strength' | 'cardio'`)
2. Middle: array of type strings (`['strength', 'walk']`)
3. Current: array of `{ type, minutes }` objects

`loadActivities()` detects which shape is stored and normalizes to the current shape on read, applying `ACTIVITY_DEFAULT_MIN[type]` for any entry missing a duration. Nothing is rewritten in storage on read — normalization happens in memory each load.

`note:<date>` also has two shapes: legacy plain-text notes (no staleness snapshot) and current JSON-encoded `{text, mealCount, calories}`. `loadDailyNote()` detects the format by checking whether the raw value starts with `{`.

---

## 4. Core calculations

**BMR** — Mifflin-St Jeor:
```
male:   10×kg + 6.25×cm − 5×age + 5
female: 10×kg + 6.25×cm − 5×age − 161
```

**TDEE** = BMR × baseline activity multiplier (`sedentary` 1.2 → `very_active` 1.9)

**Goal adjustment** (`goalAdjustment(goal, trainingsPerWeek)`):
| Goal | Adjustment |
|---|---|
| `recomp` | −8% if ≥3 trainings/week, else 0% |
| `lose` | −15% |
| `gain` | +8% |
| `maintain` | 0% |

**Daily activity calorie bump** — summed across every logged activity for the day:
```
extraKcal = Σ (minutes × ACTIVITY_KCAL_PER_MIN[type])
```
| Type | kcal/min | Default duration |
|---|---|---|
| strength | 6 | 45 |
| cardio | 10 | 30 |
| walk | 4 | 30 |
| hike | 7 | 60 |
| stretching | 2.5 | 15 |
| yoga | 3 | 30 |

**Macro split**: protein = 1.9 g/kg (+5g if strength was logged that day) → fat = 27% of *base* calories (pre-activity-bump) ÷ 9 → carbs = remaining calories ÷ 4. Activity-bump calories flow entirely into carbs, not fat, on the theory that they fuel the activity rather than being stored.

All of this is one pure function, `computeTargets(profile, activities)`, called fresh everywhere a target is needed (Today, Trends per-day, AI prompt context) — there is no cached/stored "today's target," it's always recomputed.

---

## 5. Feature-by-feature notes

### 5.1 Meal logging
Two input modes, both converge on the same AI call shape (`callEstimate`):
- **Text**: description → `POST /v1/messages` with a system prompt demanding raw JSON output (`{name, calories, protein_g, carbs_g, fat_g, confidence, note}`).
- **Photo**: resized client-side via `<canvas>` (max 1024px, JPEG q=0.82) before sending as a base64 `image` content block — this also strips EXIF metadata as a side effect. Photos are **never persisted**; only the AI's numeric estimate is saved.

Every estimate is shown for user review/edit before being committed to `state.meals` — the AI never writes directly to storage.

### 5.2 Favorites
A meal can be starred (`toggleMealFavorite`), which creates a `favorites` entry and links back via `meal.favoriteRef`. Un-starring removes the link both ways; deleting a favorite from the Favorites page also clears the flag on any meal that referenced it, so the star state can't drift out of sync between the two views.

### 5.3 Trends (4-week view)
`renderTrends()` fetches the last 28 days with `mapWithConcurrency(days, 6, ...)` — a small custom concurrency-limited `Promise.all` wrapper (no library) — parallelizing storage reads to keep load time reasonable without overwhelming the storage API. Charts are hand-drawn inline SVG (`chartBlock()`), not a charting library: bars scaled to `max(target×1.25, actual)`, a target line drawn per-day (since targets vary with logged activity), axis gridlines/labels computed manually.

*(Historical note: an earlier version cached a pre-aggregated daily-totals index to speed this up. It introduced a data-loss bug — a partially-built cache could be mistaken for a complete one — and was fully reverted in favor of the simpler direct-fetch approach documented here.)*

### 5.4 AI nutritionist
Two distinct interaction modes:
- **Today**: a real back-and-forth chat (`callCoachChat`). System prompt is rebuilt **every message** from live state (today's targets, logged meals, totals, goal) — conversation history is sent as normal message turns, but the grounding context is never "stale" because it's regenerated per request rather than stored in the chat log.
- **Past days**: a single static summary (`callCoachText` → `getDailySummary`), not a chat. Includes a staleness check: the note stores a snapshot (`mealCount`, `calories`) at generation time; if the day's current meals no longer match that snapshot (edited/deleted after the fact), a warning renders next to the note.
- **Weekly** (Trends page): same chat pattern as Today, but scoped to the trailing 7 days, keyed by today's date.

All coach/nutritionist prompts share the same "gradual, steady progress, never suggest extreme restriction" framing regardless of entry point.

**Name personalization**: a shared helper, `nameContext()`, prepends a short instruction to every one of the four AI-context builders above (today's chat, weekly chat, past-day summaries, individual meal tips) — but only if `state.profile.nickname` is set. It's phrased as "address them by name where it feels natural" rather than a hard requirement, so the AI isn't forced to use the name in every single reply. If no nickname is set, the function returns an empty string and every prompt behaves exactly as it did before this feature existed — no conditional branching needed at each call site.

### 5.8 Chart tooltips
Both the Trends bar charts (`chartBlock()`) and the body-measurements line chart (`buildMeasurementsChart()`) embed `data-tip` attributes on their SVG shapes, wired up by a shared `wireChartTooltips(container)` helper to a single floating tooltip element (`#chartTooltip`) that's styled and positioned in JS rather than relying on the browser's native (and non-touch-friendly) `<title>` tooltip.

Two details worth knowing if extending this:
- **Hit areas are deliberately oversized and separate from the visible shape.** A bar chart's actual colored `<rect>` can be just 1px tall on a near-zero day, so each bar also gets an invisible full-height `<rect class="chartbar-hit">` sitting behind it (the visible bar has `pointer-events: none` so it doesn't intercept anything); measurement points similarly get an invisible 9px-radius hit circle around the visible 3px dot. This matters far more on mobile, where "hover precisely on a 1px bar" isn't a realistic gesture.
- **Wiring is scoped carefully to avoid duplicate listeners.** `renderMeasurementsCard()` can be called on its own (e.g., after saving/deleting a measurement from the modal) without the rest of the Trends page re-rendering — so it calls `wireChartTooltips(holder)` scoped to just its own container each time it runs, rather than the macro-chart wiring (called once, unscoped, right after the four `chartBlock()` calls are set) re-matching and double-binding listeners to elements that already have them.
- **Works on both hover and tap**: each tooltip target has `mouseenter`/`mouseleave` *and* a `click` handler (which toggles the tooltip and calls `stopPropagation()` so a single document-level `click` listener can dismiss any open tooltip when tapping elsewhere).

### 5.5 Body measurements
Single `measurements` object, not date-prefixed keys (kept as one key deliberately — entries are sparse, so per-date keys would be wasteful). UI is a lightweight card (chips + chart) on Trends with a full CRUD modal (`openMeasurementsModal` / `renderMeasurementsModal`) for add/edit/delete, since the inline card intentionally shows no raw data entry.

### 5.6 CSV export
Two independent exporters:
- **Daily totals** (`exportCSV(fromDate, toDate)`): numeric-only, no escaping needed, supports an arbitrary date range via a picker on the Trends page.
- **Recipes** (`exportRecipesCSV`): includes free-text `ingredients`, so it uses proper RFC 4180 CSV escaping (`csvEscape()`) — quoting fields containing commas/quotes/newlines and doubling internal quotes.

### 5.7 Data management
`onKeepDaysClick(n)` deletes `meals:`, `note:`, `activity:`, `chat:`, `weekchat:` entries older than a cutoff date. `onDeleteAllClick()` does the same with no cutoff, plus wipes `profile`, `favorites`, `measurements`, and legacy index keys, and resets `state.currentDate` back to today (a real bug, found and fixed: without this, deleting everything while viewing a past date left that stale date selected once a new profile was set up). **Deliberate exception**: `measurements` is excluded from the day-based retention trim (small, sparse, worth keeping long-term) but included in the full wipe.

---

## 6. Storage health check
On `init()`, `checkStorageHealth()` writes and immediately reads back a probe value under `__healthcheck`. If the round-trip doesn't match, `state.storageOk = false` and a persistent red banner renders on every screen: *"Nothing you enter is being saved right now."* This exists because `window.storage` fails silently by default (per Anthropic's API — accessing/writing can throw without obvious cause, most commonly because the artifact isn't published), and the app previously had no way to distinguish "code bug" from "storage inactive."

---

## 7. Known limitations

- **20MB total storage cap** per artifact (Anthropic platform limit), text/JSON only.
- **Last-write-wins**, no conflict resolution — concurrent edits from two open tabs/devices can silently overwrite each other.
- **"Today" is the browser's local clock** — no timezone handling; logging near midnight or while traveling can misassign a date.
- **No offline queueing** — a failed save is reported via toast, but there's no retry queue; the user must manually retry.
- **AI estimates are unverified** — macro numbers and coaching text are LLM output, editable but not fact-checked by the app.
- **No multi-device conflict handling for the AI chat threads** — same last-write-wins caveat applies to `chat:`/`weekchat:`.

---

## 8. File map (major function groups)

| Area | Key functions |
|---|---|
| Storage I/O | `loadProfile/saveProfile`, `loadMeals/saveMeals`, `loadActivities/saveActivities`, `loadDailyNote/saveDailyNote`, `loadChatMessages/saveChatMessages`, `loadWeekChatMessages/saveWeekChatMessages`, `loadFavorites/saveFavoritesStorage`, `loadMeasurements/saveMeasurementsStorage` |
| Calculations | `computeTargets`, `goalAdjustment`, `totalsFromMeals`, `activitiesLabel`, `activityKcal`, `nameContext` |
| Rendering — Today | `renderLog`, `macroRow`, `renderMealList`, `renderAddForm`, `renderCoachChat`, `addActivity`/`removeActivity`/`updateActivityDuration` |
| Rendering — Trends | `renderTrends`, `chartBlock`, `renderMeasurementsCard`, `openMeasurementsModal`/`renderMeasurementsModal`/`closeMeasurementsModal`, `buildMeasurementSeries`/`buildMeasurementsChart`, `renderWeekChat`, `showChartTooltip`/`hideChartTooltip`/`wireChartTooltips` |
| Rendering — Favorites | `renderFavoritesPage`, `renderFavList`, `renderFavEditArea`, `toggleMealFavorite`, `exportRecipesCSV` |
| Rendering — Profile/Setup | `renderSetup`, `renderProfileForm`, `renderProfileFormInto` |
| AI calls | `callEstimate` (meal estimation), `callCoachText` (static summaries), `callCoachChat` (chat, daily + weekly), `buildTodayContext`/`buildWeekContext` (context builders feeding both) |
| Data management | `onKeepDaysClick`, `onDeleteAllClick`, `deleteDataByPrefix`, `deleteMeasurementEntry` |
| Utilities | `mapWithConcurrency`, `shiftDate`, `todayStr`, `fmtDateLabel`, `fmtShortDate`, `escapeHtml`/`escapeAttr`, `csvEscape`, `showToast` |

---

## 9. Extending this app

- **New per-day data**: follow the `meals:<date>` / `activity:<date>` pattern — one key per date, loaded in both `switchDate()` and `init()`.
- **New global data**: follow the `favorites` / `measurements` pattern — one key, loaded once in `init()`.
- **New AI feature**: reuse `callCoachText`/`callCoachChat` rather than writing a new `fetch` call — they already handle error surfacing (`errBody.error.message`) consistently.
- **Changing a data shape**: always write a normalizing branch in the corresponding `load*()` function (see §3.1) rather than assuming a single shape — this app has changed its activity data model three times, and every change required this pattern to avoid silently orphaning existing users' data.
- **When touching `switchDate()` or `init()`**: both must reload *every* piece of date-scoped state (meals, daily note, chat, activities). A real regression happened here — an edit to `switchDate()` accidentally dropped its `loadMeals()` call while other reloads were added around it, so the date label updated correctly on navigation but the meal list, totals, and macro bars silently kept showing whichever day was previously loaded. It went unnoticed for a while because everything *looked* like it was working. If you add a new per-day data type, add its reload call to both functions in the same edit, and manually re-verify by navigating across a couple of days afterward rather than assuming the pattern was followed correctly.

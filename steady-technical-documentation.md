# steady — Technical Documentation

A single-file, client-side nutrition and body-recomposition tracker built as a Claude artifact. No build step, no framework, no backend beyond Anthropic's artifact storage and API proxy.

---

## 1. Overview

- **File**: one self-contained `.html` file (~4,500 lines) — inline `<style>`, inline `<script>`, no external dependencies except Google Fonts (Fraunces, Inter, JetBrains Mono).
- **Runtime**: vanilla JavaScript, IIFE-wrapped, manual DOM string-templating (`innerHTML` re-renders, no virtual DOM).
- **Persistence**: Anthropic's `window.storage` API — key/value, text/JSON only, personal (per-user, non-shared), active only once the artifact is **published**.
- **AI**: calls `https://api.anthropic.com/v1/messages` directly from the browser (no API key required — handled by the platform), model `claude-sonnet-4-6`. Billed against the user's own Claude usage limits, not a separate cost.
- **No backend, no database** — `window.storage` *is* the persistence layer.

---

## 2. Architecture

The whole app is one IIFE, organized top-to-bottom into clearly banner-commented sections:

```js
(function(){
  // constants (ACTIVITY_KCAL_PER_MIN, MACRO_META, GOAL_LABELS, SCORE_INSTRUCTION, ...)
  // state object (single mutable object, no reactivity framework)
  // storage I/O — low-level wrappers + per-entity load*/save* functions
  // domain calculations (computeTargets, totalsFromMeals, ...)
  // global UI chrome (banners, Aino bubble) + top-level render dispatcher
  // rendering — setup/onboarding, demo data, profile/settings
  // rendering — Today/Log view (meal logging, meal list, activity, day context)
  // favorites/recipes
  // AI-nutritionist (tips, daily summary, live chat, chat citations, chat photo)
  // Trends page (charts, weekly summary, measurements)
  // data management (retention trim, full wipe)
  // utilities (staleness, time formatting, HTML escaping, toasts)
  // app init
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
A single mutable `state` object holds everything in memory: current profile, current date's meals/activities/chat, favorites, measurements, UI flags (modal open, editing IDs, loading flags, chat citation, pending chat photo). Nothing is reactive — every mutation is followed by an explicit `render()` or a targeted `render<Section>()` call.

### Icon language
All icons are hand-drawn inline SVG (stroke-based, `currentColor`, typically a 20×20 viewBox), not Unicode glyphs, emoji, or icon fonts. This was a deliberate migration during development: several Unicode-glyph icons (a pencil, a plus sign, an exclamation mark, a delete ×) turned out to render inconsistently — off-center or asymmetric — depending on the browser/OS's own font metrics, in a way that couldn't be reliably fixed with padding nudges. SVG paths with known, checkable coordinates don't have this problem. The one remaining exception is the native browser calendar icon inside `<input type="date">` fields, which can't be replaced without giving up native date-picker behavior (see §7).

---

## 3. Storage schema

All keys use `shared: false` (personal, private per user). None are shared.

| Key | Shape | Notes |
|---|---|---|
| `profile` | `{ nickname?, diet?, dietNotes?, unit, sex, age, weight, height, activity, trainingsPerWeek, goal, isDemo? }` | One record, no history |
| `meals:<YYYY-MM-DD>` | `[{ id, name, calories, protein_g, carbs_g, fat_g, time, source, ingredients?, amountValue?, amountUnit?, coachNote?, tipScore?, tipSnapshot?, favorited?, favoriteRef? }]` | One key per day |
| `activity:<YYYY-MM-DD>` | `[{ type, minutes }]` | Current format; see §3.1 for legacy formats still read for backward compatibility |
| `context:<YYYY-MM-DD>` | `[string]` | Day-context tag IDs (e.g. "traveling", "poor sleep") |
| `note:<YYYY-MM-DD>` | `{ text, mealCount, calories, targetCalories, score }` | Static past-day summary + a staleness snapshot + the 1–10 score |
| `chat:<YYYY-MM-DD>` | `[{ role: 'user'\|'coach', text, time, citedMealName?, hasPhoto? }]` | Today's live chat only. Deleted automatically for any date that isn't today, on every app load (see §5.4) |
| `weeksummary` | `{ text, mealCount, calories, score }` | Single key, not date-prefixed — the trailing-7-day static summary, persists until the underlying data changes, not tied to date rollover |
| `favorites` | `[{ id, name, calories, protein_g, carbs_g, fat_g, ingredients, totalYieldG, typicalPortionG, portionUnit }]` | Single key, all favorites |
| `measurements` | `{ "<date>": { waist?, bust?, hips? } }` | Single key, all entries, values in cm |
| `customActivities` | `[{ id, name, kcalPerMin, defaultMin }]` | User-defined activities with an AI-estimated calorie rate |
| `customContextTags` | `[{ id, label }]` | User-defined day-context tags |
| `__healthcheck` | probe string | Written/read once on load to detect whether storage is actually working (§6); included in the full-wipe routine |

**Legacy keys** (`daily-index`, `daily-index-built`, `weekchat:<date>`) are no longer written but are still explicitly deleted by "Delete all data" for users who had them from earlier versions of the app. `weekchat:<date>` was the storage key for the old interactive weekly chat, replaced by the single `weeksummary` key described above (see §5.5).

### 3.1 Backward compatibility
`activity:<date>` has gone through three shapes over the app's development:
1. Oldest: single string (`'rest' | 'light' | 'strength' | 'cardio'`)
2. Middle: array of type strings (`['strength', 'walk']`)
3. Current: array of `{ type, minutes }` objects

`loadActivities()` detects which shape is stored and normalizes to the current shape on read, applying `ACTIVITY_DEFAULT_MIN[type]` for any entry missing a duration. Nothing is rewritten in storage on read — normalization happens in memory each load.

`note:<date>` also has two shapes: legacy plain-text notes (no staleness snapshot, no score) and current JSON-encoded `{text, mealCount, calories, targetCalories, score}`. `loadDailyNote()` detects the format by checking whether the raw value starts with `{`.

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
Built-in presets:
| Type | kcal/min | Default duration |
|---|---|---|
| walk | 4 | 30 |
| hike | 7 | 60 |
| cardio | 10 | 30 |
| strength | 6 | 45 |
| yoga | 3 | 30 |

(An earlier version also included a `stretching` preset; it was removed to keep the built-in list to five, on the reasoning that it was the least commonly used and overlapped with yoga's intensity. Custom activities — with an AI-estimated kcal/min rate — cover anything not in this list.)

**Macro split**: protein = 1.9 g/kg (+5g if strength was logged that day) → fat = 27% of *base* calories (pre-activity-bump) ÷ 9 → carbs = remaining calories ÷ 4. Activity-bump calories flow entirely into carbs, not fat, on the theory that they fuel the activity rather than being stored.

All of this is one pure function, `computeTargets(profile, activities)`, called fresh everywhere a target is needed (Today, Trends per-day, AI prompt context) — there is no cached/stored "today's target," it's always recomputed.

---

## 5. Feature-by-feature notes

### 5.1 Meal logging
Two input modes, both converge on the same AI call shape (`callEstimate`):
- **Text**: description → `POST /v1/messages` with a system prompt demanding raw JSON output (`{name, calories, protein_g, carbs_g, fat_g, confidence, note}`).
- **Photo**: two entry points — a "Take Photo" button (hidden `<input type="file" capture="environment">`) and a "Choose from Library" button (hidden plain `<input type="file">`), rather than a single combined file input. This split exists because a single plain file input's built-in "let the OS offer both camera and gallery" behavior has a confirmed regression on Android 14/15 in Chrome and Edge specifically, where the camera option silently disappears. Two explicit inputs sidestep that regression on every platform. The photo itself is resized client-side via `<canvas>` (max 1024px, JPEG q=0.82) before sending as a base64 `image` content block — this also strips EXIF metadata as a side effect. Photos are **never persisted**; only the AI's numeric estimate and a short text description are saved.

Every estimate is shown for user review/edit before being committed to `state.meals` — the AI never writes directly to storage.

### 5.2 Favorites
A meal can be starred (`toggleMealFavorite`), which creates a `favorites` entry and links back via `meal.favoriteRef`. Un-starring removes the link both ways; deleting a favorite from the Favorites page also clears the flag on any meal that referenced it, so the star state can't drift out of sync between the two views. Re-logging a favorite ("Log again") is a quick-tap action rather than truly one-tap: it opens a small confirmation modal (pre-filled with the recipe's typical portion) rather than logging instantly, so the amount can be adjusted first.

### 5.3 Trends (4-week view)
`renderTrends()` fetches the last 28 days with `mapWithConcurrency(days, 6, ...)` — a small custom concurrency-limited `Promise.all` wrapper (no library) — parallelizing storage reads to keep load time reasonable without overwhelming the storage API. The day-fetch also retains each day's raw `meals` array (not just totals), which the trailing-7-day summary (§5.5) reuses directly rather than re-fetching. Charts are hand-drawn inline SVG (`chartBlock()`), not a charting library: bars scaled to `max(target×1.25, actual)`, a target line drawn per-day (since targets vary with logged activity), axis gridlines/labels computed manually, with a small right margin reserved so the most recent data point's dot isn't clipped at the viewBox edge.

*(Historical note: an earlier version cached a pre-aggregated daily-totals index to speed this up. It introduced a data-loss bug — a partially-built cache could be mistaken for a complete one — and was fully reverted in favor of the simpler direct-fetch approach documented here.)*

### 5.4 AI nutritionist (Aino)
Three distinct interaction modes, sharing the same `SCORE_INSTRUCTION` prompt fragment (see §5.6):

- **Today (live chat)**: a real back-and-forth chat (`callCoachChat`). System prompt is rebuilt **every message** from live state (today's targets, logged meals with timestamps, totals, goal, day-context tags) — conversation history is sent as normal message turns, but the grounding context is never stale because it's regenerated per request rather than stored in the chat log. Meals in the context are explicitly sorted chronologically and the most recent one is labeled, so Aino can correctly answer "what was my last meal" — an earlier version had no timestamps at all in this context, which caused exactly that kind of question to be answered wrong.
  - **Chat citations**: a chat icon on any of today's meal tips lets you ask a follow-up question about that specific meal. Clicking it sets a one-shot citation (just the meal's id, resolved live at send time — never a frozen copy of its name or tip text) and scrolls to the chat. The citation shows as a dismissible chip above the input before sending; once sent, the meal's full context (name, time, macros, ingredients, the tip text) rides along as a hidden instruction for that one message only, and a small "Re: [meal name]" tag is permanently stored on the sent message so it stays visible in history — this is the one piece of "citation" data that *is* persisted, since the message itself already is.
  - **Chat photos**: a paperclip inside the input opens a small menu (Take Photo / Choose from Library, same two-input pattern as §5.1). A selected photo shows as a dismissible "Photo attached" chip before sending. On send, the photo is sent as a real image content block to the API for that one message, but the chat history only ever stores a `hasPhoto: true` boolean — never the image itself, not even for the rest of the session. Aino can give advisory estimates/opinions on an attached photo but has no ability to log anything from this path; the system prompt explicitly redirects "log this" requests to the dedicated meal-logging photo upload.
  - **Reset chat**: a small button below the input clears today's conversation (with confirmation), both in memory and in storage.
  - **Auto-cleanup**: on every app load, any `chat:<date>` entry that isn't today's is deleted (`deleteDataByPrefix('chat:', todayStr())`, run in the background, not awaited). This isn't just tidying — past-day chat data was already functionally inert (never displayed, never referenced for anything, even before this cleanup existed), so this just stops storing something nobody could ever see again anyway.
- **Past days**: a single static summary (`callCoachText` → `getDailySummary`), not a chat. Includes a staleness check: the note stores a snapshot (`mealCount`, `calories`, `targetCalories`) at generation time; if the day's current data no longer matches that snapshot (edited/deleted after the fact), a warning renders above the summary text.
- **Trailing 7 days** (Trends page): also a single static summary (`getWeekSummary`), not a chat — this replaced an earlier interactive weekly-chat implementation entirely (see §5.5 for why and how). Unlike the daily summary, this one is *not* re-generated automatically when the date rolls over; it persists until the underlying 7-day data actually changes, matching a single non-date-prefixed storage key rather than one per day.

All coach/nutritionist prompts share the same "gradual, steady progress, never suggest extreme restriction" framing regardless of entry point, plus shared handling for all-or-nothing language, loss-of-control language (only ever inferred from what the person actually says, never from logged data alone), and alcohol/caffeine safety thresholds (NIAAA/FDA-based, sex-specific).

**Race-condition safety**: every async generation function (`getMealTip`, `getDailySummary`, `getWeekSummary`, `sendChatMessage`) captures its target (date, view, or the specific meals array) *at the moment the request starts*, not when the response arrives, and only touches live UI state if the user hasn't navigated away in the meantime. An earlier version read `state.currentDate` fresh at save time instead, which caused tips and summaries to occasionally vanish — they were being saved under whatever date the user had since navigated to, not the one they were actually generated for.

### 5.5 Why the weekly chat became a static summary
The original weekly chat (Trends page) worked the same way as today's chat — a live conversation, rebuilt context each message — but scoped to the trailing 7 days. It was replaced with a static, cached summary (matching the past-day summary's pattern) for two reasons: the person's actual use case was closer to "give me a periodic check-in" than "have a conversation," and a cached single-shot summary is dramatically cheaper — a 7-day summary costs roughly a quarter of a typical 3-exchange chat conversation, and because it's cached, checking in multiple times without new data costs nothing at all after the first generation, unlike a chat that resets and re-costs every day.

The summary's prompt is deliberately told not to pad itself to cover a fixed checklist of topics — only mention what's genuinely present in the week's data — and to be conservative about calling anything a "trend" from just 7 days of data.

### 5.6 The 1–10 score
Meal tips, daily summaries, and the weekly summary each include a 1–10 score rating how well that meal/day/week supports the person's stated goal, generated via a shared prompt fragment (`SCORE_INSTRUCTION`) appended to each prompt. The model is asked to output `SCORE: N` on its own first line; `parseScoreFromResponse()` strips that line before the text is shown or saved, so the raw instruction never leaks into visible content. A day's score is explicitly *not* an average of its meals' scores, and a week's score is not an average of its days — the prompt asks the model to weigh the whole pattern instead. The score renders as a small badge (Fraunces "Score" label + JetBrains Mono number) between the Aino header and the message text, using `scoreBadgeHtml()`, shared across all three contexts. Old tips/summaries saved before this feature existed simply render without a badge.

### 5.7 Body measurements
Single `measurements` object, not date-prefixed keys (kept as one key deliberately — entries are sparse, so per-date keys would be wasteful). UI is a lightweight card (chips + chart) on Trends with a full CRUD modal (`openMeasurementsModal` / `renderMeasurementsModal`) for add/edit/delete, since the inline card intentionally shows no raw data entry. The modal's date field displays as `DD/MM/YY` (locale-independent, built manually rather than via `toLocaleDateString` — see §7 for why that matters) and uses the same compact `.datepick` styling as the Today page's date picker.

### 5.8 CSV export
Two independent exporters:
- **Daily totals** (`exportCSV(fromDate, toDate)`): numeric-only, no escaping needed, supports an arbitrary date range via a picker on the Trends page.
- **Recipes** (`exportRecipesCSV`): includes free-text `ingredients`, so it uses proper RFC 4180 CSV escaping (`csvEscape()`) — quoting fields containing commas/quotes/newlines and doubling internal quotes.

### 5.9 Data management
`onKeepDaysClick(n)` deletes `meals:`, `note:`, `activity:`, `context:`, `chat:` entries older than a cutoff date. `onDeleteAllClick()` does the same with no cutoff, plus wipes `profile`, `favorites`, `measurements`, `customActivities`, `customContextTags`, `weeksummary`, `__healthcheck`, and legacy index keys. **Deliberate exception**: `measurements`, `favorites`, and `weeksummary` are excluded from the day-based retention trim (they're not per-date data, or in `weeksummary`'s case, small and sparse) but included in the full wipe.

### 5.10 In-app documentation ("User guidelines")
A global accordion, placed next to "Data & privacy" (both reachable from every screen, not just the Profile page). Structured as six independently-collapsible topics: What steady is for, Who is Aino, How Aino works, How targets work, The 1–10 score, Best practices, Troubleshooting. Aino's own chat system prompt is told to redirect broader app-mechanics questions here by name, while still answering directly anything about her own current output (a score, a tip she just gave).

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
- **Photo file inputs inside the Claude mobile app's built-in webview** may not reliably open — confirmed working in a regular mobile browser and on desktop; this is a platform/webview limitation, not something the app's code controls.
- **Custom web fonts (Fraunces) may fail to load inside the same embedded webview** — a Google Fonts `@import` is a live network request, and that webview appears to have some restriction on external resource loading that a regular browser doesn't. Every Fraunces declaration includes a Georgia fallback specifically for this case, so headers degrade to a real (if different) serif rather than a generic one.
- **`toLocaleDateString`/`toLocaleTimeString` are deliberately avoided** for anything functionally important (sorting, storage keys) after a real production bug: Finnish-locale time strings omit AM/PM, which silently broke a regex expecting it, causing meals to sort to the bottom of the list. All internal date/time formatting is now built manually from `Date` object fields (`nowTimeString()`, `fmtDateWithYear()`, etc.) rather than relying on locale-dependent browser formatting.

---

## 8. File map (major function groups)

| Area | Key functions |
|---|---|
| Storage I/O | `loadProfile/saveProfile`, `loadMeals/saveMeals`, `loadActivities/saveActivities`, `loadDailyNote/saveDailyNote`, `loadChatMessages/saveChatMessages`, `loadWeekSummary/saveWeekSummary`, `loadFavorites/saveFavoritesStorage`, `loadMeasurements/saveMeasurementsStorage`, `loadDayContext/saveDayContext` |
| Calculations | `computeTargets`, `goalAdjustment`, `totalsFromMeals`, `activitiesLabel`, `activityKcal` |
| Rendering — Today | `renderLog`, `macroRow`, `renderMealList`, `renderAddForm`, `renderCoachChat`, `tipBulbIcon` |
| Rendering — Trends | `renderTrends`, `chartBlock`, `renderMeasurementsCard`, `renderMeasurementsModal`, `renderWeekSummaryCard`, `measurementChartBlock` |
| Rendering — Favorites | `renderFavoritesPage`, `renderFavList`, `renderFavEditArea` |
| Rendering — Profile/Setup | `renderSetup`, `renderProfileForm`, `renderProfileFormInto` |
| AI calls | `callEstimate` (meal estimation), `callCoachText` (static summaries), `callCoachChat` (live chat, with citation/photo support) |
| Score | `parseScoreFromResponse`, `scoreBadgeHtml`, `SCORE_INSTRUCTION` |
| Chat citations/photos | `citeTipInChat`, `buildCitationInstruction`, `onChatPhotoSelect`, `resetChat` |
| Data management | `onKeepDaysClick`, `onDeleteAllClick`, `deleteDataByPrefix` |
| Utilities | `mapWithConcurrency`, `shiftDate`, `todayStr`, `fmtDateLabel`, `fmtShortDate`, `fmtDateWithYear`, `nowTimeString`, `timeStringTo24h`/`time24hToTimeString`, `escapeHtml`/`escapeAttr`, `csvEscape`, `showToast`, `resizeImageDataUrl` |

---

## 9. Extending this app

- **New per-day data**: follow the `meals:<date>` / `activity:<date>` pattern — one key per date, loaded in both `switchDate()` and `init()`.
- **New global data**: follow the `favorites` / `measurements` / `weeksummary` pattern — one key, loaded once in `init()` (or lazily, for `weeksummary`).
- **New AI feature**: reuse `callCoachText`/`callCoachChat` rather than writing a new `fetch` call — they already handle error surfacing (`errBody.error.message`) consistently. If it needs a score, append `SCORE_INSTRUCTION` to the prompt and parse with `parseScoreFromResponse`.
- **Changing a data shape**: always write a normalizing branch in the corresponding `load*()` function (see §3.1) rather than assuming a single shape — this app has changed its activity data model three times, and every change required this pattern to avoid silently orphaning existing users' data.
- **New icons**: use hand-drawn SVG, not Unicode glyphs or emoji — see §2 for why.
- **New async generation function that writes to storage**: capture the target (date/view/array) at the start of the function, not read it fresh when the response arrives — see the race-condition note in §5.4.

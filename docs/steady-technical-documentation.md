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
  // Trends page (charts, daily-totals caching, measurements)
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
All icons are hand-drawn inline SVG (stroke-based, typically a 20×20 viewBox), not Unicode glyphs, emoji, or icon fonts. This was a deliberate migration during development: several Unicode-glyph icons (a pencil, a plus sign, an exclamation mark, a delete ×) turned out to render inconsistently — off-center or asymmetric — depending on the browser/OS's own font metrics, in a way that couldn't be reliably fixed with padding nudges. SVG paths with known, checkable coordinates don't have this problem. The one remaining exception is the native browser calendar icon inside `<input type="date">` fields, which can't be replaced without giving up native date-picker behavior (see §7).

A second, more subtle icon-reliability issue surfaced later: every circular icon button (`.icon-btn`, `.medit`/`.mdel`, `.addactbtn`, `.tip-refresh`, `.chatsend-icon`, `.iconnav-circle`) originally drew its green circle via CSS (`background` + `border-radius: 50%`) with a smaller icon SVG centered inside via flexbox. Several of those inner icons were odd-numbered pixel sizes (11/13/15px) inside even-numbered containers (26/32/38px) — centering an odd child in an even container produces a half-pixel offset that browsers round inconsistently depending on zoom level (correct-looking at 150%, visibly shifted at 100% — that discrepancy was the actual diagnostic signal that led to the root cause). The fix was architectural, not a size patch: every icon's circle is now drawn *inside its own SVG*, in the same coordinate system as the icon itself (`fill="#628264"` for the circle, `stroke="#FFFFFF"`/`fill="#FFFFFF"` for the icon — not `currentColor`, since the SVG no longer inherits a background from its parent button), so there's no longer a separate CSS-centering step that can disagree with the icon's own geometry. The circular buttons' own CSS had their `background`/`border-radius` removed accordingly. Each icon's `viewBox` is deliberately expanded around its content (rather than tightly fit) so the icon reads as clearly smaller than its circle, matching the original CSS-centered proportions, not filling the whole circle.

One follow-on regression from that refactor is worth noting for anyone touching these buttons again: the loading-spinner state (`.icon-spinner`, used on "Get tip" and the AI-estimate buttons) is a separate CSS border-ring element, not part of any icon SVG — it had been implicitly relying on the button's now-removed CSS background to be visible against. It's now self-contained with its own circular backdrop, matching the same "don't depend on the parent's background" principle as the icon fix itself.

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
| `dailyTotals` | `{ "<date>": { calories, protein, carbs, fat, targetCalories, targetProteinG, targetCarbsG, targetFatG } }` | Single key, all dates — a cache of each day's computed totals/targets, populated incrementally as `saveMeals`/`saveActivities` are called for any date, plus backfilled by Trends/CSV export for any date it's missing. See §5.3 for the full design and why it exists. |
| `favorites` | `[{ id, name, calories, protein_g, carbs_g, fat_g, ingredients, totalYieldG, typicalPortionG, portionUnit }]` | Single key, all favorites |
| `measurements` | `{ "<date>": { waist?, bust?, hips? } }` | Single key, all entries, values in cm |
| `customActivities` | `[{ id, name, kcalPerMin, defaultMin }]` | User-defined activities with an AI-estimated calorie rate |
| `customContextTags` | `[{ id, label }]` | User-defined day-context tags |
| `__healthcheck` | probe string | Written/read once on load to detect whether storage is actually working (§6); included in the full-wipe routine |

**Legacy keys** (`daily-index`, `daily-index-built`, `weekchat:<date>`, `weeksummary`) are no longer written but are still explicitly deleted by "Delete all data" for users who had them from earlier versions of the app. `weekchat:<date>` was the storage key for the old interactive weekly chat; `weeksummary` was its later replacement, a single-key static trailing-7-day summary — that entire feature (weekly chat, then weekly summary) was removed altogether after it was identified as the source of rate-limit errors from excessive simultaneous storage calls on the Trends page (see §5.3).

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

All of this is one pure function, `computeTargets(profile, activities)`, called fresh every time for whichever day is currently open (Today, the day being viewed in the log) and for AI prompt context — there is no cached "today's target." Trends is the deliberate exception: a past day's target, once computed, is cached in `dailyTotals` and kept as-is going forward — a later change to goal, activity, or profile does not retroactively recompute it. See §5.3.

---

## 5. Feature-by-feature notes

### 5.1 Meal logging
Two input modes, both converge on the same AI call shape (`callEstimate`):
- **Text**: description → `POST /v1/messages` with a system prompt demanding raw JSON output (`{name, calories, protein_g, carbs_g, fat_g, confidence, note}`).
- **Photo**: two entry points — a "Take Photo" button (hidden `<input type="file" capture="environment">`) and a "Choose from Library" button (hidden plain `<input type="file">`), rather than a single combined file input. This split exists because a single plain file input's built-in "let the OS offer both camera and gallery" behavior has a confirmed regression on Android 14/15 in Chrome and Edge specifically, where the camera option silently disappears. Two explicit inputs sidestep that regression on every platform. The photo itself is resized client-side via `<canvas>` (max 1024px, JPEG q=0.82) before sending as a base64 `image` content block — this also strips EXIF metadata as a side effect. Photos are **never persisted**; only the AI's numeric estimate and a short text description are saved.

Every estimate is shown for user review/edit before being committed to `state.meals` — the AI never writes directly to storage.

### 5.2 Favorites
A meal can be starred (`toggleMealFavorite`), which creates a `favorites` entry and links back via `meal.favoriteRef`. Un-starring removes the link both ways; deleting a favorite from the Favorites page also clears the flag on any meal that referenced it, so the star state can't drift out of sync between the two views. Re-logging a favorite ("Log again") is a quick-tap action rather than truly one-tap: it opens a small confirmation modal (pre-filled with the recipe's typical portion) rather than logging instantly, so the amount can be adjusted first.

### 5.3 Trends (4-week view) and the `dailyTotals` cache
`renderTrends()` shows the last 28 days. Early versions fetched every day's `meals:`/`activity:` fresh on every visit (`mapWithConcurrency(days, 6, ...)`), which worked but cost ~54-56 storage calls per visit — confirmed in production to trigger rate-limit errors for real users on every single Trends visit, not just a one-time cost. A first caching attempt (a pre-aggregated index, built incrementally) was tried and fully reverted after a partially-built cache could be mistaken for a complete one, silently dropping data — that failure is why this section previously described the plain fetch-everything approach with no caching at all.

The current design (`dailyTotals`, see §3) avoids that failure mode by never distinguishing "partial" from "complete" — every read either finds a real cached value for a date or it doesn't, with no separate completeness flag to get out of sync:

- **Incremental population**: `updateDailyTotalsCache(dateStr, meals, activities)` runs (fire-and-forget, not awaited) at the end of every `saveMeals`/`saveActivities` call, for any date — not just today. Editing a past day updates that day's cache entry immediately, the same as editing today does.
- **Resolution order per day**, in `runDailyTotalsCachePass()`: (1) the currently-viewed day always uses live in-memory `state.meals`/`state.activities`, never the cache, since a just-made edit's fire-and-forget cache write may not have landed yet; (2) a cached entry, if one exists, is used as-is; (3) in demo mode, with no cached entry, the demo formula (§5.11) computes fresh; (4) otherwise, a fresh per-day fetch, batched (see below).
- **Cache takes precedence over the demo formula, not the other way around.** Demo days only fall back to the formula when nothing is cached for them — an edited demo day behaves identically to an edited real day everywhere (Trends included), not just while it happens to be the currently-viewed day. This was a real, fixed bug: the formula used to be checked *before* the cache, so an edited demo day displayed correctly only while actively being viewed and silently reverted to the untouched formula output the moment you navigated elsewhere, even though the edit was still correctly saved.
- **First-time backfill is cheap on purpose**: a single `storageList('meals:')` call up front identifies which uncached dates have any real data at all, so a day with nothing logged costs zero fetches rather than a wasted "not found" round trip. Days that do need fetching pull `meals:` only, using a flat, non-activity-adjusted target rather than also fetching that day's `activity:` — halving the per-day cost for a first-time backfill. If that day is later actually edited, `saveActivities` recomputes its real, activity-adjusted target then, same as any other cache update.
- **Fetches are batched and paced**, not fired all at once — a burst of many simultaneous storage calls is a more aggressive pattern than the same total spread over a couple of seconds, which is what actually triggered the original rate-limit reports.
- **A quiet background warm-up** runs once at app init (fire-and-forget, after the initial render, not blocking startup), so the cache is often already populated by the time the user opens Trends. A module-level guard (`dailyTotalsWarmupInFlight`) makes any later caching pass (a Trends visit, CSV export) wait for an in-flight one rather than run concurrently — two concurrent read-modify-write passes over the same `dailyTotals` key would otherwise race and lose each other's writes.
- **Demo mode's interaction with the cache needed its own fix.** `updateDailyTotalsCache` is called with only whichever of meals/activities was actually just edited — the other is `undefined` and gets resolved by fetching real storage. For a *real* day that's correct; for a demo day where that other piece was never itself edited, no real storage entry exists (the demo formula's content is never persisted unless specifically edited), so the fetch always failed and silently fell back to an empty array — editing only a demo day's activity zeroed out its cached meal calories, and editing only its meals zeroed out its cached activity/target. Fixed by falling back to that date's actual demo-formula value for whichever piece has no real entry, rather than treating it as empty.
- **CSV export** (`exportCSV`) reuses the same `ensureDailyTotalsCached()` pass Trends uses, rather than its own fetch logic — an explicit date range builds its own day list; no range given falls back to every date with either a cached entry or a real `meals:` log (via the same `storageList` check), not capped at 28 days.

Charts are hand-drawn inline SVG (`chartBlock()`), not a charting library: bars scaled to `max(target×1.25, actual)`, a target line drawn per-day (since targets vary with logged activity), axis gridlines/labels computed manually, with a small right margin reserved so the most recent data point's dot isn't clipped at the viewBox edge.

Chart axis labels have their own scaling note: each chart SVG uses `width:100%` (responsive) with a `viewBox` computed at render time from the actual expected card width (`min(window.innerWidth, 640) - 40px` padding), not a single fixed value. An earlier fixed viewBox (380 units, sized for mobile) meant `font-size` values — specified in viewBox units, not real pixels — scaled up by roughly 1.5x on a wide desktop window where the card renders far wider than 380px, visibly inflating axis label text there while looking correct on mobile. Computing the viewBox from the actual render width keeps the scale factor close to 1:1 regardless of screen width.

### 5.4 AI nutritionist (Aino)
Three distinct interaction modes, sharing the same `SCORE_INSTRUCTION` prompt fragment (see §5.6):

- **Meal tips** (`getMealTip`, per meal): a short 1-10 scored tip on an individual logged meal, shown inline on the meal row. `state.tipLoadingId` tracks which meal is currently generating — always cleared in a `finally` block only if it still matches the meal that set it (`state.tipLoadingId === id`), so a faster-finishing earlier request can't wrongly clear a newer, still-in-flight one's loading spinner if two tips are triggered close together.
- **Today (live chat)**: a real back-and-forth chat (`callCoachChat`). System prompt is rebuilt **every message** from live state (today's targets, logged meals with timestamps, totals, goal, day-context tags) — conversation history is sent as normal message turns, but the grounding context is never stale because it's regenerated per request rather than stored in the chat log. Meals in the context are explicitly sorted chronologically and the most recent one is labeled, so Aino can correctly answer "what was my last meal" — an earlier version had no timestamps at all in this context, which caused exactly that kind of question to be answered wrong.
  - **Chat citations**: a chat icon on any of today's meal tips lets you ask a follow-up question about that specific meal. Clicking it sets a one-shot citation (just the meal's id, resolved live at send time — never a frozen copy of its name or tip text) and scrolls to the chat. The citation shows as a dismissible chip above the input before sending; once sent, the meal's full context (name, time, macros, ingredients, the tip text) rides along as a hidden instruction for that one message only, and a small "Re: [meal name]" tag is permanently stored on the sent message so it stays visible in history — this is the one piece of "citation" data that *is* persisted, since the message itself already is.
  - **Chat photos**: a paperclip inside the input opens a small menu (Take Photo / Choose from Library, same two-input pattern as §5.1). A selected photo shows as a dismissible "Photo attached" chip before sending. On send, the photo is sent as a real image content block to the API for that one message, but the chat history only ever stores a `hasPhoto: true` boolean — never the image itself, not even for the rest of the session. Aino can give advisory estimates/opinions on an attached photo but has no ability to log anything from this path; the system prompt explicitly redirects "log this" requests to the dedicated meal-logging photo upload.
  - **Reset chat**: a small button below the input clears today's conversation (with confirmation), both in memory and in storage.
  - **Auto-cleanup**: on every app load, any `chat:<date>` entry that isn't today's is deleted (`deleteDataByPrefix('chat:', todayStr())`, run in the background, not awaited). This isn't just tidying — past-day chat data was already functionally inert (never displayed, never referenced for anything, even before this cleanup existed), so this just stops storing something nobody could ever see again anyway.
  - **Loading-state ownership**: `state.chatLoading` is cleared in a `finally` block gated on the request still being the active one (`myRequestId === state.chatRequestId`) — independent of whether the user has since navigated to a different day. It used to also require `state.currentDate === targetDate` to clear, which meant navigating away from today before a reply arrived orphaned the flag at `true` permanently (nothing else ever reset it), leaving the send button stuck showing "Stop" indefinitely on return, despite the reply having already arrived and saved correctly.
- **Past days**: a single static summary (`callCoachText` → `getDailySummary`), not a chat. Includes a staleness check: the note stores a snapshot (`mealCount`, `calories`, `targetCalories`) at generation time; if the day's current data no longer matches that snapshot (edited/deleted after the fact), a warning renders above the summary text. Its loading state (`state.dailyNoteLoadingDate`, holding the date currently generating rather than a bare boolean) has the same ownership-based clearing as chat's, for the same reason — it used to get stuck on "Thinking it over…" forever if you navigated away mid-generation.

All coach/nutritionist prompts share the same "gradual, steady progress, never suggest extreme restriction" framing regardless of entry point, plus shared handling for all-or-nothing language, loss-of-control language (only ever inferred from what the person actually says, never from logged data alone), and alcohol/caffeine safety thresholds (NIAAA/FDA-based, sex-specific).

**Race-condition safety**: every async generation function (`getMealTip`, `getDailySummary`, `sendChatMessage`) captures its target (date, view, or the specific meals array) *at the moment the request starts*, not when the response arrives, and only touches live UI state if the user hasn't navigated away in the meantime. An earlier version read `state.currentDate` fresh at save time instead, which caused tips and summaries to occasionally vanish — they were being saved under whatever date the user had since navigated to, not the one they were actually generated for.

### 5.5 Why the weekly summary feature was removed entirely
The Trends page originally had a fourth AI surface alongside meal tips, today's chat, and the past-day summary: a trailing-7-day check-in, first as a live chat (matching today's chat, but scoped to 7 days), later replaced with a static cached summary for cost reasons (a periodic check-in fit the actual use case better than a conversation, and a cached single-shot summary is far cheaper than a chat that re-costs on every exchange).

It was removed altogether, not just changed form again, after being identified as the source of confirmed rate-limit errors — every Trends visit was making the summary's own request in addition to the ~54-56 storage calls the page's data-fetching cost at the time (see §5.3), and removing the feature was the fix actually shipped, ahead of the later, more targeted fix to the data-fetching itself. Its storage keys (`weekchat:<date>`, then `weeksummary`) are documented as legacy in §3 — no longer written, but still cleaned up by "Delete all data" for anyone who has them from before the removal.

### 5.6 The 1–10 score
Meal tips and daily summaries each include a 1–10 score rating how well that meal/day supports the person's stated goal, generated via a shared prompt fragment (`SCORE_INSTRUCTION`) appended to each prompt. The model is asked to output `SCORE: N` on its own first line; `parseScoreFromResponse()` strips that line before the text is shown or saved, so the raw instruction never leaks into visible content. A day's score is explicitly *not* an average of its meals' scores — the prompt asks the model to weigh the whole pattern instead. The score renders as a small badge (Fraunces "Score" label + JetBrains Mono number) between the Aino header and the message text, using `scoreBadgeHtml()`, shared across both contexts. Old tips/summaries saved before this feature existed simply render without a badge.

### 5.7 Body measurements
Single `measurements` object, not date-prefixed keys (kept as one key deliberately — entries are sparse, so per-date keys would be wasteful). UI is a lightweight card (chips + chart) on Trends with a full CRUD modal (`openMeasurementsModal` / `renderMeasurementsModal`) for add/edit/delete, since the inline card intentionally shows no raw data entry. The modal's date field displays as `DD/MM/YY` (locale-independent, built manually rather than via `toLocaleDateString` — see §7 for why that matters) and uses the same compact `.datepick` styling as the Today page's date picker. CSV export (with its own date range) is a separate modal — see §5.8.

### 5.8 CSV export
Three independent exporters, two of which share a date-range picker:
- **Daily totals** (`exportCSV(fromDate, toDate)`): numeric-only, no escaping needed.
- **Body measurements** (`exportMeasurementsCSV(fromDate, toDate)`): same shape as daily totals — numeric-only, date-range filtered.
- **Recipes** (`exportRecipesCSV`): includes free-text `ingredients`, so it uses proper RFC 4180 CSV escaping (`csvEscape()`) — quoting fields containing commas/quotes/newlines and doubling internal quotes. No date range (recipes aren't dated).

**Shared export modal**: `exportCSV` and `exportMeasurementsCSV` are both driven by one popup (`openExportModal(kind)` / `closeExportModal()` / `renderExportModal()`, `kind` is `'macros'` or `'measurements'`, tracked in `state.exportModalFor`) rather than each having its own inline From/To panel. The modal's title and default "From" date adapt to `kind` — 27 days back for daily totals (matching the Trends window), the earliest logged date for measurements — and "Download CSV" calls whichever export function the modal was opened for, then closes itself on success. Follows the same `.modal-overlay`/`.modal-box` pattern as the measurements CRUD modal and the confirm dialog (click outside or the × to close), rather than a bespoke inline pattern.

### 5.9 Data management
`onKeepDaysClick(n)` deletes `meals:`, `note:`, `activity:`, `context:`, `chat:` entries older than a cutoff date, and separately prunes `dailyTotals` down to entries for dates on or after that same cutoff, keeping the summary cache consistent with the detailed data it summarizes. `onDeleteAllClick()` does the same with no cutoff, plus wipes `profile`, `favorites`, `measurements`, `customActivities`, `customContextTags`, `dailyTotals`, `__healthcheck`, and legacy index keys (including the now-removed `weeksummary`, see §5.5). **Deliberate exception**: `measurements` and `favorites` are excluded from the day-based retention trim (they're not per-date data) but included in the full wipe.

`loadDemoData()` performs the same explicit clear across all date-prefixed keys plus `dailyTotals` before seeding fresh demo content (see §5.11) — not just relying on a prior full wipe having already run, since it needs to be safe to call on its own.

### 5.10 In-app documentation ("User guidelines")
A global accordion, placed next to "Data & privacy" (both reachable from every screen, not just the Profile page). Structured as eight independently-collapsible topics: What steady is for, Who is Aino, How Aino works, How targets work, The 1–10 score, Best practices, Troubleshooting, Demo data (added last, after Troubleshooting — covers what persists from an edit, what a full reset does, and how to switch to real data; see §5.11 for the underlying mechanics). Aino's own chat system prompt is told to redirect broader app-mechanics questions here by name, while still answering directly anything about her own current output (a score, a tip she just gave).

### 5.11 Demo data
`getDemoOffsetData(dateStr)` is a pure function, not a stored dataset: it computes a day's meals/activity/context from `diffDays` — how many days before *today* that date currently is — not the literal calendar date, so the same demo content always represents "N days ago" regardless of when the app is actually opened, with no generation or storage step required. It only covers `1 <= diffDays <= 27`; today (`diffDays < 1`) is deliberately excluded.

- **Today is real, persisted data from the start**, not formula output. `loadDemoData()` seeds it with specific meals/activity via the normal `saveMeals`/`saveActivities` calls, so it behaves exactly like a genuine day immediately — the part of the demo a person is expected to actually interact with (add a meal, log activity, see it persist), rather than a formula they'd just be overriding on first touch anyway.
- **Activity follows a 7-entry cycle** (`DEMO_ACTIVITY_CYCLE[diffDays % 7]`) — none, walk, strength, cardio, strength+walk, yoga, hike — so the same activity (and its resulting target) recurs every 7 days by design. This is visible on Trends as a repeating weekly pattern in the target line, not a bug.
- **Over/under-target days are injected deliberately**, not incidentally, so Trends has clear examples of each rather than 27 uniformly on-target bars: every 4th day (`diffDays % 4 === 0`) adds an extra meal sized to land ~22% over target, rotating which metric (calories/protein/carbs) it targets across different multiples of 4; the offset case (`diffDays % 4 === 2`) scales meals down for a clear under-target example; every 5th day (`diffDays % 5 === 2`) adds a smaller protein-boost snack, independent of the over/under logic.
- **Editing a past demo day makes it real data**, indistinguishable from genuine entries at the storage level — there's no separate "demo" namespace or tag. `saveMeals`/`saveActivities` write to the same `meals:<date>`/`activity:<date>` keys either way, which is also why "Delete all data" and "Try demo data instead" (`loadDemoData()`, which explicitly clears all date-prefixed storage before reseeding — see §5.9) correctly remove any demo-day edits with no special-casing needed.
- **The `dailyTotals` cache resolving demo edits correctly required its own fix** — see §5.3's note on `updateDailyTotalsCache`'s per-piece (meals vs. activities) demo fallback.

*(Historical note: an approach was tried where "Delete all data" and "Try demo data instead" would trigger an actual `window.location.reload()` shortly after finishing, to guarantee a clean JS execution state rather than relying on in-place `state` resets to catch everything. It was implemented, verified working, and then explicitly reverted at the person's request in favor of keeping the wipe/reseed fully in-app.)*

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
- **No cache-busting on updates** — this is one HTML file with no versioned filename or build hash, so a browser tab that already has the page loaded (or a normal, non-hard refresh) can keep running old JavaScript indefinitely after the underlying file has been updated, completely independent of whether storage itself is working correctly. This produced real, confusing bug reports during development that looked like storage/logic failures but were actually just stale code still running — a hard refresh (or a fresh tab) is required after any update to guarantee the latest version is actually executing.

---

## 8. File map (major function groups)

| Area | Key functions |
|---|---|
| Storage I/O | `loadProfile/saveProfile`, `loadMeals/saveMeals`, `loadActivities/saveActivities`, `loadDailyNote/saveDailyNote`, `loadChatMessages/saveChatMessages`, `loadFavorites/saveFavoritesStorage`, `loadMeasurements/saveMeasurementsStorage`, `loadDayContext/saveDayContext` |
| `dailyTotals` caching | `ensureDailyTotalsCached` (entry point, with the concurrency guard), `runDailyTotalsCachePass` (the actual per-day resolution + batched fallback + backfill), `updateDailyTotalsCache` (called from `saveMeals`/`saveActivities`, demo-aware) |
| Calculations | `computeTargets`, `goalAdjustment`, `totalsFromMeals`, `activitiesLabel`, `activityKcal` |
| Rendering — Today | `renderLog`, `macroRow`, `renderMealList`, `renderAddForm`, `renderCoachChat`, `tipBulbIcon` |
| Rendering — Trends | `renderTrends`, `chartBlock`, `renderMeasurementsCard`, `renderMeasurementsModal`, `measurementChartBlock`, `openExportModal`/`closeExportModal`/`renderExportModal` |
| Rendering — Favorites | `renderFavoritesPage`, `renderFavList`, `renderFavEditArea` |
| Rendering — Profile/Setup | `renderSetup`, `renderProfileForm`, `renderProfileFormInto` |
| AI calls | `callEstimate` (meal estimation), `callCoachText` (static summaries), `callCoachChat` (live chat, with citation/photo support), `getMealTip`, `getDailySummary` |
| Score | `parseScoreFromResponse`, `scoreBadgeHtml`, `SCORE_INSTRUCTION` |
| Chat citations/photos | `citeTipInChat`, `buildCitationInstruction`, `onChatPhotoSelect`, `resetChat` |
| Demo data | `getDemoOffsetData` (pure, offset-based formula), `loadDemoData` (seeds today for real, clears + reseeds on repeat calls), `DEMO_ACTIVITY_CYCLE` |
| Data management | `onKeepDaysClick`, `onDeleteAllClick`, `deleteDataByPrefix` |
| Utilities | `mapWithConcurrency`, `shiftDate`, `todayStr`, `fmtDateLabel`, `fmtShortDate`, `fmtDateWithYear`, `nowTimeString`, `timeStringTo24h`/`time24hToTimeString`, `escapeHtml`/`escapeAttr`, `csvEscape`, `showToast`, `resizeImageDataUrl` |

---

## 9. Extending this app

- **New per-day data**: follow the `meals:<date>` / `activity:<date>` pattern — one key per date, loaded in both `switchDate()` and `init()`.
- **New global data**: follow the `favorites` / `measurements` / `dailyTotals` pattern — one key, loaded once in `init()` (or lazily, for `dailyTotals`, via `ensureDailyTotalsCached`).
- **New AI feature**: reuse `callCoachText`/`callCoachChat` rather than writing a new `fetch` call — they already handle error surfacing (`errBody.error.message`) consistently. If it needs a score, append `SCORE_INSTRUCTION` to the prompt and parse with `parseScoreFromResponse`.
- **Changing a data shape**: always write a normalizing branch in the corresponding `load*()` function (see §3.1) rather than assuming a single shape — this app has changed its activity data model three times, and every change required this pattern to avoid silently orphaning existing users' data.
- **New icons**: use hand-drawn SVG. For a circular icon button specifically, bake the circle into the same SVG as the icon (not a separate CSS `background`/`border-radius` on the button) — see §2 for why.
- **New async generation function that writes to storage**: capture the target (date/view/array) at the start of the function, not read it fresh when the response arrives — see the race-condition note in §5.4.
- **New async generation function with a loading state**: track ownership (the specific id/date the request is for), not a bare boolean, and clear it in a `finally` block gated on that ownership still holding — not on the user still being on the same view. `tipLoadingId`, `chatRequestId`, and `dailyNoteLoadingDate` all follow this pattern after two related bugs: a bare boolean gated on "same view" got permanently stuck if the user navigated away before the response arrived (nothing else ever reset it), and a bare id/boolean with no ownership check let a faster-finishing request wrongly clear a slower, still-in-flight one's state.

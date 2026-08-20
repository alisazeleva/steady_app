# steady

**A daily ledger for gradual, steady progress.**

A nutrition and body-recomposition tracker built as a Claude artifact — designed, built, and debugged entirely through conversation with Claude, with no manual coding. Log meals by text or photo, get AI-estimated macros, track daily and weekly trends, chat with an AI nutritionist grounded in your actual logged data, and keep it all private to your own account.

[**Try the live app →**](#) *(a published claude.ai link will be added later)*

![steady screenshot](#) *(a screenshot will be added later)*

---

## What it does

- **AI meal logging** — describe a meal in plain text or snap a photo; Claude estimates calories and macros, shown for review before anything is saved
- **Dynamic targets** — calorie/macro goals computed from the Mifflin-St Jeor formula, adjusted for your stated goal (recomposition, fat loss, muscle gain, maintenance), training frequency, and *today's* logged activity — multiple activities, each with a duration, summed into a real calorie adjustment
- **AI nutritionist chat** — a real back-and-forth conversation for today (re-grounded in your live data on every message) and a separate weekly chat scoped to your trailing 7 days; past days get a static, regeneratable summary instead
- **Trends** — 4-week charts with visible axes, per-day target lines that shift with logged activity, and a body-measurements tracker with its own three separate charts (bust, waist, hips), each with its own axis scale
- **Favorite recipes** — save frequent meals with full ingredient lists, searchable, with one-tap re-logging and CSV export
- **Data control** — CSV export for daily totals, recipes, and body measurements; a "keep last N days" retention tool; full-history deletion; and a self-diagnosing storage health check that tells you plainly if something isn't saving instead of failing silently

---

## Tech stack

Plain HTML/CSS/JavaScript — one self-contained file, no build step, no framework, no backend beyond the Claude artifact platform:

- **Persistence**: Anthropic's `window.storage` API (personal, per-user key/value storage, 20MB cap)
- **AI**: direct browser calls to `https://api.anthropic.com/v1/messages`, model `claude-sonnet-4-6` — no API key needed, billed against the user's own Claude usage
- **Rendering**: hand-written `innerHTML` templating, no virtual DOM, no React
- **Charts**: hand-drawn inline SVG, no charting library
- **Fonts**: Fraunces (display serif), Inter (body), JetBrains Mono (numbers) via Google Fonts

See [`steady-technical-documentation.md`](./steady-technical-documentation.md) for the full architecture, storage schema, and calculation reference.

---

## Running this project

This app is built specifically as a **Claude artifact**, not a standalone static site. It depends on two things only available inside that environment:

1. `window.storage` for persistence — this API doesn't exist in a regular browser, so opening the HTML file directly (e.g. via `file://` or a generic static host) will load the UI but **nothing will save**.
2. Direct, unauthenticated calls to Claude's API — these work inside a published artifact because the platform handles auth; outside of it, the AI features (meal estimation, coach chat) won't function without wiring up your own API key and backend.

**To actually use it**: open it as a Claude artifact and **publish it** (required for storage to activate) — see [Anthropic's artifact docs](https://support.claude.com) for how publishing and persistent storage work. Once published, it's usable from web or desktop; mobile behavior (especially inside the native Claude app vs. a mobile browser) hasn't been fully verified.

**To adapt it elsewhere**: the calculation logic, UI, and rendering are all plain, portable JavaScript — only the `load*`/`save*` storage functions and the `fetch` calls in `callEstimate`/`callCoachText`/`callCoachChat` are platform-specific and would need to be swapped for your own backend and API key handling.

---

## Privacy & data

- All storage is **personal, per-user** (`shared: false` on every call) — nobody else who opens this artifact can see your data, and you can't see theirs.
- Meal photos are **never persisted** — they're resized client-side, sent once for AI estimation, and discarded; only the resulting numeric estimate is saved.
- Full data export (CSV) and full deletion are both built in — see the Profile page's "Manage your data" section.

---

## Known limitations

- 20MB total storage cap per artifact (platform limit)
- Last-write-wins, no conflict resolution across multiple open tabs/devices
- "Today" is your browser's local clock — no timezone handling
- AI estimates are unverified LLM output — editable, not fact-checked
- No offline queueing — a failed save must be retried manually

Full list and technical detail in [`steady-technical-documentation.md`](./steady-technical-documentation.md).

---

## About this project

This was built as a demonstration of AI-assisted development for a full writeup of the process: how requirements and design decisions were driven by product/QA judgment, real production bugs that were caught and fixed, and what that says about applying the same approach to other software work.

**Built by Alisa Zeleva** with [Claude](https://claude.com).

---

## License

*Not yet set*

---

## Disclaimer

This is a personal project, provided as-is with no warranty. It is not medical or professional nutrition advice — estimates and suggestions are AI-generated and may be inaccurate. Consult a doctor or registered dietitian for personalized guidance. Use at your own risk.

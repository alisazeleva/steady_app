# steady

**A daily ledger for gradual, steady progress.**

A nutrition and body-recomposition tracker built as a Claude artifact — designed, built, and debugged entirely through conversation with Claude, with no manual coding. Log meals by text or photo, get AI-estimated macros, track daily and weekly trends, chat with an AI nutritionist grounded in your actual logged data, and keep it all private to your own account.

[**Try the live app →**](#) *(a published claude.ai link will be added later)*

![steady screenshot](#) *(a screenshot will be added later)*

---

## What it does

- **AI meal logging** — describe a meal in plain text, or add a photo (Take Photo or Choose from Library), and get AI-estimated calories and macros, shown for review before anything is saved
- **Dynamic targets** — calorie/macro goals computed from the Mifflin-St Jeor formula, adjusted for your stated goal (recomposition, fat loss, muscle gain, maintenance), training frequency, and *today's* logged activity — built-in presets for Walk, Hike, Cardio, Strength, and Yoga, plus custom activities with an AI-estimated calorie rate
- **A 1–10 goal-alignment score** on every meal tip and daily summary — rating how well that meal or day supports your goal (never an average of its parts; a day is judged as a whole pattern), not a moral judgment of the food
- **AI nutritionist (Aino)** — a real back-and-forth conversation for today, re-grounded in your live data on every message, with the option to attach a photo for advice (never saved or logged — that's a separate path), and tap a chat icon on any meal tip to ask a follow-up question about that specific meal. Past days get a static, regeneratable summary instead of an ongoing conversation. Today's chat can be reset at any time, and is automatically cleared once the day it belongs to has passed.
- **Trends** — 4-week charts with visible axes, per-day target lines that shift with logged activity, and a body-measurements tracker (waist/bust/hips) with its own mini trend chart
- **Favorite recipes** — save frequent meals with full ingredient lists, searchable, with quick-tap re-logging and CSV export
- **Data control** — a date-range CSV export for your daily totals and body measurements, a "keep last N days" retention tool, full-history deletion, and a self-diagnosing storage health check that tells you plainly if something isn't saving instead of failing silently
- **In-app documentation** — a "User guidelines" section (alongside Data & privacy, reachable from every screen) covering how the app works, the score, best practices, troubleshooting, and how demo data behaves

---

## Tech stack

Plain HTML/CSS/JavaScript — one self-contained file, no build step, no framework, no backend beyond the Claude artifact platform:

- **Persistence**: the artifact's private per-user storage (the `db` capability) — one small document per saved item, kept in your own area and hidden even from the person who built the app
- **AI**: the artifact's Claude access (the `sample` capability, including photos) — no API key needed, billed against the viewer's own Claude usage, and allowed by the viewer the first time it's used
- **Downloads**: CSV exports go through the artifact's file-save prompt (the `downloads` capability)
- **Rendering**: hand-written `innerHTML` templating, no virtual DOM, no React
- **Charts**: hand-drawn inline SVG, no charting library
- **Icons**: hand-drawn SVG throughout (not icon fonts or emoji), chosen deliberately after several icon-reliability bugs surfaced from relying on Unicode glyphs and browser-default rendering across platforms
- **Fonts**: Fraunces (display serif, with a Georgia fallback in case the Google Fonts request fails to load), Inter (body), JetBrains Mono (numbers) via Google Fonts `@import`

See [`steady-technical-documentation.md`](./steady-technical-documentation.md) for the full architecture, storage schema, and calculation reference.

---

## Running this project

This app is built specifically as a **Claude artifact**, not a standalone static site. It relies on four features that only exist inside a published artifact: private per-user storage, a way to ask Claude from the page, a file-save prompt for CSV exports, and the viewer's identity (which addresses your private storage area). Opened as a plain HTML file or on a generic static host, the UI loads but **nothing saves** and the AI features don't work.

**To use it**: open the artifact's published link while signed in to Claude. The first time you use an AI feature, Claude asks you to allow it, and each CSV export asks you to confirm the save.

**Development**: the published artifact is the source of truth. There is no separate local file to keep in sync; changes are made to the artifact and republished to the same link, which keeps everyone's stored data. The copy in this repository is a snapshot of it.

**Mobile note**: the app works inside the Claude mobile app's built-in view for most features, but two known gaps exist there specifically (not in a regular mobile browser): photo file inputs may not open the camera/gallery picker reliably, and custom web fonts loaded from Google Fonts may fail to load, falling back to a generic serif for headers. Both are limitations of that specific embedded webview, not the app's code — opening the artifact's link in the phone's regular browser avoids both.

**To adapt it elsewhere**: the calculation logic, UI, and rendering are plain, portable JavaScript. Only the block at the top of the script that wraps storage, Claude calls and downloads (`getBridge`, the `kv*` and `storage*` functions, `aiFetch`, `saveCsv`) is platform-specific, and would need to be swapped for your own backend and API key handling.

---

## Privacy & data

- Your data is stored in your own **private area** of the artifact's storage — nobody else who opens this artifact can see it, including the person who built it, and you can't see theirs.
- Meal photos are **never persisted** — they're resized client-side, sent once for AI estimation, and discarded; only the resulting numeric estimate and a short AI-written description are saved.
- Photos attached in the daily chat are held **only in memory** for that one message and discarded immediately after sending — never saved to storage, and not even kept for the rest of the conversation.
- Chat conversations are inherently ephemeral: today's chat is the only one that's ever interactive, and once a day passes, its chat history is automatically deleted from storage (it was already never shown again in the UI even before that).
- Full data export (CSV) and full deletion are both built in — see the Profile page's "Manage your data" section.

---

## Known limitations

- Storage limits: 256 KiB per stored item and 25,000 items per artifact across all viewers (platform limits), plus a per-viewer rate limit on storage calls
- AI features need the viewer to allow Claude on first use; if you decline, they stay unavailable
- Last-write-wins, no conflict resolution across multiple open tabs/devices
- "Today" is your browser's local clock — no timezone handling
- AI estimates are unverified LLM output — editable, not fact-checked
- No offline queueing — a failed save must be retried manually
- Photo file inputs and custom web fonts may behave differently inside the Claude mobile app's built-in view than in a regular mobile browser (see Mobile note above)
- No cache-busting — this is one HTML file with no versioned filename, so an already-open tab (or a normal, non-hard refresh) can keep running old code after an update. Hard refresh (or open a fresh tab) after any update to guarantee you're on the latest version.

Full list and technical detail in [`steady-technical-documentation.md`](./steady-technical-documentation.md).

---

## About this project

This was built as a demonstration of AI-assisted development — see [`ai-assisted-coding-case-study.md`](./ai-assisted-coding-case-study.md) for a full writeup of the process: how requirements and design decisions were driven by product/QA judgment, real production bugs that were caught and fixed, and what that says about applying the same approach to other software work.

**Built by Alisa Zeleva** with [Claude](https://claude.com).

---

## License

All rights reserved — see [`LICENSE.md`](./LICENSE.md). This code is public for viewing only; no permission is granted to use, copy, modify, or distribute it.

---

## Disclaimer

This is a personal project, provided as-is with no warranty. It is not medical or professional nutrition advice — estimates and suggestions are AI-generated and may be inaccurate. Consult a doctor or registered dietitian for personalized guidance. Use at your own risk.

# Building with AI: a case study

**How I use Claude to design, build, debug, and ship real software — for myself and at work.**

[**Click here to try the live app**](https://claude.ai/artifact/4aX39ZtdWs875zq5fMK6jm)

---

## The short version

I designed and built **steady**, a working nutrition-tracking web app, through iterative conversation with Claude — no manual coding. Over dozens of sessions I acted as product owner and QA: setting requirements, reviewing UI mockups before a line of code was written, catching real production bugs, and pushing back until the details were right. This page walks through how that process actually worked, and what it says about how I'd use the same approach on a team.

---

## The project: steady

Steady is a daily macro/calorie tracker built for a specific goal: slow, sustainable body recomposition — not a crash diet. It runs as a self-contained web app (HTML/CSS/JS) with:

- **AI-powered meal logging** — describe a meal in plain text, or snap a photo, and it estimates calories and macros automatically
- **A conversational AI coach** — chat about your day's meals and get specific, context-aware suggestions, grounded fresh in your actual logged data every message
- **Dynamic targets** — calorie and macro goals that adjust for your stated goal (recomposition, fat loss, muscle gain, maintenance), training frequency, and *that day's* logged activity (multiple activities, each with a duration, summed into a real calorie adjustment)
- **Trends and export** — 4-week charts with visible axes and target lines, plus CSV export with a custom date range
- **A recipe/favorites system** — save frequent meals with ingredients for one-tap re-logging
- **Data control** — retention settings, full-history deletion, and a self-diagnosing storage health check that tells you plainly when something isn't saving instead of failing silently

None of this was scoped up front. It was built the way real products get built: one requirement, one bug, one design revision at a time.

---

## How the process actually worked

**1. I set direction, Claude proposed a plan, I approved or redirected it.**
Before any nontrivial change — a new feature, a data-model change — Claude laid out the approach and trade-offs first. I'd approve, adjust, or scope it down. This kept me in control of product decisions while offloading implementation.

**2. Design decisions were visualized before being built.**
For UI and color-palette changes, I asked to *see* options rendered side-by-side before committing code — comparing background treatments, chart styles, and icon choices as live mockups, not descriptions. That let me make fast, confident calls without waiting on a full implementation to judge whether something looked right.

**3. I caught real bugs — including ones Claude introduced.**
This is the part that mattered most. Partway through, a performance optimization introduced a subtle data-loss bug (a caching layer that silently skipped historical data under specific conditions). When something looked wrong in the app, I reported the symptom, not the cause — and Claude traced it back through the actual code rather than guessing. When storage started failing intermittently, I pushed to find the *root* cause rather than accept a workaround, which eventually surfaced two more real bugs: a "delete all data" button that wasn't deleting everything it claimed to, and a data-retention feature with the same gap. All were found by treating "it doesn't look right" as a debugging starting point, not a dead end.

**4. Every change was verified, not assumed.**
Code changes were checked for syntax validity before being shipped, every time — not just written and handed over. When I asked factual questions about how the underlying platform worked (storage limits, billing, ownership terms), answers were pulled from current documentation rather than guessed from memory, and I was told plainly when something was genuinely uncertain instead of getting a confident wrong answer.

**5. Iteration was fast enough to actually explore the design space.**
Because implementation cost approached zero, I could try a color palette, reject it, try another, and land on one I actually liked — instead of committing early because changes were expensive. The same was true for UX flows: several features (the activity picker, the coach chat) went through a visual mockup, a round of feedback, and a rebuild before final implementation.

---

## What this demonstrates

- **Product thinking under AI assistance**: knowing what to ask for, in what order, and when to say "no, not that" — the skill isn't prompting, it's judgment.
- **QA instinct**: most of the real value I added was *not* trusting first drafts — testing edge cases, asking "what happens if two people use this at once," and refusing to accept "it should work now" without proof.
- **Debugging discipline**: reporting symptoms accurately and pushing for root causes instead of surface patches, even when a quick fix was available.
- **Comfort with iteration speed**: treating a working prototype as a starting point to refine, not a final answer to defend.
- **Working across a full build**: this wasn't just chat — it spanned UI design, state management, a persistence layer with real data-migration constraints, third-party API integration, and ongoing production debugging.

---

## Applying this at work

*[This is the section to make specific to you — a template, not a finished claim. A few honest starting points, worth replacing with your own real examples:]*

- A recurring manual task in my role that I've automated or tooled around using this same request → review → verify loop
- An internal tool, dashboard, or script I built to solve a team problem, where I drove the requirements and Claude handled implementation
- A time I used AI assistance to debug or understand an unfamiliar codebase faster than I could have alone
- How I evaluate AI-generated code before trusting it in a real system (the same instinct on display above: verify, don't assume)

*(Steady is deliberately included here as the fullest, most inspectable example — because it's something a reviewer can actually open and try, not just take my word for.)*

---

## Tech notes

Plain HTML/CSS/JavaScript, no build step or framework — runs entirely client-side, calls the Claude API directly for AI features, and uses a simple key-value persistence layer for data storage. Built and iterated entirely through conversation with Claude (Sonnet).

*Built with Claude — [read more about how](https://claude.com) or [see the code](#).*

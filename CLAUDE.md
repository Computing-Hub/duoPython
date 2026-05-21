# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

**Indent** — a Duolingo-style web trainer for learning Python. It is a zero-dependency,
zero-build static **PWA**. There is no package.json, bundler, framework, test runner, or
linter. Everything is hand-written vanilla HTML/CSS/JS.

> Note: the on-disk repo is `duoPython` and the app was previously called *PyLingo*. The
> brand is now **Indent**, but `localStorage` keys keep the historical `pylingo_*` prefix
> (changing them would wipe existing students' progress). Don't "fix" that.

## Running & developing

There is no build step — edit files and reload. But you **must serve over HTTP**, not
`file://`, or the service worker won't register and PWA behavior breaks:

```sh
python3 -m http.server 8000   # then open http://localhost:8000
```

There are no automated tests or lint — verification is manual in a browser.

## Where everything lives

Almost the entire app is **`index.html`** (~3360 lines): `<style>` block, then one big
`<script>`. The other tracked files are small and supporting:

- `sw.js` — service worker (offline cache + update banner)
- `manifest.json` — PWA manifest
- `mascot.svg`, `icon-*.png`, `apple-touch-icon.png` — branding assets
- `ATTRIBUTIONS.md` — asset provenance

## Architecture (the parts that span files / functions)

**Screens, not routes.** The UI is a set of full-screen `<div>`s; `showScreen(id)` (around
`index.html:1971`) hides all and shows one. There is no router and no URL state. The screen
list is the `screens` array. The home screen is `mapScreen`.

**Single `state` object → localStorage.** `state = { xp, completed, streak, lastActive }`
(`completed` is `{ lessonId: true }`). `loadState()`/`saveState()` persist to
`STORAGE_KEY` (`pylingo_v2`), with a one-time migration from `pylingo_v1`. Sound and arcade
high score are stored under separate keys. Always go through `saveState()` after mutating
`state`.

**Course content is data, in `TOPICS`** (`index.html:747`). 10 topics. The 4 **core** topics
have 4 levels each (L1→L4 — the 4th being `vars_l4` "Casting", `sel_l4` "Ternary & Nesting",
`loops_l4` "Patterns & Steps", `funcs_l4` "Built-in Functions"); the 6 **advanced** topics
have 3 levels (L1→L3). Each level holds an array of `exercises`. Level count per topic is not
fixed: the map renders one node per `topic.levels` entry and the card subtext is generated
from that array, so adding/removing a level needs no other code changes. The 4 **core** topics are listed explicitly in
`CORE_TOPIC_IDS` (`vars, select, loops, funcs`); the 6 `advanced: true` topics (`strings`,
`lists`, `oop`, `logic`, `datarep`, `media`) stay locked until L2 is cleared in at least
`ADVANCED_UNLOCK_THRESHOLD` (2) core topics — see `topicUnlocked()` / `coreL2Cleared()`.
Within a topic, levels unlock sequentially (`levelState()`). **To add or change lessons,
edit the `TOPICS` data — no code changes needed.** Each topic needs a `color`/`colorD` pair
from the `:root` palette (one distinct colour per topic).

**Exercise type system.** Each exercise has a `type` that drives both rendering and grading:
- `mc` — multiple choice (`renderMC`); `codeChoices: true` renders choices in monospace
- `fill` — fill in the blank in a code snippet using `___` (`renderFill`)
- `tokens` — assemble an answer from shuffled token chips (`renderBuild`)
- `order` — reorder shuffled lines of code (`renderBuild`, `isLines` branch)

Grading lives in `isCorrect()`. If you add a new exercise type you must touch the render
dispatch (`renderExercise`), `isCorrect`, `correctText`, and `hasResponse`.

**Three play modes share the exercise renderers** but have separate session/flow logic:
- **Lessons** — the map flow. `startLesson` → per-question check → `finishLesson`, with a
  hearts budget (`HEARTS_PER_LESSON`). Awards XP/crowns and marks `completed`.
- **Arcade** (`openArcade`/`startArcadeRun`) — timed survival run. Lives, a countdown timer
  that shortens with score (`timerForScore`), combo multipliers, and score *tiers*
  (`ARCADE_TIER_L2/L3`) that fold harder questions into the pool. Pulls real exercises out
  of `TOPICS` via `arcadePool()`.
- **Practice** (`openPractice`/`startPractice`) — endless, **procedurally generated**
  questions. `PRACTICE_GENERATORS` (`index.html:2892`) maps each topic to `gen*` functions
  that build fresh exercises with `pickInt`/`pickFrom`/`distinctChoices`. A generator only
  becomes eligible once the student has **completed a level** in the matching `TOPICS` track
  (`practiceTopicUnlocked` → `eligiblePracticeGenerators`). Not every track has generators
  (e.g. `oop` has none), so the practice topic set is a subset of `TOPICS`.

All three funnel into a shared `session` object and the same exercise DOM; keep that
contract in mind when editing renderers.

**Service worker / updates.** `sw.js` is **network-first for HTML** (students always get the
latest `index.html` when online) and **cache-first for static assets**. Content changes ship
automatically on reload. Only bump `CACHE_VERSION` in `sw.js` when you want to force a clean
cache flush / show the in-app "Update available" banner (`showUpdateBanner`).

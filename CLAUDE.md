# Japan Trip 2026 app — working notes

This is the **active, deployed** repo — a single-file static PWA (`index.html`, ~750KB: all HTML/CSS/JS in one file, no build step, no framework, no backend). It's what's live at the GitHub Pages site and what both Hanan and Pola actually use day to day. Deployment is `git push origin main` — no manual copy/upload step.

This file intentionally does **not** carry the full itinerary-planning history. That lives in a separate archive folder (`~/Downloads/EXPORT_japan-trip-2026/`, its own `CLAUDE.md`/`PROJECT_INSTRUCTIONS.md`) from when the trip was actively being planned. That phase is mostly closed — see "Planning status" below for the exceptions.

## Architecture

- **Persistence**: everything is `localStorage`, synced across Hanan's and Pola's devices via a Firestore module already in the file (fixed trip doc ID, no login — any device opening this exact site auto-joins the same shared state). Synced keys: `v2_state`, `jt_done`, `jt_book`, `jt_cbk`, `jt_lug` — any new *top-level* key needs adding there; new fields nested inside an already-synced key (e.g. inside `v2_state` or a `jt_cbk` entry) sync for free.
- **`V2Store`**: a localStorage overlay object (`hidden`/`edits`/`links` maps, keyed by a static event's `data-id`) used to hide/edit/link pre-planned schedule cards without touching the underlying static HTML. Custom user-added events (`jt_cbk`) are a separate system with their own id space (`cb0`, `cb1`, ...) and deliberately never use `data-id`, so the two systems never collide.
- **Static day markup**: all 36 days live as hardcoded HTML (`<section id="dayN">`), each card in a `.tl` container. There's no numeric order/time field backing this — visible order is pure HTML source order, and the Morning/Afternoon/etc. labels are decorative text with no functional role in that view (they're only used, loosely, by the Now-screen's approximate-time heuristic). Custom events get inserted into the correct chronological slot among these static cards by a boot-time render pass, not by editing the static HTML.

## Quirks that will bite you

- **`interaction-fixes` script**: a capture-phase document click handler that hijacks any element with class `.addbk` or `.attbtn` — `.addbk` clicks open an old, effectively-dead booking modal; `.attbtn` clicks get forced through `pickFile()` regardless of what the element's own handler was supposed to do. **Never assign either class to a new/different-purpose button** — style-clone the CSS instead if you want that look.
- **`file-picker-fix-js` script**: runs on every DOM mutation (via `MutationObserver`) and physically removes any `.attbtn` element it finds, replacing it with a proper `<label class="file-picker">` control. So raw `.attbtn` buttons get superseded at runtime regardless of CSS visibility — don't be surprised if a CSS change to `.attbtn` has no visible effect.
- **In-app edits vs. this file**: edits made through the live app (Edit button, Hide from schedule, custom events, done-checks) are stored client-side (`V2Store` overlay in `localStorage`, synced across devices via Firestore) — they never touch `index.html` and are invisible from this repo. So "I already fixed that in the app" can be true even when a grep of `index.html` still shows the old content — that's expected, not a bug. If Hanan/Pola say something is already handled, ask whether that was an in-app edit (fine, no repo change needed) or something they want baked into the static file as the new baseline (needs an actual edit here). There is currently no sync mechanism from the app back to this repo — see `FUTURE_IDEAS.md` for the parked hands-free auto-sync idea.

## Hard rules

- No native `alert()`/`confirm()` — use the existing `nwConfirm()` helper. (Two pre-existing native `confirm()` calls are known legacy, not something to imitate.)
- Bump the version stamp in the header on every push: format `vN · bK · DD.MM` (date only, no time).
- No new `<style>` layer — CSS edits go inside the existing `#design-washi`/`#v2-components` blocks.
- No emoji in UI strings — SVG icons only. (Plain dingbat glyphs already in long-standing use — ✓, ✕, ➔ — are not emoji for this rule's purposes.)
- Every `PINS` entry (the map-data array) carries an `"id"` field matching its schedule card's `data-id`, except `"cat":"Hotel"` entries (no `.card` counterpart exists for those). `app-patch-reviewer`'s rule W9 enforces this on every future edit — a card added/removed/renamed must keep its PINS twin in sync via the `id` link.
- Sourcing standard for any place/venue content: Tabelog (restaurants/bars/cafes only — not rated for sights/attractions) + Google Maps (open-status and location only, never as a rating source). TripAdvisor is banned. See `place-verifier` agent for the full rule set (allergies, smoking, back/neck safety, tourist-trap filter).
- **Never reuse a `data-id` for different content — including an id whose old card was removed or fully replaced.** `V2Store` edits are keyed by `data-id` and persist indefinitely in Hanan/Pola's synced state, invisible from this repo (see the in-app-edits quirk above). Give repurposed or replaced cards a fresh id instead (check it's never appeared before with `git log --all -S'dNiM'`) — reusing an old id risks a stale edit silently reappearing on unrelated new content, with no error and no trace in `index.html`, only reproducible live in the app. This has actually happened three times (KI NO BI→Bar TRENCH showing the wrong title, a pre-emptive Bar Kugel→BARCRAFT rename, and Rest-at-hotel→Sennichimae Doguyasuji showing stale text through a hard refresh) — treat any "the app still shows old text/title after a confirmed-correct deploy" report as this bug first, not a caching issue, whenever the card's content was recently changed on a pre-existing id.

## After every push

Immediately after `git push`, give Hanan a short checklist of what changed — one line per change, in plain terms (which day/card, what's different), so he knows exactly what to open the app and test. Don't wait to be asked. Group multiple small pushes from the same conversation topic into one checklist if that reads better, but never skip it entirely.

## Subagent dispatch

Two subagents live in `.claude/agents/` here:
- **`place-verifier`** — dispatch when adding a genuinely new place, or changing/replacing an existing one. Don't re-verify something just because it's old — but don't assume a day is closed either; check the "planning status" list below first.
- **`app-patch-reviewer`** — dispatch after any `index.html` edit, ideally with a prior git ref to diff against (e.g. the commit before your changes) so it can precisely check things like "did the version number actually increment" and "was a new `.addbk`/`.attbtn` class added" rather than just scanning a single snapshot.

## Future ideas

Implementation ideas that get raised and set aside for later live in `FUTURE_IDEAS.md`. Check it when starting new work, and add anything new that gets parked mid-conversation instead of letting it get lost.

## Planning status

Itinerary content is locked (36 days, approved), including Kamakura (Day 6) — confirmed set by Hanan on 2026-08-08, no longer under reconsideration.

Extend this list if items get reopened.

# Future implementation ideas

Things worth doing eventually, deliberately set aside for now. Not a bug list — see git history/issues for that. Add a short "why parked" note when setting something aside, and remove/move it here to done once implemented (or delete if abandoned).

## Open

- **Alternatives as a carousel** — instead of the current show/hide-all toggle for alternative options on a day, consider a swipeable carousel UI for browsing alternatives. (Raised 2026-08-07, not yet scoped.)
- **Hands-free auto-sync from app to this repo** — a one-tap in-app button that automatically commits current in-app edits (V2Store overlay, custom events, done-checks) straight to `index.html` in this GitHub repo, no manual export/download step. Would need a Firebase Cloud Function (watching the Firestore trip doc) that calls the GitHub API to commit changes, plus logic to safely translate overlay data into valid static HTML. Explicitly deferred 2026-08-07 because it adds a backend + a GitHub write token to manage, and a bad auto-merge could break the live site. Revisit if manually reconciling "app says fixed, repo says not fixed" (see CLAUDE.md's in-app-edits-vs-repo note) becomes a recurring pain. Simpler fallback considered and also parked: a button that exports the overlay as a downloadable JSON file to hand to Claude manually.

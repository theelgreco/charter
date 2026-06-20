<!-- charter-rule v1 · managed by the Charter plugin; /charter:scaffold offers to refresh this file when a newer version ships -->

# Charter — the docs are the source of truth

This repo uses **Charter** doc sets as its single source of truth. Code is written
*from* the docs, never the other way around. Before implementing anything, find and
read the doc set that governs it.

## Where the docs live

- **Project** — the whole-product SSOT: repo-root `docs/`.
- **Feature** — one unit (module / package / domain / service): `<unit>/docs/`
  (or `docs/<unit>/` if the unit had no code home).
- **Task** — ephemeral, one ticket's worth: `.claude/tasks/<key>/` (gitignored).

A set holds **PRODUCT** (what it is) · **SPEC** (how it's built) · optional
**DESIGN** (how it looks & behaves) · **ROADMAP** (what to build next) · optional
**FUTURE** (what it can become). The `/charter:locate-docset` skill resolves which
set applies here.

## How to work

- **Read before you build:** PRODUCT → SPEC → [DESIGN] → ROADMAP, plus
  `CLAUDE.md` / `AGENTS.md` for repo-wide conventions.
- **Build milestone by milestone** with `/charter:milestone <M>`; settle or extend
  the docs with `/charter:scaffold`.
- **Don't let code diverge from the docs.** If the work needs something the docs
  don't cover, it's either out of scope (defer it) or the docs must be updated
  *first*, with the user's agreement — never let the code silently drift.

## Precedence when sources conflict

Within a set: **SPEC ▸ PRODUCT ▸ DESIGN** (SPEC owns technical truth; PRODUCT owns
product intent; DESIGN owns UX; a referenced visual prototype is canonical for
appearance but loses to SPEC on data shape). Across altitudes, **project docs
outrank feature/task docs** on product/architecture intent, while
`CLAUDE.md` / `AGENTS.md` owns repo-wide conventions.

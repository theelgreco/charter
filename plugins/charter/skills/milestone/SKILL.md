---
name: milestone
description: Execute a roadmap milestone from a Charter doc set (project/feature/task) produced by /charter:scaffold. Use when implementing a specific milestone — code is written from the docs, not the other way around.
argument-hint: "<milestone, e.g. M2> [unit or task key]"
---

Execute milestone **$ARGUMENTS** from a doc set produced by `/charter:scaffold`. Code is
written *from* the docs, not the other way around.

## Step 1 — Locate the doc set

A milestone lives in a doc set produced by `/charter:scaffold`, in one of three places
(see `/charter:scaffold` for how they're created):

- **project** (in-repo): repo-root `docs/` — the SSOT for a whole product.
  Has a `ROADMAP.md` only when the product is *built directly* from the root docs
  (smaller products). Larger, *decomposed* projects keep no root ROADMAP —
  roadmapping lives in the per-feature sets.
- **feature** (in-repo): `<unit>/docs/` — the permanent SSOT for one unit
  (module / package / domain / service), e.g. `billing/docs/`. If the unit had no
  natural code home at scaffold time, the set may instead live at repo-root
  `docs/<unit>/`.
- **task** (host, not in git): `~/.claude/projects/<encoded-repo-root>/tasks/<key>/`
  — an ephemeral, cross-cutting chunk keyed by ticket key or branch/feature name.
  `<encoded-repo-root>` is the **main worktree root** with every non-alphanumeric
  character replaced by `-` (e.g. `/Users/me/git/app` → `-Users-me-git-app`).
  Derive it from git so every worktree/chat for the task shares one set:
  `root=$(cd "$(git rev-parse --git-common-dir)/.." && pwd)`. This is a **fixed
  host path**, the same regardless of which worktree you're in. If the repo isn't
  under git, `/charter:scaffold` keyed the set off the cwd instead — encode the cwd to find it.

Resolve which set this milestone belongs to, in order:

1. If `$ARGUMENTS` includes an identity token whose `<unit>/docs/ROADMAP.md` or
   repo-root `docs/<unit>/ROADMAP.md` (the `/charter:scaffold` no-code-home fallback)
   exists → **feature mode**.
2. Else if a token (or the current git branch) yields a task `<key>` whose
   host-path `ROADMAP.md` exists → **task mode**. Try the bare key too (e.g.
   `BMR-1985` from `BMR-1985-watchlist-webhook`).
3. Else if the cwd is under a unit dir with `docs/ROADMAP.md` → **feature mode**.
4. Else if repo-root `docs/ROADMAP.md` exists → **project mode**.
5. Else if repo-root `docs/` exists but has **no** `ROADMAP.md` → this is a
   *decomposed* project. Stop and tell the user roadmapping lives in the
   per-feature sets; list the `*/docs/ROADMAP.md` and `docs/*/ROADMAP.md` files
   found and ask which unit.
6. Else **stop and ask** which doc set this milestone belongs to — list every
   `ROADMAP.md` found under the repo and under the tasks dir.

Call the resolved directory `<docs>`. **Project and feature are both in-repo and
are handled identically from here** (committed, milestone-tagged commits); only
**task mode** differs — its docs live on the host, are never committed, and don't
appear in a PR.

## Step 2 — Read the docs in order

Read `CLAUDE.md` (or `AGENTS.md`) first for repo-wide conventions. If you're in a
feature/task set **and** a repo-root project `docs/` set exists, also read its
`PRODUCT.md` / `SPEC.md` / `DESIGN.md` as higher-level context. Then, from
`<docs>`: `PRODUCT.md` → `SPEC.md` → `DESIGN.md` (if present) → `ROADMAP.md`.
Skim `FUTURE.md` if present (project/feature mode) so you know what's explicitly
out of scope; in task mode read the ROADMAP's **Out of scope** section for the same.

Honour the SSOT rules stated atop every doc:

- **Conflict precedence:** `SPEC ▸ PRODUCT ▸ DESIGN` within a set (SPEC owns
  technical truth; PRODUCT owns product intent; DESIGN owns UX). If a DESIGN doc
  points at a visual prototype, the prototype is canonical for *appearance* but
  loses to SPEC on *data shape*. Across altitudes, **project docs outrank
  feature/task docs** on product/architecture intent, while `CLAUDE.md` /
  `AGENTS.md` owns repo-wide conventions.
- If the work needs something not in the docs, either it's out of scope (defer
  it — to `FUTURE.md` in project/feature mode, or to the tracker in task mode) or
  the docs must be updated **first** (with the user's agreement). Never let the
  code silently diverge from the docs.

## Step 3 — Check dependencies before starting

Find the **Depends on:** line for the milestone in `<docs>/ROADMAP.md`. For each
prerequisite milestone listed, confirm it has landed:

- **With git:** run `git log --grep="<prereq>:"` (e.g. `git log --grep="M2:"`)
  and confirm at least one matching commit exists. Satisfying commits are
  subject-prefixed with the milestone tag (optionally after a ticket key, e.g.
  `BMR-1985 M2: …`).
- **Without git** (or for task-mode docs, whose host dir isn't git): treat the
  prerequisite's ` [done]` heading in `ROADMAP.md` as the marker instead.

If anything looks missing, **stop and ask** before continuing.

## Step 4 — Do the milestone, and only the milestone

Implement exactly the milestone's **acceptance criteria**. Don't skip ahead or
drift into adjacent milestones. For any UI surface in scope, read the relevant
DESIGN section (and prototype screen, if one is referenced) first.

## Step 5 — Mark it done

Don't commit unless the user asks. When the acceptance criteria are met, the work
is verified, and the user approves:

- **In-repo (project / feature):** the satisfying commit is subject-prefixed with
  the milestone tag (`<milestone>: …`) and, **in the same commit**, flips the
  milestone's `ROADMAP.md` heading to suffix ` [done]`.
- **Task mode** (docs on the host, not in git): the satisfying *code* commit is
  subject-prefixed `<milestone>: …` (after the ticket key is fine); flip the
  milestone's heading to ` [done]` in the host-dir `ROADMAP.md` once that commit
  lands. The doc edit isn't committed anywhere — the host dir is not a git repo.
- **No git at all:** there's no commit to tag — flipping the milestone's
  ` [done]` heading in `ROADMAP.md` is the sole completion marker.

---
name: milestone
description: Execute a roadmap milestone from a Charter doc set (project/feature/task) produced by /charter:scaffold. Use when implementing a specific milestone — code is written from the docs, not the other way around.
argument-hint: "<milestone, e.g. M2> [unit or task key]"
---

Execute milestone **$ARGUMENTS** from a doc set produced by `/charter:scaffold`. Code is
written *from* the docs, not the other way around.

## Step 1 — Locate the doc set

A milestone lives in a doc set produced by `/charter:scaffold`, in one of three
modes — **project** / **feature** (in-repo, committed) or **task** (in the main
worktree's `.claude/tasks/<key>/`, gitignored, never committed).

**Resolve it with the `locate-docset` skill** instead of re-deriving the rules
here: read `${CLAUDE_PLUGIN_ROOT}/skills/locate-docset/SKILL.md` and follow it,
passing through any identity token in `$ARGUMENTS` (a unit or task key). **Don't**
pass a mode — let it *infer* from the arguments, the cwd, the git branch, and
what's on disk; a milestone's set must already exist. It reports `mode`, `docs`
(the doc root — call it `<docs>` below), `committed`, `unit`/`key`, `exists`,
`present`, and `has_roadmap`.

If it can't settle on a single existing set — e.g. a **decomposed project**
(repo-root `docs/` with no `ROADMAP.md`, where roadmapping lives in the per-feature
sets) or nothing matched — it stops and asks; surface that rather than guessing.

**Project and feature are handled identically from here** (committed,
milestone-tagged commits); only **task mode** differs — its docs live in
`.claude/tasks/`, are never committed, and don't appear in a PR.

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
  `BMR-1985 M2: …`). This covers **task mode too** — task *code* commits are tagged
  and committed to the repo like any other; only the task docs are gitignored.
- **Without git:** treat the prerequisite's ` [done]` heading in `ROADMAP.md` as the
  marker instead.

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
- **Task mode** (docs in `.claude/tasks/`, gitignored): the satisfying *code* commit
  is subject-prefixed `<milestone>: …` (after the ticket key is fine) and lands in
  the repo like any other; flip the milestone's heading to ` [done]` in
  `.claude/tasks/<key>/ROADMAP.md` once that commit lands. That doc edit is **not
  committed** — `.claude/tasks/` is gitignored — so it never appears in a PR.
- **No git at all:** there's no commit to tag — flipping the milestone's
  ` [done]` heading in `ROADMAP.md` is the sole completion marker.

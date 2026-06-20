---
name: milestone
description: Execute a roadmap milestone from a Charter doc set (project/feature/task) produced by /charter:scaffold. Use when implementing a milestone — named explicitly (e.g. M2) or auto-selected with next — code is written from the docs, not the other way around.
argument-hint: "<milestone (e.g. M2) | next> [unit or task key]"
---

Execute a ROADMAP milestone from a doc set produced by `/charter:scaffold` — either the
one **named in `$ARGUMENTS`** (e.g. `M2`) or, when `$ARGUMENTS` is `next`, the **next
unblocked milestone**, picked automatically. Code is written *from* the docs, not the
other way around.

`$ARGUMENTS` is a **milestone selector** — a specific tag (`M2`) or the literal `next` —
optionally followed by an **identity token** (a unit or task key) that scopes which doc
set to use. `next` is reserved for auto-selection; any other leading token is treated as
the milestone tag.

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

## Step 3 — Choose the milestone and confirm it's ready

First decide whether any given milestone has **landed** — the same judgement powers
both auto-selecting `next` and checking dependencies, and it works identically in every
mode. Milestone tags **restart at `M1` each batch**, and the current `<docs>/ROADMAP.md`
holds only the current batch (earlier batches are archived under `roadmaps/`), so the
authoritative signal is batch-local:

- **Landed = its heading in the current `<docs>/ROADMAP.md` carries ` [done]`.** For
  in-repo modes this suffix is flipped *in* the satisfying commit (see Step 5), so it
  always agrees with the commit record within a batch; in task and no-git modes it's the
  marker outright.
- **Don't** decide landedness by grepping all of history (`git log --grep="M1:"`) — a
  reused `M1` from an archived batch would match. The milestone tag on the commit
  (`<tag>: …`, optionally after a ticket key) stays the durable landing record; to
  confirm a commit landed, scope the grep to the current batch (since the last
  `Roll ROADMAP:` commit).

Then resolve which milestone to build:

- **A tag was named** (e.g. `M2`): that's the milestone. Find its **Depends on:** line in
  `<docs>/ROADMAP.md` and confirm every prerequisite has landed by the rule above. If a
  prerequisite is missing — or the named milestone has already landed — **stop and ask**
  before continuing.
- **`next` was given:** scan `<docs>/ROADMAP.md` top-to-bottom and pick the **first
  milestone that hasn't landed**. Announce the pick — its tag and one-line goal — before
  starting. Two edge cases, neither resolved silently:
  - *Its **Depends on:** prerequisites haven't all landed* — the roadmap is out of order,
    or an earlier milestone was left unfinished. Don't skip ahead; **stop and report**
    which prerequisite is missing, and ask.
  - *Every milestone has already landed* — the batch is complete; there's nothing to
    build. Say so rather than inventing work, and offer `/charter:roll-roadmap` to
    archive this batch and seed the next ROADMAP from FUTURE (project/feature only).

If anything looks missing or inconsistent, **stop and ask** before continuing.

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

**If that was the last un-done milestone** (project/feature mode), the batch is now
complete. Tell the user and offer `/charter:roll-roadmap` to archive this ROADMAP to
`roadmaps/NN-<label>.md` and seed the next batch from FUTURE — don't roll it over
automatically, since that settles a label and which FUTURE items to promote.

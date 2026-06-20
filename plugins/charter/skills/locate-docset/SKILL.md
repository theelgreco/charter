---
name: locate-docset
description: Resolve which Charter doc set applies here (project/feature/task) and where its docs live. The shared resolver that /charter:scaffold and /charter:milestone defer to, and runnable directly to find which doc set applies and where its docs are.
argument-hint: "[project | feature <unit> | task <key>]"
---

Resolve **which doc set applies here and where its docs live**. This is the single
source of resolution truth: `/charter:scaffold` and `/charter:milestone` both defer
to this procedure instead of each re-deriving it. It is a **pure resolver** — it
reads the filesystem and git, reports what it finds, and **creates nothing** (doc
roots are created by `/charter:scaffold`, never here).

Two ways it's used:

- **With an explicit mode** (how `/charter:scaffold` calls it, for *authoring*):
  resolve where the set lives **even if it doesn't exist yet**, and report its
  current state so the caller can resume.
- **Without a mode** (how `/charter:milestone` calls it, for *executing*): **infer**
  the mode from the arguments, the cwd, and what's on disk — the set must already
  exist.

## Inputs

- An optional **mode** token in `$ARGUMENTS`: `project`, `feature`, or `task`.
- An optional **identity** token: the `<unit>` (a code directory) for feature mode,
  or the `<key>` (ticket key or branch/feature name) for task mode.
- Implicit context: the current working directory, the current git branch, and the
  repo's git layout.

## Roots

Resolve the two roots once, up front, with git:

- **Committed root** (project & feature docs live here, in the working tree):
  `top=$(git rev-parse --show-toplevel)`. Use the **current worktree's** top, since
  committed docs are part of the branch you're on.
- **Task root** (task docs are shared and uncommitted): the **main worktree** root,
  so every linked worktree and chat for a task shares one set —
  `main=$(cd "$(git rev-parse --git-common-dir)/.." && pwd)`, then task docs live at
  `<main>/.claude/tasks/<key>/`.

If this isn't a git repo: use the cwd as both roots, and — for task mode — **warn**
that the set won't be shared across worktrees.

## Resolve the mode and doc root

### A. Explicit mode given (authoring path)

| Mode | Doc root | Committed | Notes |
|------|----------|-----------|-------|
| `project` | `<top>/docs/` | yes | — |
| `feature <unit>` | `<unit>/docs/` | yes | If `<unit>` has no real code home yet, fall back to `<top>/docs/<unit>/`. |
| `task <key>` | `<main>/.claude/tasks/<key>/` | **no** | Main-worktree path above; never committed. |

Resolve the path deterministically and report its state — it need **not** exist yet.

### B. No mode given (inference path)

Try these in order and stop at the first that matches:

1. An **identity token** whose `<unit>/docs/ROADMAP.md` **or** `<top>/docs/<unit>/ROADMAP.md`
   exists → **feature**.
2. A token (or the current git branch) yielding a task `<key>` whose
   `<main>/.claude/tasks/<key>/ROADMAP.md` exists → **task**. Try the **bare key**
   too (e.g. `BMR-1985` from `BMR-1985-watchlist-webhook`).
3. The **cwd is under a unit dir** with `docs/ROADMAP.md` → **feature** (that unit).
4. `<top>/docs/ROADMAP.md` exists → **project** (built directly).
5. `<top>/docs/` exists but has **no** `ROADMAP.md` → a **decomposed project**: stop
   and tell the user roadmapping lives in the per-feature sets; list the
   `*/docs/ROADMAP.md` and `docs/*/ROADMAP.md` files found and ask which unit.
6. Otherwise **stop and ask** which doc set this is — list every `ROADMAP.md` found
   under `<top>` and under `<main>/.claude/tasks/`.

## Report

Emit the resolution as these fields; the caller relies on them:

- **`mode`** — `project` | `feature` | `task`.
- **`docs`** — absolute path to the doc root.
- **`committed`** — `true` (project/feature) | `false` (task).
- **`unit`** — the code dir (feature mode only).
- **`key`** — the ticket/branch key (task mode only).
- **`exists`** — whether `docs` exists on disk.
- **`present`** — which of `PRODUCT` / `SPEC` / `DESIGN` / `ROADMAP` / `FUTURE` are
  on disk in `docs` (check `<docs>/<NAME>.md` for each).
- **`has_roadmap`** — project mode only: `true` if `<docs>/ROADMAP.md` exists (the
  product is built directly), `false` if it doesn't (a decomposed project whose
  roadmapping lives in per-feature sets).

If resolution couldn't settle on a single set (cases B5/B6, or a missing required
identity), don't guess — report what was found and ask.

# Charter

Doc-driven development, packaged as a Claude Code plugin + marketplace. Charter
scaffolds a single-source-of-truth **doc set** (PRODUCT/SPEC/[DESIGN]/ROADMAP/[FUTURE])
for a project, feature, or task, then builds it **milestone by milestone, from those
docs**. Code is written *from* the docs, never the other way around. This repo **is**
the product.

## Repo layout

- `.claude-plugin/marketplace.json` — this repo doubles as the marketplace (name `charter`)
- `plugins/charter/` — the plugin (name `charter`)
  - `.claude-plugin/plugin.json`
  - `skills/scaffold/SKILL.md` — author a doc set (interview-driven)
  - `skills/milestone/SKILL.md` — execute one ROADMAP milestone from a doc set
- `README.md` — install + usage

**Procedural truth for each skill lives
in its `SKILL.md`** — read the relevant one before changing its behaviour. This file
is the conventions + pointer layer, not a place to restate skill procedures.

## Status

`/charter:scaffold` + `/charter:milestone` validating; the shared resolver and the
docs-SSOT rule have landed. Build order, roughly in dependency order (✓ = done):

1. ✓ **`locate-docset`** — the shared resolver every other skill calls.
2. ✓ `.claude/rules/charter.md` written by `scaffold` (the docs-SSOT rule), with a
   `charter-rule vN` marker so existing repos can be offered updates.
3. ROADMAP lifecycle (archive completed batch to `roadmaps/`).
4. ✓ Task docs relocated to `.claude/tasks/` (landed with #1).
5. `milestone next`.
6. `reconcile-docs`, `status`, `promote`.

## Decisions / invariants

- **Everything is a skill** (`skills/<name>/SKILL.md`), not a flat command. Model
  invocation stays on unless a skill has side effects worth gating
  (`disable-model-invocation: true`).
- **`locate-docset` is the shared resolver** — "which doc set applies here
  (project/feature/task), and where do its docs live?" Every other skill calls it.
  That logic is currently duplicated inside `scaffold` + `milestone`; factoring it
  out is the first job.
- **Docs are the SSOT.** When `scaffold` creates a doc set it writes
  `.claude/rules/charter.md` into the *target* repo so the rule loads every session
  at CLAUDE.md priority (deliberately chosen over a SessionStart hook).
- **ROADMAP = the current batch of milestones** — forward-looking, **no dates**, not
  a time-boxed sprint. On completion (all `[done]`), archive to
  `<doc-root>/roadmaps/NN-<label>.md` (incremental number + label, no dates in the
  filename, one-line outcome header) and author a fresh ROADMAP seeded from FUTURE.
  **Project + feature modes only** — task mode keeps no `roadmaps/`.
- **Three modes:** `project` (`docs/`, committed), `feature` (`<unit>/docs/`,
  committed), `task` (ephemeral). Task docs live in the **main worktree's**
  `.claude/tasks/<key>/` (resolve via `git rev-parse --git-common-dir` so worktrees
  share one set); `scaffold` adds `.claude/tasks/` to `.gitignore` on the first task
  set. *(The ported SKILL.md files still reference the OLD `~/.claude/projects/...`
  host path — update them.)*
- **Conflict precedence within a doc set:** `SPEC ▸ PRODUCT ▸ DESIGN`. Project docs
  outrank feature/task docs; `CLAUDE.md`/`AGENTS.md` owns repo-wide conventions.

## Working conventions

- **Docs-first / plan-first.** Surface a short plan before any large or multi-file
  change and get the OK, then execute. Small reversible changes — just do them.
- **One coherent unit at a time** (one skill / one milestone per change).
- **Validate after changes:** `claude plugin validate ./plugins/charter` (and the
  marketplace with `claude plugin validate .`). Test live with
  `claude --plugin-dir ./plugins/charter`, then `/reload-plugins`.
- **Plugin file references must use `${CLAUDE_PLUGIN_ROOT}`** — installed plugins run
  from a cache and can't read paths outside the plugin dir.
- **Bump the `charter-rule vN` marker** in `skills/scaffold/templates/charter.md`
  whenever you change that rule's text — `scaffold` compares versions and offers
  existing repos the update; skip the bump and they never pick it up.
- **Frontmatter is strict YAML.** Quote any value starting with `[` or containing
  YAML-significant characters (this bit the `argument-hint` lines).
- **Commits:** plain messages, **no `Co-Authored-By` trailer**. Commit/push only when
  asked.
- **Read official library/CLI docs before writing or modifying code that uses them**
  (see `~/.claude/CLAUDE.md`). Plugin/skill/marketplace formats included — verify
  against current Claude Code docs, don't recall from memory.

## Read before deep work

1. This file.
2. The relevant `SKILL.md` (procedural truth for that skill).
3. Once it exists, Charter's own doc set under `docs/` — we plan to **dogfood**
   Charter on itself, so the ROADMAP there drives the remaining build.

---
name: reconcile-docs
description: Reconcile a Charter doc set with the code as actually built — surface every doc↔code drift and milestone-status mismatch, then resolve each with you (update the stale doc to match reality, or flag code that wrongly diverged). Use when code and docs may have fallen out of sync.
argument-hint: "[unit or task key]"
disable-model-invocation: true
---

Bring a Charter doc set back into agreement with the code as it was **actually built**.
Charter's rule is that code is written *from* the docs and never silently diverges — but
reality drifts: a value gets tuned in code, an endpoint is added, a milestone lands
without its heading flipped. This skill surfaces every such drift and resolves each one
**with you**, in the right direction: when the docs are merely stale, update them to
match legitimate reality; when the code wrongly diverged, leave the docs and flag the
code. The docs stay the source of truth — reconciling means making the SSOT *true again*,
not rubber-stamping whatever the code happens to do.

It **edits docs** (with your sign-off) and may commit, so it's user-invoked, not
model-invoked. It owns the **docs**, not the product code: it rewrites a stale doc
itself, but a genuine code violation it *flags* for a follow-up fix (via
`/charter:milestone` or a direct change) rather than rewriting code here.

## Step 1 — Locate the doc set

Resolve it with the `locate-docset` skill: read
`${CLAUDE_PLUGIN_ROOT}/skills/locate-docset/SKILL.md` and follow it, passing through any
identity token in `$ARGUMENTS` (a unit or task key). **Don't** pass a mode — let it
*infer* from the arguments, the cwd, the git branch, and what's on disk; the set must
already exist. It reports `mode`, `docs` (call it `<docs>`), `committed`, `unit`/`key`,
`exists`, `present`, and `has_roadmap`.

If it can't settle on a single existing set — a **decomposed project** (repo-root `docs/`
with no `ROADMAP.md`; reconcile a named unit's set instead) or nothing matched — stop and
ask rather than guessing.

## Step 2 — Read the docs in order (the intended picture)

Same reading discipline as `/charter:milestone`. Read `CLAUDE.md` (or `AGENTS.md`) first
for repo-wide conventions. If you're in a feature/task set **and** a repo-root project
`docs/` set exists, read its `PRODUCT.md` / `SPEC.md` / `DESIGN.md` as higher-level
context. Then, from `<docs>`: `PRODUCT.md` → `SPEC.md` → `DESIGN.md` (if present) →
`ROADMAP.md`; skim `FUTURE.md` (project/feature) or the ROADMAP's **Out of scope**
section (task) so you know what's deliberately *not* here.

Hold the SSOT precedence in mind, because it decides **which doc owns a fact** when you
update: `SPEC ▸ PRODUCT ▸ DESIGN` within the set (SPEC owns technical truth; PRODUCT owns
product intent; DESIGN owns UX — a referenced visual prototype is canonical for
appearance but loses to SPEC on data shape). Across altitudes, **project docs outrank
feature/task docs**, and `CLAUDE.md` / `AGENTS.md` owns repo-wide conventions — never
reconcile a repo-wide convention into a scope-specific doc.

## Step 3 — Survey the code as built (the real picture)

Look at what the code actually does, scoped to **what this set governs** — the SPEC's
modules, the milestones' **Touches** sections, the unit's code directory in feature mode.
Don't audit the whole repo; reconcile this set. Collect drift in two buckets:

- **Content drift — docs ↔ code.** Facts the docs assert that the code contradicts, and
  reality the code carries that the docs don't mention:
  - data model / schema, API or interface contract, config keys and **default values**,
    validation rules, state & lifecycle, module layout — wherever a concrete claim in
    SPEC/PRODUCT/DESIGN no longer matches the code;
  - code that **exists but isn't documented** — an endpoint, module, flag, or behaviour
    with no home in the docs.
- **Milestone-status drift — ROADMAP ↔ code.** For each milestone, judge landedness by
  the **batch-local** rule (the ` [done]` heading in the current `ROADMAP.md`; tags
  restart at `M1` each batch, so **never** grep all of history — a reused `M1` from an
  archived batch would match). Flag the two mismatches:
  - **Implemented but not `[done]`** — its acceptance criteria are met in the code (and,
    in git modes, a `<tag>: …` commit landed *in this batch* — scope any grep to since
    the last `Roll ROADMAP:` commit), yet the heading lacks ` [done]`.
  - **`[done]` but not built** — the heading claims done, but the acceptance criteria
    aren't actually satisfied in the code.

## Step 4 — Resolve each drift, with you

Don't batch-apply. Go item by item; for each, show **what the doc says vs. what the code
does**, and settle the direction with the user:

- **Doc is stale, the code's reality is legitimate** → **update the doc to match.** Edit
  the **owning** doc per precedence (a data-shape fact belongs in SPEC, not DESIGN), and
  ripple the change to any dependent doc that restated it. For milestone-status drift this
  way, flip the heading — add ` [done]` to a finished milestone, or remove a wrongly-set
  ` [done]`.
- **The code wrongly diverged** → **leave the doc; flag the code.** The doc is the
  intended truth and stays. Record the violation precisely (file, what it should be) and
  point the user at the fix — `/charter:milestone <tag>` if it belongs to a milestone, or
  a direct change. Don't rewrite product code here.
- **It's genuinely new scope** the docs shouldn't absorb → **defer it**, don't document
  it: `FUTURE.md` in project/feature mode, or the tracker in task mode (task docs are
  ephemeral). Documenting out-of-scope reality just launders scope creep into the SSOT.

When unsure which direction is right, ask — the user knows whether the code's deviation
was intended. Never edit a doc without sign-off; never silently overwrite settled
content.

## Step 5 — Apply, and report what's left

Apply the approved doc edits to `<docs>`. Then:

- **Committing** (project/feature): don't commit unless asked. When the user asks, commit
  the doc edits subject-prefixed `Reconcile docs: …`. A heading flipped to ` [done]` here
  is a *doc-only correction after the fact* — unlike the normal flow where `[done]` rides
  in the satisfying commit — so it's fine for it to land in a reconciliation commit.
- **Task mode**: doc edits are **not committed** — `.claude/tasks/` is gitignored — so
  they never appear in a PR. No commit step.

Close with a short summary: which docs were updated, which **code divergences were flagged**
for a follow-up fix, and anything deferred. Point at `/charter:milestone` to fix flagged
code and `/charter:status` to re-check the set.

## Things to NOT do

- **Don't rewrite product code** to make it match the docs — reconcile owns the docs;
  flag the code and fix it via `/charter:milestone` or a direct change.
- **Don't edit a doc without sign-off**, and never silently overwrite settled content.
- **Don't rubber-stamp the code** — updating the doc is for *legitimate* drift; wrong
  code gets flagged, out-of-scope reality gets deferred, not documented.
- **Don't judge milestone done-ness by grepping all of git history** — use the ` [done]`
  heading in the current `ROADMAP.md` (batch-local); scope any commit grep to since the
  last `Roll ROADMAP:`.
- **Don't reconcile repo-wide conventions into a scope-specific doc** — those belong to
  `CLAUDE.md` / `AGENTS.md`; respect `SPEC ▸ PRODUCT ▸ DESIGN` and project-over-feature
  precedence when choosing which doc owns a fact.
- **Don't commit task-mode docs** — they live in the gitignored `.claude/tasks/`.

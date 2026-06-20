---
name: status
description: Report where a Charter doc set stands — which docs are settled and where each ROADMAP milestone sits (done / next / blocked) — without changing anything. Use to see a project/feature/task's current state and what to build next.
argument-hint: "[unit or task key]"
---

Report **where a Charter doc set stands** — how complete the docs are and where each
milestone sits — and **change nothing**. This is a pure read: it resolves the set,
reads it, and tells you the state and what to build next. It never edits a doc, never
commits, never touches code. (To bring docs and code back into sync, that's
`/charter:reconcile-docs`; to settle a missing doc, `/charter:scaffold`; to build the
next milestone, `/charter:milestone`.)

## Step 1 — Locate the doc set

Resolve it with the `locate-docset` skill instead of re-deriving the rules here: read
`${CLAUDE_PLUGIN_ROOT}/skills/locate-docset/SKILL.md` and follow it, passing through any
identity token in `$ARGUMENTS` (a unit or task key). **Don't** pass a mode — let it
*infer* from the arguments, the cwd, the git branch, and what's on disk. It reports
`mode`, `docs` (the doc root — call it `<docs>` below), `committed`, `unit`/`key`,
`exists`, `present`, and `has_roadmap`.

If it can't settle on a single set, surface what it found rather than guessing:

- **Nothing matched / no set exists** — say so, and (from its report) list any
  `ROADMAP.md` files found so the user can point at one or run `/charter:scaffold`.
- **Decomposed project** (repo-root `docs/` with `has_roadmap` false — roadmapping
  lives in the per-feature sets) — report that, list the per-feature
  `*/docs/ROADMAP.md` sets found, and offer to run `status` for a named unit.

## Step 2 — Report doc completeness

From the `present`/`exists` report (or by scanning `<docs>` yourself), say which docs
are on disk versus missing — **for the docs this mode expects**, so a deliberate
absence doesn't read as a gap:

- **project** (built directly) / **feature**: PRODUCT · SPEC · [DESIGN if UI] · ROADMAP · FUTURE
- **project** (decomposed): PRODUCT · SPEC · [DESIGN if UI] · FUTURE — **no ROADMAP**
- **task**: PRODUCT · SPEC · [DESIGN if UI] · ROADMAP — **no FUTURE**

Render it as a compact line, e.g. `PRODUCT ✓ · SPEC ✓ · DESIGN — · ROADMAP ✓ · FUTURE —`,
and note any *expected* doc that's missing as the natural next `/charter:scaffold` step.
DESIGN is optional (UI surfaces only), so its absence is "not a gap" unless the set
clearly has a UI — don't flag it. There's no "draft vs settled" marker on disk, so
report **present / missing**, not settledness; don't infer a doc is unfinished.

## Step 3 — Report milestone state

This is the heart of the report (skip it when there's no ROADMAP — decomposed project,
or a set that doesn't have one yet). Read `<docs>/ROADMAP.md` and judge each milestone
by the **batch-local rule** Charter uses everywhere:

- **Landed = its heading carries ` [done]`.** The current `ROADMAP.md` holds only the
  current batch (earlier batches are archived under `roadmaps/`), and milestone tags
  **restart at `M1` each batch** — so the ` [done]` suffix is the authoritative signal.
  **Don't** decide landedness by grepping all of history (`git log --grep="M1:"`); a
  reused `M1` from an archived batch would match.

Then classify every milestone, in ROADMAP order, by tag and one-line goal:

- **Done** — heading carries ` [done]`.
- **Next** — the **first not-done milestone whose `Depends on:` prerequisites have all
  landed**. This is exactly the pick `/charter:milestone next` would make; name it
  outright and lead with it.
- **Blocked** — not done, and at least one prerequisite hasn't landed (the roadmap is
  out of order, or an earlier milestone was left unfinished). Say which prerequisite.
- **Ready, later** — not done with deps satisfied but sitting after **Next** in order.

Give a one-line summary up top — e.g. `4/6 done · next M5 · M6 blocked on M5` — then the
per-milestone breakdown. Two whole-batch cases to call out rather than bury:

- **Every milestone is done** — the batch is complete. Say so and, in project/feature
  mode, offer `/charter:roll-roadmap` to archive it and seed the next batch from FUTURE.
- **The ` [done]` headings look out of step with the code** (e.g. a milestone marked
  done whose work plainly isn't there, or finished work never marked) — you may *suspect*
  drift, but don't investigate or fix it here; point the user at
  `/charter:reconcile-docs`, which reconciles exactly that.

## Step 4 — Close with the next move

End with the concrete next action, drawn from what you found: `/charter:milestone next`
(or a specific tag) to build, `/charter:scaffold` to settle a missing doc,
`/charter:reconcile-docs` if code and docs may have drifted, or `/charter:roll-roadmap`
if the batch is complete.

## Things to NOT do

- **Don't change anything** — no doc edits, no commits, no code. `status` is read-only;
  reconciling or building is a different skill.
- **Don't judge milestone done-ness by grepping all of git history** — use the ` [done]`
  heading in the current `ROADMAP.md` (batch-local).
- **Don't guess which set applies** — if `locate-docset` can't settle on one, report what
  it found and ask.
- **Don't report settledness** — there's no draft marker on disk; report present/missing.
- **Don't flag DESIGN as missing** unless the set clearly has a UI surface — it's optional.

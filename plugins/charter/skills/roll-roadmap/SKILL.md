---
name: roll-roadmap
description: Archive a completed ROADMAP batch to roadmaps/NN-<label>.md and seed a fresh ROADMAP from FUTURE. Use when a Charter project or feature ROADMAP is finished (all milestones [done]) and you want to roll over to the next batch of work.
argument-hint: "[unit]"
disable-model-invocation: true
---

Roll a Charter doc set's ROADMAP over to its next batch: **archive** the completed
one and **seed a fresh ROADMAP from FUTURE**. A ROADMAP is *the current batch of
milestones* — when they're all done, this skill closes the batch out and starts the
next. `/charter:milestone` offers this when it completes the last milestone, but you
can also run it directly.

This is **project & feature modes only** — task sets are ephemeral and keep no
`roadmaps/`. It has side effects (rewrites docs, commits), so it's user-invoked, not
model-invoked.

## Step 1 — Locate the doc set

Resolve the set with the `locate-docset` skill: read
`${CLAUDE_PLUGIN_ROOT}/skills/locate-docset/SKILL.md` and follow it, passing any
identity token in `$ARGUMENTS` (a unit). You need `mode`, `docs` (call it
`<docs>`), and `has_roadmap`. Stop and tell the user if:

- **mode is task** — task sets keep no `roadmaps/`; there's nothing to roll over.
- **it's a decomposed project** (`<docs>` exists but `has_roadmap` is false) —
  roadmapping lives in the per-feature sets; ask which unit and resolve that instead.

## Step 2 — Confirm the batch is complete

Read `<docs>/ROADMAP.md`. Normally every milestone heading carries ` [done]` — that's
the trigger for a clean rollover. If any aren't done, this is an **early rollover**:
don't silently drop the unfinished milestones. Surface them and ask whether to

- **carry them into the next batch** (they reappear, renumbered from `M1`, in the
  fresh ROADMAP), or
- **defer them to FUTURE** (move them there as deferred ideas).

## Step 3 — Archive the completed batch

Write the batch to `<docs>/roadmaps/NN-<label>.md`:

- **`NN`** — the next integer, zero-padded to two digits. Scan `<docs>/roadmaps/` for
  existing `NN-*.md` and use `max + 1` (first archive is `01`).
- **`<label>`** — a short kebab-case description of what the batch delivered
  (e.g. `core-resolver`). Suggest one from the batch's theme; let the user confirm.
- **No dates** anywhere — not in the filename, not in the file.
- The file is a **one-line outcome header** (one sentence on what the batch shipped)
  followed by the archived `ROADMAP.md` content verbatim, milestones and their
  ` [done]` headings intact as the historical record.

Create `<docs>/roadmaps/` if it doesn't exist.

## Step 4 — Seed a fresh ROADMAP from FUTURE

Author a new `<docs>/ROADMAP.md` for the next batch, following the **ROADMAP** rules
in `/charter:scaffold` (`${CLAUDE_PLUGIN_ROOT}/skills/scaffold/SKILL.md`, the ROADMAP
entry under *Scope per doc* — Goal / Acceptance criteria / Depends on / Touches, and
the SSOT preamble). Specifically:

- **Seed from FUTURE.** Read `<docs>/FUTURE.md` and, with the user, promote the items
  that are now in scope into milestones. Don't invent work that isn't in FUTURE or the
  conversation. **Remove the promoted items from FUTURE** so it stays a deferred-only
  backlog.
- **Number milestones from `M1`.** Restarting each batch is safe: the archived batch
  is no longer in `ROADMAP.md`, and dependencies are always intra-batch (resolved by
  the ` [done]` heading in the current ROADMAP, never by grepping old commits). New
  milestones must **not** depend on archived ones.
- If FUTURE is empty (or nothing is ready to promote), say so and leave the fresh
  ROADMAP with a short note rather than fabricating milestones.

## Step 5 — Commit

Don't commit unless the user asks. When they approve, commit the archive, the fresh
`ROADMAP.md`, and the `FUTURE.md` edits **together**, subject-prefixed
`Roll ROADMAP: archive NN-<label>, seed next batch`. This commit is the batch
boundary — keeping it recognisable lets `/charter:milestone` scope any
since-last-rollover commit search if it ever needs to.

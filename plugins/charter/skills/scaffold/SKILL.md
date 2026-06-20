---
name: scaffold
description: Scaffold a single-source-of-truth doc set (PRODUCT/SPEC/[DESIGN]/ROADMAP/[FUTURE]) for a new project, feature, or task. Use when starting new work that should be specified before any code is written; code is built from these docs afterwards via /charter:milestone.
argument-hint: "[project | feature <unit> | task <key>] [DOC]"
---

Scaffold a single-source-of-truth doc set for new work. Code gets written *from*
these docs afterwards (via `/charter:milestone`), so the goal of this command is to
**settle the docs, not to write code**.

`CLAUDE.md` (or `AGENTS.md`) owns repo-wide conventions — stack, dev CLI,
testing, coding standards. These docs layer **scope-specific** truth on top of
it; they never restate repo-wide rules. There is **no per-doc-set CLAUDE.md**
(the one exception is *project* mode bootstrapping the repo's CLAUDE.md as its
final step — see below).

This command is **resumable and chat-agnostic.** The docs on disk are the only
handoff medium: a settled, signed-off doc is the input the next doc (or the next
chat) reads. You can run one doc per chat, across worktrees, days apart — every
run re-reads what already exists and picks up where the set left off.

## Step 1 — Mode & identity (settle before writing anything)

Parse `$ARGUMENTS`, or ask. Two tokens settle the run: the **mode** and (for
feature/task) the **identity**. An optional trailing **DOC** name
(`PRODUCT` / `SPEC` / `DESIGN` / `ROADMAP` / `FUTURE`) scopes the run to a single
doc.

- **Mode:**
  - **project** — the SSOT for a *whole product*. Permanent, committed.
    Doc root = repo-root `docs/`. Settle one extra thing here: will the product
    be **built directly** from the root docs, or **decomposed into features**?
    - *Built directly* (smaller products): the root set includes **ROADMAP.md**
      with normal, achievable implementation milestones — `/charter:milestone` runs them
      straight from here, exactly like feature mode.
    - *Decomposed* (larger products): the root set **omits ROADMAP.md** —
      roadmapping lives in the per-feature doc sets you scaffold later. The root
      docs stay product/architecture-level; each `/charter:scaffold feature …` carries
      its own ROADMAP.
  - **feature** — permanent SSOT for *one unit* of code (a module / package /
    domain / service). Permanent, committed. Doc root = `<unit>/docs/`, where
    `<unit>` is the directory the unit's code lives in. Ask for it, or detect it
    from the cwd; if there's no natural code home yet, fall back to
    `docs/<feature>/` at the repo root.
  - **task** — an ephemeral, often cross-cutting chunk of work (one ticket's
    worth). Throwaway: when the work lands, the docs are done. Doc root = the
    **main worktree's** `.claude/tasks/<key>/`, where `<key>` is the ticket key or
    branch/feature name. It lives in the main worktree (resolved via
    `git rev-parse --git-common-dir`) so **every worktree and chat for this task
    shares one doc set**, and it is **gitignored** — in the working tree but never
    committed and never in a PR. `locate-docset` derives the exact path (and the
    non-git fallback, which keys off the cwd and isn't shared across worktrees);
    see the resolve step below.

- **UI surface?** If the work has a meaningful UI surface, include **DESIGN.md**.
  Otherwise omit it.

Decide the doc set from mode, the UI answer, and (project mode) the build-style
answer:

| Mode | Doc root | Committed? | Docs |
|------|----------|------------|------|
| project | repo-root `docs/` | yes | PRODUCT · SPEC · [DESIGN if UI] · [ROADMAP if built directly] · FUTURE |
| feature | `<unit>/docs/` | yes | PRODUCT · SPEC · [DESIGN if UI] · ROADMAP · FUTURE |
| task | `.claude/tasks/<key>/` (main worktree, gitignored) | **no** | PRODUCT · SPEC · [DESIGN if UI] · ROADMAP |

Task mode has **no FUTURE.md** — the docs are ephemeral, so a standalone deferred
backlog would be orphaned the moment the task lands. Its scope-fence function
moves into ROADMAP's **Out of scope** section, and anything worth keeping past
the task is promoted to a **tracker ticket**, not a file.

### Resolve the doc root with `locate-docset`

Once mode + identity are settled, **resolve the doc root with `locate-docset`**
rather than deriving the path here: read
`${CLAUDE_PLUGIN_ROOT}/skills/locate-docset/SKILL.md` and follow it, passing the
**explicit** mode and identity you just settled. It returns the `docs` path plus
`exists` / `present` — which powers the resume check below. Create the doc-root
directory if it doesn't exist.

**First task set only:** when you create the very first `.claude/tasks/<key>/` set
in a repo, add `.claude/tasks/` to the repo's `.gitignore` (append the line if it's
not already there) so task docs are never committed.

### Resume: check what already exists first

Before drafting anything, use `locate-docset`'s `present` / `exists` report (or
**scan the doc root** yourself) for docs that are already written. For each that exists, read it (it's an authoritative, settled input).
Then report the set's state to the user — e.g. `PRODUCT settled ✓ · SPEC settled
✓ · DESIGN missing · ROADMAP missing` — and pick up at the **next unsettled
doc** (or the one named by the trailing DOC arg). Never silently restart a set
that's partly built; never overwrite a settled doc without the user's say-so.

### Ground feature/task sets in the project docs

If you're scaffolding a **feature or task** and a repo-root project `docs/` set
exists, read its PRODUCT/SPEC/(DESIGN) first as higher-level context — your set
defers to it on product/architecture intent (and to `CLAUDE.md`/`AGENTS.md` on
conventions). This is the same higher-altitude context `/charter:milestone` reads.

### Optional: seed from a tracker

**Only if the user explicitly asks** to pull context from their issue tracker
(Jira, Linear, GitHub Issues, Notion, …), fetch the ticket via the appropriate
MCP/CLI (title, description, acceptance criteria, subtasks) and use it to inform
the PRODUCT and ROADMAP drafts. Never do this by default — most runs are
author-from-scratch with the user.

## Step 2 — Walk the docs, settling each before the next

Order: **PRODUCT → SPEC → [DESIGN] → ROADMAP → [FUTURE]**. The set is built in
dependency order; a draft is **not** a settled input. For each doc, run this
loop:

1. **Interview — relentlessly, before drafting.** This is the heart of the
   command, especially for PRODUCT and SPEC. Do not transcribe what the user
   says into a doc; *interrogate* it. Surface the things they haven't
   considered: scope boundaries, edge and unhappy paths, hidden assumptions,
   dependencies, non-functional requirements, trade-offs, and **blockers /
   contradictions**. Push back when something is hand-waved or inconsistent.
   - Keep it productive, not exhausting: **batch** questions, lead with the
     decisions that actually change the doc, and separate **blockers**
     (resolve now) from **open questions** (park them visibly in the doc and
     move on). Honour an explicit "good enough" from the user.
   - Question banks to draw from:
     - **PRODUCT:** who is this for (precisely)? what problem, why now? what is
       explicitly *not* in scope? what does success look like? what's the
       user-visible before/after? what happens on the unhappy path? what does it
       replace or affect?
     - **SPEC:** data model / schema; API or interface contract; state &
       lifecycle; concurrency, failure, retries, idempotency; migration &
       backward compatibility; authz & security; performance & scale;
       observability; testing approach; what existing code it touches;
       build-vs-reuse; sequencing/dependencies; the single riskiest unknown.
2. **Draft** the doc — only once no open question would still change it.
3. **Sign-off.** The bar is that **both** of you are satisfied the doc covers
   everything and the plan is solid — not merely "looks fine." Do not start the
   next doc until the current one is agreed.
4. **Write** the file at `<doc-root>/<NAME>.md`.

### The preamble every doc carries

Each doc opens with a blockquote that makes the set self-enforcing. Build it from:

- **Doc set / read order** — list the docs *actually in this set*, in read order,
  linking the others; bold the current one. (e.g. for a task with no DESIGN:
  `PRODUCT → SPEC → ROADMAP`.)
- **"This is the SSOT for *X*"** — one line naming what this doc owns
  (PRODUCT = *what it is*; SPEC = *how it's built*; DESIGN = *how it looks &
  behaves*; ROADMAP = *what to build next*; FUTURE = *what it can become*).
- **Conflict precedence** — `SPEC ▸ PRODUCT ▸ DESIGN`. If DESIGN references a
  visual prototype, extend to `SPEC ▸ PRODUCT ▸ DESIGN ▸ <prototype>/`. In
  feature/task mode, also note these docs defer to the **project docs** (if a
  repo-root `docs/` set exists) and to **CLAUDE.md** for repo-wide conventions.
- **Scope guard** — "describes the work as built/planned; anything else is out of
  scope or deferred." For task docs, point deferrals at the tracker; for
  project/feature docs, at FUTURE.

### Scope per doc

- **PRODUCT.md** — what the work is and does, for whom, and why it exists.
  **What-it-is/does only:** no tech detail, no done-criteria, no dates/phases.
  - project: the product's intent — problem, users, core concepts, primary
    flows, governing rules.
  - feature: the unit's product intent — problem, users, concepts, flows, rules.
  - task: what this change delivers and why, in plain terms — the before/after
    and the user-visible behaviour. Still "the product view," scoped to the
    change, not the whole app.
- **SPEC.md** — the "how it's built": architecture, data model / schema, module
  layout, API/interface contract, operational rules, validation, testing
  approach. The technical truth code is written from. In feature/task mode,
  defer to CLAUDE.md for repo-wide conventions and only capture what's specific
  to this work. In **project mode** there may be no CLAUDE.md yet — so the
  project SPEC carries the repo-wide technical truth, and conventions are
  distilled out into CLAUDE.md at the end (see below).
- **DESIGN.md** (only if UI) — the UX layer. A **thin bridge**, not a pixel
  re-description: screen→route/component map, interaction rules over time, and
  the mapping from any prototype to the app's component library / design system.
  If a visual prototype exists, say it's canonical for appearance and SPEC wins
  on data shape.
- **ROADMAP.md** — milestones in dependency order, forward-looking only. Each
  milestone is a **vertical slice** and is defined by:
  - **Goal:** one line on what gets delivered.
  - **Acceptance criteria:** the testable bullets that prove it's done.
  - **Depends on:** prerequisite milestones (or "none").
  - **Touches:** which PRODUCT/SPEC/[DESIGN] sections it implements (by section
    *name*, never number).

  In **project mode**, ROADMAP exists only when the product is **built directly**
  (smaller products); its milestones are then normal implementation slices, just
  like feature mode. Larger, **decomposed** projects have no root ROADMAP —
  milestones live in the per-feature sets instead. Milestones are satisfied by a
  commit subject-prefixed with the tag (`M2: …`, optionally after a ticket key) and
  their heading gets ` [done]` when complete — this is what `/charter:milestone` reads.
  In **task mode**, add a short **Out of scope** section here as the scope-fence
  (replacing FUTURE).
- **FUTURE.md** (project & feature mode only) — deferred ideas, one line each
  under a blockquote intro, no priority/timeframe tags. PRODUCT/SPEC/DESIGN/
  ROADMAP must **not** reference these, so current work stays focused. Don't
  invent ideas — only what genuinely surfaced as deferred-and-out-of-scope. May
  be left near-empty.

### Project mode only — create CLAUDE.md last

Once the whole doc set is settled, **distil repo-wide conventions into CLAUDE.md
as the final step** (only in project mode):

- It's **doc-derived** — pull the conventions that surfaced during the interview
  (mostly in SPEC): stack + versions, dev/build/test commands, testing rules,
  coding standards, directory layout. Greenfield projects have no code to analyse
  yet; once code exists, `/init` can enrich it.
- **Boundary, to avoid duplication:** CLAUDE.md = conventions / how-we-work;
  project SPEC = architecture / data model / technical decisions-and-rationale.
  Once CLAUDE.md exists, the docs defer to it for conventions.
- **Never overwrite an existing CLAUDE.md.** If one already exists, defer to it
  and at most *offer* to augment it — don't clobber it.

## After scaffolding

Tell the user the doc set is ready and that work proceeds milestone by milestone
via `/charter:milestone <M> [unit or task key]`. For **task mode**, remind them the docs
live in the (gitignored) `.claude/tasks/<key>/` and won't be committed or appear in
the PR.

## Things to NOT do

- Don't add a "Last updated" or preamble line beyond the SSOT blockquote.
- Don't fabricate context or invent content. Only what's sourced from the
  conversation, the user, the tracker (if requested), or existing repo state.
- Don't cite docs by section number — use section names.
- Don't write a CLAUDE.md *for the doc set*. The only CLAUDE.md you ever write is
  the repo's, in project mode, as the final step — and never over an existing one.
- Don't create FUTURE.md in task mode.
- Don't commit task-mode docs — they live in the **gitignored** `.claude/tasks/`,
  so they're in the working tree but never staged and never in a PR.
- Don't restart or overwrite a doc set that's already partly built — resume from
  what's on disk.
- Don't start the next doc before the current one is signed off — the set is
  built in dependency order, and a draft is not a settled input.
- Don't draft a doc before the interview has converged — settle the open
  questions (or park them explicitly) first.
- Don't write code in this command. Scaffolding ends at agreed docs; `/charter:milestone`
  starts the build.

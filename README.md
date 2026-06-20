# Charter

Doc-driven development for any repo, packaged as a Claude Code plugin.

Charter scaffolds a single-source-of-truth **doc set** — PRODUCT, SPEC, optional
DESIGN, ROADMAP, FUTURE — for a project, feature, or task, then helps you build it
**milestone by milestone, straight from those docs**. Code is written *from* the
docs, not the other way around.

## Install

```
/plugin marketplace add theelgreco/charter
/plugin install charter@charter
```

Then, in any repo:

```
/charter:scaffold project          # settle the docs for a new product
/charter:milestone M1              # build the first milestone from them
/charter:milestone next            # …or let Charter pick the next unblocked one
```

### Try it locally (before publishing)

Load the plugin directly for one session:

```
claude --plugin-dir ./plugins/charter
```

…or add this checkout as a local marketplace and install from it:

```
/plugin marketplace add ./          # run from the repo root
/plugin install charter@charter
/charter:scaffold
```

## Skills

| Skill | What it does |
|-------|--------------|
| `/charter:scaffold` | Interview-driven authoring of a doc set (project / feature / task), settling each doc before the next. |
| `/charter:milestone` | Execute one ROADMAP milestone from a doc set — named (`M2`) or auto-selected (`next`); reads the docs, checks dependencies, implements only that milestone. |
| `/charter:locate-docset` | Resolve which doc set applies here (project / feature / task) and where its docs live. The shared resolver the other skills defer to; also runnable directly. |
| `/charter:roll-roadmap` | Archive a completed ROADMAP batch to `roadmaps/NN-<label>.md` and seed a fresh ROADMAP from FUTURE. Offered by `milestone` when a batch finishes; project / feature only. |

## Modes

| Mode | Doc root | Committed? |
|------|----------|------------|
| **project** | repo-root `docs/` | yes |
| **feature** | `<unit>/docs/` | yes |
| **task** | host path (ephemeral) | no |

## Status

The two core skills plus the shared `locate-docset` resolver are installable.
Recently landed: `locate-docset`; the docs-as-SSOT rule written into `.claude/rules/`
on scaffold (versioned, so existing repos can be offered updates); task docs relocated
into the repo's auto-gitignored `.claude/tasks/`; `/charter:milestone next`, which
auto-picks the next unblocked milestone; and `/charter:roll-roadmap`, the ROADMAP
lifecycle — archive a completed batch to `roadmaps/NN-<label>.md` and seed the next
from FUTURE. Planned next:

- `reconcile`, `status`, `promote`

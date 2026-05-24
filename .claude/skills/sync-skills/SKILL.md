---
name: sync-skills
description: "Sync skills from the fabricir workbench-next repo into the current project's .agents/skills/ directory. Wraps workbench/scripts/sync-skills.sh with filter presets (kanban, solo, sdlc, governance, all)."
argument-hint: "[FILTER] — optional filter preset (default: kanban). One of: kanban, solo, sdlc, governance, all"
tags: [tooling]
---

# Sync Skills

Wraps the canonical script at `workbench/scripts/sync-skills.sh` in the fabricir workbench-next repo. The script copies filtered skills from fabricir's `.agents/skills/` (source of truth) into the current project's `.agents/skills/` (target), and prunes fabricir-managed locals that fall outside the filter.

## Locate workbench-next

Find the fabricir workbench-next repo by checking these locations in order:

1. `../../exercir/workbench-next` (standard side-by-side layout for braighter-io projects)
2. `$HOME/development/projects/exercir/workbench-next`
3. The path in the project's `CLAUDE.md` (grep for `workbench-next`)

Override via the `FABRICIR` env var if the layout differs. If not found, stop and ask the user for the path.

## Run the sync

From the consumer project's root (the directory containing `.agents/skills/`):

```bash
{WORKBENCH_NEXT}/workbench/scripts/sync-skills.sh {FILTER}
```

The script reads its source from the resolved workbench-next location and writes to `$(pwd)/.agents/skills/`. Prune is on by default; add `noprune` as a second arg to keep extras.

## Filter presets

| Filter | What it includes |
| --- | --- |
| `kanban` (default) | SDLC skills compatible with continuous-flow Kanban — story-by-story, WIP-limited (excludes sprint-batching skills) |
| `solo` | Day-one subset: SDLC skills that work without sprint/board/backlog infrastructure |
| `sdlc` | Full SDLC spine: kanban + sprint-batching (plan-sprint, sprint-runner, sprint-retro, all-sprints-runner) |
| `governance` | Quality gates routed via the concierges |
| `all` | Everything fabricir ships (39 skills as of 2026-05-15) |

## After sync

1. Report what was synced (count + list)
2. Report any pruned skills (formerly managed by fabricir, now outside the filter)
3. Report any retained non-fabricir skills (e.g. `nx-*` from the Nx CLI — those have a different source of truth and are left alone)

## Examples

```bash
# From the consumer project root (default: kanban, prune on)
../../exercir/workbench-next/workbench/scripts/sync-skills.sh

# Full SDLC spine
../../exercir/workbench-next/workbench/scripts/sync-skills.sh sdlc

# Everything, keep extras
../../exercir/workbench-next/workbench/scripts/sync-skills.sh all noprune

# Custom workbench-next location
FABRICIR=/custom/path/workbench-next /custom/path/workbench-next/workbench/scripts/sync-skills.sh kanban
```

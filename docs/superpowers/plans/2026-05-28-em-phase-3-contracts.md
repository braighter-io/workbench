# EM Phase 3: move contracts + workflows — Implementation Plan

> Subagent-driven.

**Goal:** Move all 69 contract files + 2 workflow utility files from `eyecatchers-core` to `design-system-core`. Repath eyecatchers-angular + showcase contract imports. eyecatchers-core becomes a near-empty shell (still conforming) — its actual deletion is Phase 5.

**Architecture:** File moves via `git mv` (preserves rename detection) + barrel updates + import repaths. api-snapshots regenerate: `eyecatchers-core` shrinks to near-empty; `design-system-core` grows by 69+2 entries. **Consumer-internal change** for components (`eyecatchers-angular` + showcase): same symbols, different import source.

**Repo/env:** Same caveats — branch-recheck before commit; scratch files never staged; STOP on `Cannot find module @de-braighter/...`.

---

### Task 1 — Branch + baseline

```bash
git checkout main && git pull --ff-only
git checkout -b chore/em-phase-3-contracts
npm run ci:local  # baseline exit 0
```

### Task 2 — Move contracts

```bash
mkdir -p libs/design-system-core/src/public/contracts
git mv libs/eyecatchers-core/src/public/contracts/* libs/design-system-core/src/public/contracts/
# Likewise workflows (2 files):
mkdir -p libs/design-system-core/src/public/workflows
git mv libs/eyecatchers-core/src/public/workflows/* libs/design-system-core/src/public/workflows/
```

### Task 3 — Update barrels

- `libs/design-system-core/src/index.ts`: add `export * from './public/contracts/<name>'` lines for each moved contract (one per file) + the workflow exports.
- `libs/eyecatchers-core/src/index.ts`: REMOVE the contracts + workflow export lines. The barrel becomes essentially empty (may need `export {};` so the file is a valid ES module).

### Task 4 — Repath imports

- `libs/eyecatchers-angular/src/public/**/*.ts`: any `from '@de-braighter/eyecatchers-core'` → `from '@de-braighter/design-system-core'`. (Phase 2 already moved the math/RM imports; this phase moves the remaining contract imports.)
- `apps/showcase/src/**/*.ts`: contract/type imports from `'@de-braighter/eyecatchers-core'` → `'@de-braighter/design-system-core'`. Component imports from `'@de-braighter/eyecatchers-angular'` STAY — those are Phase 4.

Verification grep: after edits, `grep -rn "from '@de-braighter/eyecatchers-core'"` across `libs/eyecatchers-angular/src` AND `apps/showcase/src` → **zero hits** (eyecatchers-core is now empty of contracts/workflows, so any remaining import would dangle).

### Task 5 — Build, regenerate snapshots, gate

```bash
npx nx run-many -t build
npm run api:update
git diff --stat libs/eyecatchers-core/etc libs/design-system-core/etc
# Expected: eyecatchers-core SHRINKS to near-empty; design-system-core GROWS by ~71 exports.
# eyecatchers-angular snapshot unchanged (consumer-internal).
npx nx run-many -t api-check
npm run ci:local  # full gate exit 0
```

### Task 6 — Commit + push + PR

Branch-recheck, targeted commit:
```bash
git add -u
# Plus explicit add for new dirs:
git add libs/design-system-core/src/public/contracts libs/design-system-core/src/public/workflows libs/design-system-core/src/index.ts libs/design-system-core/etc/design-system-core.api.md libs/eyecatchers-core/etc/eyecatchers-core.api.md
git commit -m "refactor(merge): move 69 contracts + 2 workflows to design-system-core (EM Phase 3)"
git push -u origin chore/em-phase-3-contracts
```
PR + issue per the pattern.

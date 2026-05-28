# EM Phase 2: math/tokens graduation — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development.

**Goal:** Delete the (already `@deprecated`) duplicated math/tokens in `eyecatchers-core`; consolidate to `design-system-core` as the single source. Update `eyecatchers-angular` imports + the TsWriter to target one core.

**Architecture:** Pure deletion + import repath. eyecatchers-core's public surface SHRINKS (api-snapshot regenerates intentionally); eyecatchers-angular's behavior unchanged (same symbols, different import source). Now possible because Phase 1 retired the scope wall.

**Tech Stack:** nx, postcss/tokens-compiler, api-extractor.

**Repo/env:** `D:\development\projects\de-braighter\layers\design-system\`, branch off `main` (post Phase 1). Branch-recheck before commit. If `Cannot find module '@de-braighter/...'`, STOP + report BLOCKED.

---

## Files

**Delete (eyecatchers-core, 25 files):**
- `libs/eyecatchers-core/src/public/math/*` — 17 files
- `libs/eyecatchers-core/src/public/tokens/*` — 6 files (including `.generated.ts` + wrapper `.ts`)
- `libs/eyecatchers-core/src/public/math/reduced-motion.ts` + `motion-loop.ts` (counted in the 17 above)

**Modify:**
- `libs/eyecatchers-core/src/index.ts` — drop ~25 export lines for the deleted modules.
- `tools/tokens-compiler/build-ts.mjs` — drop the `eyecatchers-core` target from the LIBS loop.
- `tools/tokens-compiler/check-ts.mjs` — drop the `eyecatchers-core` target from its verification loop (parallel to build-ts).
- `libs/eyecatchers-core/etc/eyecatchers-core.api.md` — regenerated (shrinks substantially; deliberately committed).
- `libs/eyecatchers-angular/src/public/**/*.ts` — repath any `from '@de-braighter/eyecatchers-core'` for math/tokens/RM symbols to `from '@de-braighter/design-system-core'`. (Contracts stay in eyecatchers-core for now; that's Phase 3.)

---

### Task 1 — Branch + baseline

```bash
cd /d/development/projects/de-braighter/layers/design-system
git checkout main && git pull --ff-only
git checkout -b chore/em-phase-2-math-tokens
git branch --show-current  # MUST print: chore/em-phase-2-math-tokens
npm run ci:local  # baseline exit 0
```

### Task 2 — Update tokens-compiler to target only design-system-core

In `tools/tokens-compiler/build-ts.mjs`: locate the array of target libs (e.g., `const LIBS = ['design-system-core', 'eyecatchers-core']`). Reduce to `['design-system-core']`.

In `tools/tokens-compiler/check-ts.mjs`: same change to its lib-iteration array.

Run `npm run tokens:check` — must PASS (the check now only looks at design-system-core; eyecatchers-core's copies are still there at this point and unverified, but that's fine because they're about to be deleted).

### Task 3 — Delete the duplicated math + tokens in eyecatchers-core

```bash
git rm -r libs/eyecatchers-core/src/public/math
git rm -r libs/eyecatchers-core/src/public/tokens
```
(Removes 17 math files + 6 tokens files = 23 files.)

### Task 4 — Update eyecatchers-core barrel

Edit `libs/eyecatchers-core/src/index.ts`: delete every `export * from './public/math/...'` and `export * from './public/tokens/...'` line. Other exports (contracts/workflows) stay.

### Task 5 — Repath eyecatchers-angular imports

For every file in `libs/eyecatchers-angular/src/public/**/*.ts` that imports math/tokens/RM symbols (e.g. `createFrameLoop`, `browserFrameHost`, `createMotionLoop`, `prefersReducedMotion`, `onReducedMotionChange`, `damp`, `EASING`, `PALETTE`, `MOTION`, `clamp`, `lerp`, `vec2`-etc., bezier helpers, etc.), change the import source from `'@de-braighter/eyecatchers-core'` to `'@de-braighter/design-system-core'`. Imports of CONTRACTS (e.g. `MagneticCursorContract`) STAY pointing at `'@de-braighter/eyecatchers-core'` — those are Phase 3.

Practical approach: grep for `from '@de-braighter/eyecatchers-core'` in eyecatchers-angular; for each hit, read the imported symbols; if any are math/tokens/RM, split the import into two lines (one from each core), or move all the non-contract symbols to design-system-core if no contracts are in that import.

Verification grep: after edits, `grep -rn "from '@de-braighter/eyecatchers-core'" libs/eyecatchers-angular/src` should show ONLY contract-name imports (e.g. `import { GravityFieldContract } from '@de-braighter/eyecatchers-core'`).

### Task 6 — Build, regenerate snapshots, gate

```bash
npx nx run-many -t build
npm run api:update   # regenerates etc/*.api.md
git diff --stat libs/eyecatchers-core/etc  # expect: shrinks substantially
git diff --stat libs/design-system-core/etc libs/eyecatchers-angular/etc  # expect: unchanged
npx nx run-many -t api-check  # must PASS
```

If `tokens:check` fails because `check-ts.mjs` still references eyecatchers-core paths, finish updating it. If `Cannot find module @de-braighter/...`, STOP + report.

### Task 7 — Full ci:local

```bash
npm run ci:local
```
Expected: EXIT 0.

### Task 8 — Branch-recheck + commit

```bash
[ "$(git branch --show-current)" = "chore/em-phase-2-math-tokens" ] || { echo "WRONG BRANCH"; exit 1; }
git add -A libs/eyecatchers-core libs/eyecatchers-angular tools/tokens-compiler
git status --short  # confirm scratch files not staged
git commit -m "refactor(merge): graduate math/tokens to design-system-core (EM Phase 2)

Deletes the @deprecated duplicates of math/, tokens/, reduced-motion, and
motion-loop from eyecatchers-core. The tokens-compiler emits the dark palette
only into design-system-core now. eyecatchers-angular components repath their
math/tokens/RM imports from @de-braighter/eyecatchers-core to
@de-braighter/design-system-core. Contracts stay in eyecatchers-core for Phase 3.

api-extractor snapshot for eyecatchers-core deliberately shrinks; design-system-core
and eyecatchers-angular snapshots unchanged (no API surface migration there).

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

### Task 9 — Push + PR

```bash
git push -u origin chore/em-phase-2-math-tokens
gh issue create --title "EM Phase 2: graduate math/tokens to design-system-core" --body "Story for Phase 2 of the em-charter. Deletes deprecated duplicates; repath eyecatchers-angular imports."
gh pr create --base main --title "refactor: graduate math/tokens to design-system-core (EM Phase 2)" --body "Phase 2 of the em-charter. Closes #NN. api-snapshot for eyecatchers-core deliberately shrinks; others unchanged."
```

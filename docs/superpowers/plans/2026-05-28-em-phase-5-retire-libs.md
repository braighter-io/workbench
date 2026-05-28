# EM Phase 5: retire eyecatchers libs + packages — Retrospective Plan

> **Shipped:** de-braighter/design-system PR #115 (merge commit `cb26fe6`).
>
> **Merging this PR completed the entire eyecatchers→design-system merge.**

**Goal:** Delete `eyecatchers-core` and `eyecatchers-angular` entirely; retire the `@de-braighter/eyecatchers-*` packages; remove their `tsconfig.base.json` path aliases + `package.json` script entries; light-touch update of misleading docs.

**Outcome:**
- `git rm -r` of `libs/eyecatchers-core/` and `libs/eyecatchers-angular/` (~20 tracked files deleted total — both were near-empty shells after Phase 4c).
- `tsconfig.base.json` — removed both `@de-braighter/eyecatchers-*` path aliases.
- `package.json` — removed eyecatchers entries from `build:libs` / `publish:libs`.
- `tools/api-update.mjs` — trimmed `LIBS` list from 5 entries (incl. 2 eyecatchers) to 3 (`design-system-core`, `design-system-angular`, `design-system-angular-forms`).
- `tools/test-reduced-motion.mjs` — `distRoot` updated from `dist/libs/eyecatchers-core/...` to `dist/libs/design-system-core/...`.
- `scripts/publish-libs.sh` — rewritten for the 3 published design-system packages.
- `docs/publishing.md` — rewritten; eyecatchers retirement noted.
- `CLAUDE.md` — Libraries table updated (now 4 libs instead of 6); eyecatchers retirement noted.
- Final code/config grep: zero `@de-braighter/eyecatchers` references (only a historical comment in `publish-libs.sh` + the private workspace-root `package.json` name remain — both intentional / non-functional).
- `lib-conformance`: now 4 libs (was 6) — automatically picked up from the `libs/*/` glob.
- `ci:local` exit 0.

**End state:** the design-system layer ships **4 libs** (`design-system-css`, `design-system-core`, `design-system-angular`, `design-system-angular-forms`) instead of 6. The `@de-braighter/eyecatchers-*` npm scopes are retired.

**Charter correction:** the charter said "3 libs after merge" — it forgot `design-system-angular-forms`. Real count is **4 libs**; the merge eliminated the 2 eyecatchers libs but `design-system-angular-forms` was always its own lib.

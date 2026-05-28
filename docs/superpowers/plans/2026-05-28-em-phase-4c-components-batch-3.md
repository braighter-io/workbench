# EM Phase 4c: move components batch 3 of 3 — Retrospective Plan

> **Shipped:** de-braighter/design-system PR #113 (merge commit `793d16c`).

**Goal:** Move the final 20 eyecatcher components from `eyecatchers-angular` to `design-system-angular`, including the `segmented-control` / `tabbed-panel` sibling-dep cluster (tabbed-panel imports `../segmented-control/...`). After this phase, **zero `@de-braighter/eyecatchers-angular` imports remain anywhere in the cluster**.

**Components moved:** pick-list, pressure-commit, region-globe, relation-field, rhythm-ring, sankey, season-stretch-hero, segmented-control, service-map, sparkle-hover, spotlight-card, tabbed-panel, text-roll, text-scramble, tilt-card, tool-bloom, type-writer, venn-picker, week-mosaic-hero, workflow-map.

**Outcome:**
- 20 components moved.
- 21 showcase files repathed (incl. `playground-tabs.component`).
- Final cluster grep: zero `@de-braighter/eyecatchers-angular` imports remain in `apps/showcase/src` or `libs/`.
- Sibling-dep grep: `from '../segmented-control'` → 1 hit (tabbed-panel), `from '../number-flow'` → 5 hits (Phase 4a cluster preserved).
- api-snapshots: `eyecatchers-angular` −1115 lines (shrunk to near-empty); `design-system-angular` +1107 lines.
- `ci:local` exit 0.
- Added `tsdoc-malformed-html-name` suppression to `design-system-angular/api-extractor.json` to match the suppression eyecatchers-angular had (the rhythm-ring component carries that warning pattern).

**Pattern:** same as 4a/4b. After this batch, `eyecatchers-angular` is a near-empty shell (only `src/public/stub.ts` + an empty barrel) awaiting deletion in Phase 5.

# EM Phase 4b: move components batch 2 of 3 — Retrospective Plan

> **Shipped:** de-braighter/design-system PR #111 (merge commit `539bd57`).

**Goal:** Move the alphabetic-next 25 of 70 eyecatcher components from `eyecatchers-angular` to `design-system-angular`. No sibling-dep clusters in this batch.

**Components moved:** glitch-text, glow-border, gradient-text, gravity-field, heartbeat, horizon-chart, inertial-dial, latency-heatmap, log-waterfall, magnetic-button, magnetic-cursor, magnetic-tag-cloud, marble-stream, marquee, match-poster-hero, morph-blob, multi-handle-elastic-slider, muscle-map, noise-field, observer-pulse, orbit-picker, orbit-ring, paper-plane, phase-curve-picker, phase-ribbon.

**Outcome:**
- 25 components moved (all `R100` renames).
- 27 showcase files repathed (incl. `home.page.ts` split-import: moved 8 names to design-system-angular, kept 6 in eyecatchers-angular for 4c).
- api-snapshots: `eyecatchers-angular` −1118 lines; `design-system-angular` +1246 lines.
- `ci:local` exit 0.
- Per Phase 4a's lesson: precise per-class-name edits, no `perl` sweeps.

**Same pattern as 4a.**

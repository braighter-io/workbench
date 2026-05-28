# EM Phase 4a: move components batch 1 of 3 — Retrospective Plan

> **Shipped:** de-braighter/design-system PR #109 (merge commit `459f799`).

**Goal:** Move the first 25 of 70 eyecatcher components from `eyecatchers-angular` to `design-system-angular`, including the `number-flow` sibling-dep cluster (6 components must move together so `../number-flow/*` relative imports survive). Showcase imports for the moved components repath in-batch.

**Components moved:** ambient-toggle, anatomy-ticker-hero, aurora-background, beeswarm-timeline, body-map, calendar-heatmap, chord-diagram, chord-wheel, clock-dial, command-palette, confetti-burst, constellation, count-ticker, cursor-trail, date-picker, date-range-picker, distribution-sketch, equalizer-bars, flame-graph, gauge, gesture-glyph, glow-slider, heart-pulse, number-flow, orbit-dial.

**Outcome:**
- 25 components moved (all `R100` renames).
- 26 showcase files repathed.
- 5 `../number-flow` sibling imports verified inside `design-system-angular/src/public/`.
- api-snapshots: `eyecatchers-angular` −1280 lines; `design-system-angular` +1278 lines (consumer-internal public-API surface preserved per-component).
- `ci:local` exit 0.

**Mid-run finding:** the implementer's bulk `perl` rewrite accidentally repathed 3 unrelated hero pages (match-poster-hero, season-stretch-hero, week-mosaic-hero); caught at build-time, reverted before commit. Lesson recorded for 4b/4c: precise per-class-name edits only, no broad sweeps.

**Pattern:** `git mv libs/eyecatchers-angular/src/public/<c> libs/design-system-angular/src/public/<c>` per component; barrel adds in `design-system-angular/src/index.ts`, removes in `eyecatchers-angular/src/index.ts`; showcase imports repath; api-snapshots regenerate intentionally.

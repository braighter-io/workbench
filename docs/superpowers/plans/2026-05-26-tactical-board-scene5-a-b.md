# Tactical-board (Scene 5) — S5-A + S5-B Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship the `<db-pitch>` SVG design-system brick (cross-repo, published) and a **read-only live tactical board** at `/coach/board` that renders the match lineup on a full pitch and re-renders from the live SSE match-event stream.

**Architecture:** S5-A adds a chrome-only, frame-parameterized `<db-pitch>` brick to `@de-braighter/design-system-angular`, publishes it to GitHub Packages, and wires it as a new exercir dependency. S5-B builds the read-only board in `pack-football-ui` using the four-layer pattern (pure layout → standalone OnPush scene over `<db-pitch frame="full">`), plus a `MatchEventClient` (EventSource) that re-renders on `FormationChanged.v1` / `SubstitutionMade.v1`. No board-driven writes, annotations, or snapshots in this plan (those are S5-C…E).

**Tech Stack:** TypeScript, Angular (standalone, signals, OnPush), Nx, Vitest, SVG, SSE/EventSource. Spec: `docs/superpowers/specs/2026-05-26-tactical-board-scene5-design.md`.

**Two repos:**
- **design-system:** `D:/development/projects/de-braighter/layers/design-system` (own git repo; remote `de-braighter/design-system`). Build `nx build design-system-angular`; test `nx test design-system-angular` (vitest). Bricks: `db-*` selector, standalone, OnPush, signal `input()`, token colours, barrel `src/index.ts`. Mirror the existing `libs/design-system-angular/src/lib/pitch-diagram/pitch-diagram.component.ts`.
- **exercir:** `D:/development/projects/de-braighter/domains/exercir` (own git repo; remote `de-braighter/exercir`). Tests `npx nx test <project> -t "<name>"`; gate `npm run ci:local`; `.js` import extensions; UI must NOT import the NestJS `pack-football` barrel (use the `wire-schemas.ts` mirror).

**Conventions:** Branch off `main` in each repo; PR-gated; local gate is the bar (remote Actions billing-blocked). Commit messages via a temp file OUTSIDE the repo (`git commit -F`, then delete; stage only listed files). TDD: failing test → run/fail → implement → run/pass → commit.

**Registry / publish (S5-A):** `@de-braighter` → GitHub Packages (`npm.pkg.github.com`), auth via `$GITHUB_TOKEN`. Publishing needs a token with `write:packages` — a **credentialed step** (if the executing agent lacks the token, ask the user to run `! $env:GITHUB_TOKEN='…'; npm run publish:libs` from the design-system repo, or use the local-tarball fallback in Task A4).

---

## File Structure

**design-system repo (S5-A):**
- Create: `libs/design-system-angular/src/lib/pitch/pitch.component.ts` — `<db-pitch>` brick.
- Create: `libs/design-system-angular/src/lib/pitch/pitch.spec.ts`
- Create: `libs/design-system-angular/src/lib/pitch/index.ts` — barrel.
- Modify: `libs/design-system-angular/src/index.ts` — export the brick.
- Modify: `libs/design-system-angular/package.json` — version bump `1.4.0 → 1.5.0`.

**exercir repo (S5-A wiring + S5-B):**
- Modify: `package.json` — add `@de-braighter/design-system-angular` + `@de-braighter/design-system-core` deps.
- Create: `libs/pack-football-ui/src/lib/generation/tactical-board-template.ts` — full-pitch (100×120) 4-3-3 slot coords + `.spec.ts`.
- Create: `libs/pack-football-ui/src/lib/generation/tactical-board.types.ts` — `TacticalBoardView` + element types.
- Create: `libs/pack-football-ui/src/lib/generation/tactical-board-layout.ts` — pure `layoutTacticalBoard` + `.spec.ts`.
- Create: `libs/pack-football-ui/src/lib/data/match-event.client.ts` — SSE consumer + `.spec.ts`.
- Create: `libs/pack-football-ui/src/lib/coach/ui/coach-tactical-board.component.ts` — the scene + `.spec.ts`.
- Modify: `libs/pack-football-ui/src/lib/shell/fc-workspace.routes.ts` — `/coach/board` route.
- Modify: `libs/pack-football-ui/src/lib/shell/fc-workspace.types.ts` — nav entry + theme.
- Modify: `libs/pack-football-ui/src/index.ts` — exports.

---

## Task A1: `<db-pitch>` brick (design-system)

**Repo:** `layers/design-system`. Branch: `feat/db-pitch-brick`.
**Files:** Create `libs/design-system-angular/src/lib/pitch/{pitch.component.ts,pitch.spec.ts,index.ts}`; Modify `src/index.ts`.

- [ ] **Step 1: Write the failing spec** `pitch/pitch.spec.ts` (mirror the vitest+TestBed harness used elsewhere in design-system-angular; if no component spec exists yet, this is the first — use Angular `TestBed.createComponent`):

```ts
import { describe, it, expect, beforeEach } from 'vitest';
import { TestBed } from '@angular/core/testing';
import { Component } from '@angular/core';
import { DbPitch } from './pitch.component.js';

@Component({
  standalone: true,
  imports: [DbPitch],
  template: `<db-pitch [frame]="frame"><circle data-overlay cx="50" cy="60" r="3" /></db-pitch>`,
})
class Host { frame: 'mini' | 'half' | 'full' = 'full'; }

describe('DbPitch', () => {
  beforeEach(() => TestBed.configureTestingModule({ imports: [Host] }));

  it('renders an svg whose viewBox matches the frame', () => {
    const f = TestBed.createComponent(Host);
    f.detectChanges();
    const svg = (f.nativeElement as HTMLElement).querySelector('svg')!;
    expect(svg.getAttribute('viewBox')).toBe('0 0 100 120'); // full
  });

  it('mini and half frames use their own viewBox', () => {
    const f = TestBed.createComponent(Host);
    f.componentInstance.frame = 'mini'; f.detectChanges();
    expect((f.nativeElement as HTMLElement).querySelector('svg')!.getAttribute('viewBox')).toBe('0 0 100 60');
    f.componentInstance.frame = 'half'; f.detectChanges();
    expect((f.nativeElement as HTMLElement).querySelector('svg')!.getAttribute('viewBox')).toBe('0 0 100 64');
  });

  it('projects overlay content into the svg coordinate space', () => {
    const f = TestBed.createComponent(Host);
    f.detectChanges();
    expect((f.nativeElement as HTMLElement).querySelector('svg [data-overlay]')).toBeTruthy();
  });

  it('marks the field chrome aria-hidden (decorative)', () => {
    const f = TestBed.createComponent(Host);
    f.detectChanges();
    expect((f.nativeElement as HTMLElement).querySelector('[data-chrome]')!.getAttribute('aria-hidden')).toBe('true');
  });
});
```

- [ ] **Step 2: Run — expect FAIL.** `cd /d/development/projects/de-braighter/layers/design-system && npx nx test design-system-angular -t "DbPitch"`

- [ ] **Step 3: Implement** `pitch/pitch.component.ts` (mirror `pitch-diagram.component.ts` conventions: `db-` selector, standalone, OnPush, signal input, token colours, `@maturity experimental`). Frame → viewBox + chrome. Project overlays via `<ng-content>` inside the `<svg>`:

```ts
import { ChangeDetectionStrategy, Component, computed, input } from '@angular/core';

export type PitchFrame = 'mini' | 'half' | 'full';

/**
 * `<db-pitch>` — SVG football-pitch primitive. Renders field chrome only;
 * scene overlays project as child SVG via <ng-content> into the same
 * coordinate space. `frame` selects the viewBox + which chrome renders.
 *
 * @maturity experimental
 * @since 1.5.0
 * Consumed by pack-football tactical-board (ADR-160 Scene 5). SVG per ADR-177.
 */
@Component({
  selector: 'db-pitch',
  standalone: true,
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <svg [attr.viewBox]="viewBox()" preserveAspectRatio="xMidYMid meet" [style.width.px]="width()" [style.height]="'auto'">
      <g data-chrome aria-hidden="true">
        <rect x="0" y="0" [attr.width]="dims().w" [attr.height]="dims().h" fill="var(--bg-sunken)" stroke="var(--rule)" stroke-width="0.4" />
        @if (frame() !== 'mini') {
          <!-- halfway line + centre circle -->
          <line x1="0" [attr.y1]="dims().h / 2" [attr.x2]="dims().w" [attr.y2]="dims().h / 2" stroke="var(--rule)" stroke-width="0.4" />
          <circle [attr.cx]="dims().w / 2" [attr.cy]="dims().h / 2" r="9.15" fill="none" stroke="var(--rule)" stroke-width="0.4" />
          <circle [attr.cx]="dims().w / 2" [attr.cy]="dims().h / 2" r="0.5" fill="var(--rule)" />
          <!-- top + bottom penalty boxes (full) / single box (half) -->
          @for (box of penaltyBoxes(); track $index) {
            <rect [attr.x]="box.x" [attr.y]="box.y" [attr.width]="box.w" [attr.height]="box.h" fill="none" stroke="var(--rule)" stroke-width="0.4" />
          }
        }
      </g>
      <ng-content />
    </svg>
  `,
})
export class DbPitch {
  readonly frame = input<PitchFrame>('full');
  readonly width = input<number>(320);

  protected readonly dims = computed(() => {
    switch (this.frame()) {
      case 'mini': return { w: 100, h: 60 };
      case 'half': return { w: 100, h: 64 };
      case 'full': return { w: 100, h: 120 };
    }
  });
  protected readonly viewBox = computed(() => `0 0 ${this.dims().w} ${this.dims().h}`);

  protected readonly penaltyBoxes = computed(() => {
    const { w, h } = this.dims();
    const boxW = 40; const boxH = 16; const x = (w - boxW) / 2;
    // full: a box at each end; half: one box at the bottom (own goal).
    return this.frame() === 'full'
      ? [{ x, y: 0, w: boxW, h: boxH }, { x, y: h - boxH, w: boxW, h: boxH }]
      : [{ x, y: h - boxH, w: boxW, h: boxH }];
  });
}
```

- [ ] **Step 4: Run — expect PASS.** `npx nx test design-system-angular -t "DbPitch"`

- [ ] **Step 5: Barrel exports.** `pitch/index.ts`:
```ts
export * from './pitch.component.js';
```
Add to `libs/design-system-angular/src/index.ts` (next to the other brick exports):
```ts
export * from './lib/pitch/index.js';
```

- [ ] **Step 6: Build the lib — expect clean.** `npx nx build design-system-angular`

- [ ] **Step 7: Lint.** `npx nx lint design-system-angular`

- [ ] **Step 8: Commit.** (temp-file message)
```
git add libs/design-system-angular/src/lib/pitch/ libs/design-system-angular/src/index.ts
git commit -m "feat(design-system-angular): add <db-pitch> SVG pitch brick (mini|half|full)"
```

---

## Task A2: Version bump (design-system)

**Files:** Modify `libs/design-system-angular/package.json`.

- [ ] **Step 1: Bump the version** `1.4.0 → 1.5.0` (new minor — additive export) in `libs/design-system-angular/package.json` (`"version": "1.5.0"`).

- [ ] **Step 2: Build verifies the dist carries the new version + brick.** `npx nx build design-system-angular` then confirm `dist/libs/design-system-angular/package.json` shows `"version": "1.5.0"` and the built `index` re-exports `DbPitch`.

- [ ] **Step 3: Commit.**
```
git add libs/design-system-angular/package.json
git commit -m "chore(design-system-angular): release 1.5.0 (<db-pitch>)"
```

- [ ] **Step 4: Push + PR (design-system repo).**
```
git push -u origin feat/db-pitch-brick
gh pr create --base main --title "feat(design-system-angular): <db-pitch> SVG brick + 1.5.0" --body "Adds the frame-parameterized <db-pitch> chrome primitive (ADR-160 Scene 5 consumer; SVG per ADR-177). Bumps to 1.5.0 for publish."
```

---

## Task A3: Publish `@de-braighter/design-system-angular@1.5.0`

> **Credentialed step.** Requires `$GITHUB_TOKEN` with `write:packages`. If the executing agent lacks it, STOP and ask the user to run it (see Task A4 for a no-publish local fallback to keep S5-B moving).

- [ ] **Step 1: Publish** from the design-system repo root (after the A2 PR merges, or from the branch for a prerelease — prefer publishing the merged `main`):
```
# $env:GITHUB_TOKEN must be set (write:packages)
npm run publish:libs
```
This runs `build:libs` then `npm publish` for core/css/angular/react from `dist/libs/*`. Expected: `@de-braighter/design-system-angular@1.5.0` (+ `design-system-core@1.1.3`) published to GitHub Packages.

- [ ] **Step 2: Verify the published version is resolvable.**
```
npm view @de-braighter/design-system-angular@1.5.0 version
```
Expected output: `1.5.0`.

---

## Task A4: Wire the dependency into exercir (+ smoke import)

**Repo:** `domains/exercir`. Branch: `feat/tactical-board-scene5` (S5-B shares this branch).
**Files:** Modify `package.json`; create a temporary smoke test.

- [ ] **Step 1: Add the deps** to `domains/exercir/package.json` `dependencies` (mirror the substrate caret-range pattern):
```json
"@de-braighter/design-system-angular": "^1.5.0",
"@de-braighter/design-system-core": "^1.1.2",
```

- [ ] **Step 2: Install.**
```
# $env:GITHUB_TOKEN must be set (read:packages)
cd /d/development/projects/de-braighter/domains/exercir && npm install
```
**Fallback (no token / not yet published):** in the design-system repo run `cd dist/libs/design-system-angular && npm pack` (produces a `.tgz`) and `cd dist/libs/design-system-core && npm pack`, then in exercir `npm install <abs-path-to-design-system-core.tgz> <abs-path-to-design-system-angular.tgz>`. Replace with the caret-range published deps before the S5-B PR merges.

- [ ] **Step 3: Verify install (real dirs).** `ls node_modules/@de-braighter/` shows `design-system-angular` + `design-system-core`.

- [ ] **Step 4: Smoke-import test** — confirm the brick resolves + Angular peer compatibility. Create `libs/pack-football-ui/src/lib/generation/db-pitch-smoke.spec.ts`:
```ts
import { describe, it, expect, beforeEach } from 'vitest';
import { TestBed } from '@angular/core/testing';
import { Component } from '@angular/core';
import { DbPitch } from '@de-braighter/design-system-angular';

@Component({ standalone: true, imports: [DbPitch], template: `<db-pitch frame="full" />` })
class Host {}

describe('db-pitch smoke import', () => {
  beforeEach(() => TestBed.configureTestingModule({ imports: [Host] }));
  it('resolves and renders the brick from the published package', () => {
    const f = TestBed.createComponent(Host);
    f.detectChanges();
    expect((f.nativeElement as HTMLElement).querySelector('svg')).toBeTruthy();
  });
});
```

- [ ] **Step 5: Run — expect PASS.** `npx nx test pack-football-ui -t "db-pitch smoke"`. If the import fails to resolve, the dependency/registry wiring is wrong — fix before proceeding. (Keep this smoke spec; it doubles as a dependency-presence guard.)

- [ ] **Step 6: Commit.**
```
git add package.json package-lock.json libs/pack-football-ui/src/lib/generation/db-pitch-smoke.spec.ts
git commit -m "build(exercir): consume @de-braighter/design-system-angular ^1.5.0 (<db-pitch>)"
```

---

## Task B1: Full-pitch formation template + view types

**Repo:** `domains/exercir` (branch `feat/tactical-board-scene5`).
**Files:** Create `libs/pack-football-ui/src/lib/generation/tactical-board-template.ts` (+ `.spec.ts`), `tactical-board.types.ts`.

- [ ] **Step 1: Write the failing template spec** `tactical-board-template.ts` test:
```ts
import { describe, it, expect } from 'vitest';
import { FULL_PITCH_4_3_3, FULL_PITCH } from './tactical-board-template.js';

describe('FULL_PITCH_4_3_3', () => {
  it('has 11 slots within the 100x120 vertical pitch bounds', () => {
    expect(FULL_PITCH_4_3_3).toHaveLength(11);
    for (const s of FULL_PITCH_4_3_3) {
      expect(s.x).toBeGreaterThanOrEqual(0); expect(s.x).toBeLessThanOrEqual(100);
      expect(s.y).toBeGreaterThanOrEqual(0); expect(s.y).toBeLessThanOrEqual(120);
    }
  });
  it('places the keeper near the own goal (high y) and forwards near the opponent goal (low y)', () => {
    expect(FULL_PITCH_4_3_3[0].position).toBe('GK');
    expect(FULL_PITCH_4_3_3[0].y).toBeGreaterThan(100);          // own goal end
    expect(FULL_PITCH_4_3_3[9].y).toBeLessThan(FULL_PITCH_4_3_3[0].y); // striker further up
  });
  it('exposes the full-pitch dimensions', () => {
    expect(FULL_PITCH).toEqual({ length: 100, width: 120 });
  });
});
```

- [ ] **Step 2: Run — expect FAIL.** `npx nx test pack-football-ui -t "FULL_PITCH_4_3_3"`

- [ ] **Step 3: Implement** `tactical-board-template.ts` (vertical 100×120, attacking toward y=0; GK at the y≈113 own-goal end). Reuse the `Position`/`FormationSlot` types from `formation-template.ts` if exported, else define a local `FullPitchSlot`:
```ts
import type { Position } from './formation-template.js';

export interface FullPitchSlot { x: number; y: number; position: Position; }

export const FULL_PITCH = { length: 100, width: 120 } as const;

/** 4-3-3 on the full vertical pitch (attacking up; own goal at y≈120). Reading order GK→def→mid→fwd. */
export const FULL_PITCH_4_3_3: readonly FullPitchSlot[] = [
  { x: 50, y: 113, position: 'GK' },
  { x: 82, y: 92, position: 'RB' },
  { x: 62, y: 96, position: 'CB' },
  { x: 38, y: 96, position: 'CB' },
  { x: 18, y: 92, position: 'LB' },
  { x: 30, y: 66, position: 'CM' },
  { x: 50, y: 70, position: 'CM' },
  { x: 70, y: 66, position: 'CM' },
  { x: 80, y: 38, position: 'RW' },
  { x: 50, y: 30, position: 'ST' },
  { x: 20, y: 38, position: 'LW' },
];
```
> If `Position` is not exported from `formation-template.ts`, import it from the lib barrel or define the union locally (`'GK'|'RB'|'CB'|'LB'|'CM'|'RW'|'ST'|'LW'`). Confirm against `formation-template.ts`.

- [ ] **Step 4: Define view types** `tactical-board.types.ts`:
```ts
export interface BoardViewport { width: number; height: number; }
export interface BoardSlot { slotIndex: number; position: string; cx: number; cy: number; playerId: string | null; ariaLabel: string; }
export interface TacticalBoardView {
  viewport: BoardViewport;
  slots: readonly BoardSlot[];
  formationKey: string;
  summaryAriaLabel: string;
}
```

- [ ] **Step 5: Run — expect PASS.** `npx nx test pack-football-ui -t "FULL_PITCH_4_3_3"`

- [ ] **Step 6: Commit.**
```
git add libs/pack-football-ui/src/lib/generation/tactical-board-template.ts libs/pack-football-ui/src/lib/generation/tactical-board-template.spec.ts libs/pack-football-ui/src/lib/generation/tactical-board.types.ts
git commit -m "feat(pack-football-ui): full-pitch 4-3-3 template + tactical-board view types"
```

---

## Task B2: Pure layout fn

**Files:** Create `libs/pack-football-ui/src/lib/generation/tactical-board-layout.ts` (+ `.spec.ts`).

- [ ] **Step 1: Write the failing layout spec:**
```ts
import { describe, it, expect } from 'vitest';
import { layoutTacticalBoard } from './tactical-board-layout.js';

const lineup = ['gk-id', null, null, null, null, null, null, null, null, 'st-id', null];

describe('layoutTacticalBoard', () => {
  it('maps the 11 slots onto the full pitch and carries playerIds', () => {
    const v = layoutTacticalBoard({ formationKey: '4-3-3', lineup }, { width: 300, height: 360 });
    expect(v.slots).toHaveLength(11);
    expect(v.slots[0]).toMatchObject({ slotIndex: 0, position: 'GK', playerId: 'gk-id' });
    expect(v.slots[9].playerId).toBe('st-id');
  });
  it('scales normalized slot coords into the viewport', () => {
    const v = layoutTacticalBoard({ formationKey: '4-3-3', lineup }, { width: 100, height: 120 });
    // GK at normalized (50,113) → same in a 100x120 viewport
    expect(Math.round(v.slots[0].cx)).toBe(50);
    expect(Math.round(v.slots[0].cy)).toBe(113);
  });
  it('builds per-slot + summary aria labels', () => {
    const v = layoutTacticalBoard({ formationKey: '4-3-3', lineup }, { width: 100, height: 120 });
    expect(v.slots[0].ariaLabel.toLowerCase()).toContain('gk');
    expect(v.summaryAriaLabel).toContain('4-3-3');
  });
});
```

- [ ] **Step 2: Run — expect FAIL.** `npx nx test pack-football-ui -t "layoutTacticalBoard"`

- [ ] **Step 3: Implement** `tactical-board-layout.ts`:
```ts
import { FULL_PITCH, FULL_PITCH_4_3_3 } from './tactical-board-template.js';
import type { BoardViewport, TacticalBoardView } from './tactical-board.types.js';

export interface TacticalBoardInput { formationKey: string; lineup: readonly (string | null)[]; }

export function layoutTacticalBoard(input: TacticalBoardInput, viewport: BoardViewport): TacticalBoardView {
  const sx = viewport.width / FULL_PITCH.length;
  const sy = viewport.height / FULL_PITCH.width;
  const slots = FULL_PITCH_4_3_3.map((slot, i) => ({
    slotIndex: i,
    position: slot.position,
    cx: slot.x * sx,
    cy: slot.y * sy,
    playerId: input.lineup[i] ?? null,
    ariaLabel: `${slot.position} slot ${i + 1}${input.lineup[i] ? ', assigned' : ', empty'}`,
  }));
  const filled = slots.filter((s) => s.playerId !== null).length;
  return {
    viewport,
    slots,
    formationKey: input.formationKey,
    summaryAriaLabel: `Formation ${input.formationKey}; ${filled} of 11 positions filled`,
  };
}
```
> v1 supports the 4-3-3 template only (matching the desk-side scene's v1). Other formations are a follow-on; `formationKey` is carried through and surfaced but the slot geometry is 4-3-3 until more templates land.

- [ ] **Step 4: Run — expect PASS.** `npx nx test pack-football-ui -t "layoutTacticalBoard"`

- [ ] **Step 5: Commit.**
```
git add libs/pack-football-ui/src/lib/generation/tactical-board-layout.ts libs/pack-football-ui/src/lib/generation/tactical-board-layout.spec.ts
git commit -m "feat(pack-football-ui): pure tactical-board layout fn"
```

---

## Task B3: `MatchEventClient` (SSE consumer)

**Files:** Create `libs/pack-football-ui/src/lib/data/match-event.client.ts` (+ `.spec.ts`).

**Mirror:** `data/drill-catalog.client.ts` / `substrate-client.ts` for config (`{ baseUrl }`), the `FC_LANGGASSE_*` header context, and the typed-error style. The SSE endpoint emits frames `id: <seq>\n event: football-<kebab-event>\n data: <{envelope,payload} JSON>\n\n` (e.g. `football:SubstitutionMade.v1` → event name `football-substitution-made`).

- [ ] **Step 1: Write the failing client spec** (inject a fake EventSource so the test is deterministic):
```ts
import { describe, it, expect, vi } from 'vitest';
import { MatchEventClient, type MatchEvent } from './match-event.client.js';

class FakeEventSource {
  static instances: FakeEventSource[] = [];
  url: string; listeners = new Map<string, (e: MessageEvent) => void>();
  closed = false;
  constructor(url: string) { this.url = url; FakeEventSource.instances.push(this); }
  addEventListener(type: string, cb: (e: MessageEvent) => void) { this.listeners.set(type, cb); }
  close() { this.closed = true; }
  emit(type: string, data: unknown) { this.listeners.get(type)?.({ data: JSON.stringify(data) } as MessageEvent); }
}

describe('MatchEventClient', () => {
  it('subscribes to the match events URL and parses substitution events', () => {
    const events: MatchEvent[] = [];
    const client = new MatchEventClient({ baseUrl: 'http://x', eventSourceImpl: FakeEventSource as unknown as typeof EventSource });
    client.subscribe('match-1', (e) => events.push(e));
    const es = FakeEventSource.instances.at(-1)!;
    expect(es.url).toContain('/pack-football/live/match/match-1/events');
    es.emit('football-substitution-made', { envelope: { aggregateId: 'match-1', sequence: '7' }, payload: { matchId: 'match-1', playerOutRef: { id: 'a' }, playerInRef: { id: 'b' }, minute: 63 } });
    expect(events).toHaveLength(1);
    expect(events[0]).toMatchObject({ kind: 'substitution-made', minute: 63 });
  });
  it('parses formation-changed events', () => {
    const events: MatchEvent[] = [];
    const client = new MatchEventClient({ baseUrl: 'http://x', eventSourceImpl: FakeEventSource as unknown as typeof EventSource });
    client.subscribe('match-1', (e) => events.push(e));
    FakeEventSource.instances.at(-1)!.emit('football-formation-changed', { envelope: { sequence: '8' }, payload: { matchId: 'match-1', fromFormation: '4-3-3', toFormation: '4-4-2', minute: 70 } });
    expect(events[0]).toMatchObject({ kind: 'formation-changed', toFormation: '4-4-2' });
  });
  it('close() tears down the EventSource', () => {
    const client = new MatchEventClient({ baseUrl: 'http://x', eventSourceImpl: FakeEventSource as unknown as typeof EventSource });
    const sub = client.subscribe('match-1', () => {});
    sub.close();
    expect(FakeEventSource.instances.at(-1)!.closed).toBe(true);
  });
});
```

- [ ] **Step 2: Run — expect FAIL.** `npx nx test pack-football-ui -t "MatchEventClient"`

- [ ] **Step 3: Implement** `match-event.client.ts`:
```ts
export type MatchEvent =
  | { kind: 'substitution-made'; matchId: string; playerOutId: string; playerInId: string; minute: number; sequence: string }
  | { kind: 'formation-changed'; matchId: string; fromFormation: string; toFormation: string; minute: number; sequence: string };

export interface MatchEventClientConfig {
  baseUrl: string;
  /** Injectable for tests; defaults to the global EventSource. */
  eventSourceImpl?: typeof EventSource;
}

export interface MatchEventSubscription { close(): void; }

const SUB_EVENT = 'football-substitution-made';
const FORM_EVENT = 'football-formation-changed';

export class MatchEventClient {
  constructor(private readonly config: MatchEventClientConfig) {}

  subscribe(matchId: string, onEvent: (e: MatchEvent) => void): MatchEventSubscription {
    const Impl = this.config.eventSourceImpl ?? EventSource;
    const url = `${this.config.baseUrl}/pack-football/live/match/${encodeURIComponent(matchId)}/events`;
    const es = new Impl(url, { withCredentials: false });

    es.addEventListener(SUB_EVENT, (ev) => {
      const { envelope, payload } = JSON.parse((ev as MessageEvent).data) as { envelope: { sequence?: string }; payload: { matchId: string; playerOutRef: { id: string }; playerInRef: { id: string }; minute: number } };
      onEvent({ kind: 'substitution-made', matchId: payload.matchId, playerOutId: payload.playerOutRef.id, playerInId: payload.playerInRef.id, minute: payload.minute, sequence: envelope.sequence ?? '' });
    });
    es.addEventListener(FORM_EVENT, (ev) => {
      const { envelope, payload } = JSON.parse((ev as MessageEvent).data) as { envelope: { sequence?: string }; payload: { matchId: string; fromFormation: string; toFormation: string; minute: number } };
      onEvent({ kind: 'formation-changed', matchId: payload.matchId, fromFormation: payload.fromFormation, toFormation: payload.toFormation, minute: payload.minute, sequence: envelope.sequence ?? '' });
    });

    return { close: () => es.close() };
  }
}
```
> The native `EventSource` reconnects automatically and replays via `Last-Event-ID`; no manual reconnect loop needed for v1. (The header trio the API guard needs is set by the dev proxy / same-origin context; if cross-origin auth headers on the SSE are required later, that's a follow-on — the read-only board degrades gracefully if the stream is quiet.)

- [ ] **Step 4: Run — expect PASS.** `npx nx test pack-football-ui -t "MatchEventClient"`

- [ ] **Step 5: Commit.**
```
git add libs/pack-football-ui/src/lib/data/match-event.client.ts libs/pack-football-ui/src/lib/data/match-event.client.spec.ts
git commit -m "feat(pack-football-ui): MatchEventClient SSE consumer (substitution + formation events)"
```

---

## Task B4: `CoachTacticalBoardComponent` (read-only live scene)

**Files:** Create `libs/pack-football-ui/src/lib/coach/ui/coach-tactical-board.component.ts` (+ `.spec.ts`).

**Mirror:** the read-only drill-board scene for the SVG-overlay + a11y (`role="img"` summary + visually-hidden list) structure, and `coach-lineup-page.component.ts` for how it reads the match lineup from the plan-tree (`CoachStore`/derive fns). Use `<db-pitch frame="full">` for chrome; project slot circles as overlay children.

- [ ] **Step 1: Write the failing component spec:**
```ts
import { describe, it, expect, beforeEach } from 'vitest';
import { TestBed } from '@angular/core/testing';
import { CoachTacticalBoardComponent } from './coach-tactical-board.component.js';

describe('CoachTacticalBoardComponent', () => {
  beforeEach(() => TestBed.configureTestingModule({ imports: [CoachTacticalBoardComponent] }));

  it('renders 11 slot markers over a full-frame db-pitch', () => {
    const f = TestBed.createComponent(CoachTacticalBoardComponent);
    f.componentRef.setInput('match', { formationKey: '4-3-3', lineup: Array(11).fill(null) });
    f.detectChanges();
    const el = f.nativeElement as HTMLElement;
    expect(el.querySelector('db-pitch')).toBeTruthy();
    expect(el.querySelectorAll('[data-slot]').length).toBe(11);
  });

  it('exposes a role=img summary + visually-hidden slot list', () => {
    const f = TestBed.createComponent(CoachTacticalBoardComponent);
    f.componentRef.setInput('match', { formationKey: '4-3-3', lineup: Array(11).fill(null) });
    f.detectChanges();
    const el = f.nativeElement as HTMLElement;
    expect(el.querySelector('[role="img"]')?.getAttribute('aria-label')).toContain('4-3-3');
    expect(el.querySelectorAll('.sr-only li').length).toBe(11);
  });

  it('applies a substitution event to the committed lineup and re-renders', () => {
    const f = TestBed.createComponent(CoachTacticalBoardComponent);
    const lineup = ['out-id', ...Array(10).fill(null)];
    f.componentRef.setInput('match', { formationKey: '4-3-3', lineup });
    f.detectChanges();
    const cmp = f.componentInstance;
    cmp.onMatchEvent({ kind: 'substitution-made', matchId: 'm', playerOutId: 'out-id', playerInId: 'in-id', minute: 63, sequence: '7' });
    f.detectChanges();
    expect(cmp['liveLineup']()).toContain('in-id');
    expect(cmp['liveLineup']()).not.toContain('out-id');
  });
});
```

- [ ] **Step 2: Run — expect FAIL.** `npx nx test pack-football-ui -t "CoachTacticalBoard"`

- [ ] **Step 3: Implement** the component. Standalone, OnPush, selector `lib-coach-tactical-board`, imports `DbPitch` (from `@de-braighter/design-system-angular`). Input `match = input.required<{ formationKey: string; lineup: (string|null)[] }>()`; `liveLineup = signal(...)` seeded from the input (effect+untracked, as in the drill editor); `view = computed(() => layoutTacticalBoard({ formationKey: match().formationKey, lineup: liveLineup() }, viewport()))`. `onMatchEvent(e)` updates `liveLineup` (substitution: replace playerOutId→playerInId by value; formation-changed: update formationKey signal). Template: a `<div role="img" [attr.aria-label]="view().summaryAriaLabel">` containing `<db-pitch frame="full">` with projected `@for (s of view().slots)` `<circle data-slot [attr.cx]="s.cx" [attr.cy]="s.cy" r="3.4" [attr.aria-label]="s.ariaLabel">`, plus a visually-hidden `<ul class="sr-only">` of slot ariaLabels, plus an `aria-live="polite"` region announcing applied events. Wire `MatchEventClient.subscribe` in `ngOnInit` using the configured baseUrl, calling `onMatchEvent`; `close()` in `ngOnDestroy`/`DestroyRef`. (For the spec, `onMatchEvent` is called directly; the live subscription is exercised in the browser smoke-test.)

- [ ] **Step 4: Run — expect PASS.** `npx nx test pack-football-ui -t "CoachTacticalBoard"`

- [ ] **Step 5: Lint.** `npx nx lint pack-football-ui`

- [ ] **Step 6: Commit.**
```
git add libs/pack-football-ui/src/lib/coach/ui/coach-tactical-board.component.ts libs/pack-football-ui/src/lib/coach/ui/coach-tactical-board.component.spec.ts
git commit -m "feat(pack-football-ui): read-only live tactical-board scene (db-pitch + SSE re-render)"
```

---

## Task B5: Route + nav + exports

**Files:** Modify `libs/pack-football-ui/src/lib/shell/fc-workspace.routes.ts`, `fc-workspace.types.ts`, `src/index.ts`.

- [ ] **Step 1: Add the coach route** in `fc-workspace.routes.ts` `COACH_ROUTES` (mirror the existing `lineup`/`drills` lazy-load entries):
```ts
{
  path: 'board',
  loadComponent: () =>
    import('../coach/ui/coach-tactical-board.component.js').then((m) => m.CoachTacticalBoardComponent),
  data: { packKey: 'football', persona: 'coach', surface: 'board' },
},
```
> The board component needs the selected match. Mirror how `lineup`/`match` pages obtain the active match (`CoachStore.activePlanTree` + `extractMatches`/derive). If the board needs a wrapper page (to source the match + provide `MatchEventClient` baseUrl), create `coach-tactical-board-page.component.ts` mirroring `coach-lineup-page.component.ts` and lazy-load that instead. Keep the scene component input-driven (testable) and the page component the data-sourcing shell.

- [ ] **Step 2: Add the nav entry + theme** in `fc-workspace.types.ts`: add `{ id: 'board', label: 'Taktiktafel' }` (or the established nav-entry shape) to the coach `FC_ROUTES.coach` list, and `board: 'stadion'` to `FC_THEME_FOR_ROUTE.coach`.

- [ ] **Step 3: Export** from `libs/pack-football-ui/src/index.ts`:
```ts
export { CoachTacticalBoardComponent } from './lib/coach/ui/coach-tactical-board.component.js';
export { layoutTacticalBoard } from './lib/generation/tactical-board-layout.js';
export { MatchEventClient, type MatchEvent } from './lib/data/match-event.client.js';
export { FULL_PITCH_4_3_3, FULL_PITCH } from './lib/generation/tactical-board-template.js';
export type { TacticalBoardView, BoardSlot } from './lib/generation/tactical-board.types.js';
```

- [ ] **Step 4: Run project tests + lint.** `npx nx test pack-football-ui && npx nx lint pack-football-ui` — expect PASS.

- [ ] **Step 5: Commit.**
```
git add libs/pack-football-ui/src/lib/shell/fc-workspace.routes.ts libs/pack-football-ui/src/lib/shell/fc-workspace.types.ts libs/pack-football-ui/src/index.ts libs/pack-football-ui/src/lib/coach/ui/
git commit -m "feat(pack-football-ui): /coach/board route + nav + tactical-board exports"
```

---

## Task B6: Gate + browser smoke-test

- [ ] **Step 1: Local gate.** From `domains/exercir`: `npm run ci:local` — build + lint + test green across all projects. Fix any boundary/public-API breakage.

- [ ] **Step 2: Sonar.** `npm run sonar:coverage && npm run sonar:scan` — Quality Gate **OK** (new code covered; 0 new violations).

- [ ] **Step 3: Browser smoke-test.** `nx serve pack-football-api` (:3100) + `nx serve pack-football-visual-editor` (:4200); navigate to `/t/b6c5d8e2-1234-4abc-9def-fc1a55e1a55e/p/football/coach/board`. Confirm: the full pitch renders with 11 slot markers in 4-3-3; the accessibility tree shows the `role="img"` summary + hidden slot list. If a live match event is emitted on the SSE stream (or can be triggered), confirm the board re-renders. **Check the browser console for CORS/SSE errors** (the SSE GET should be allowed; the drill-board PUT-CORS bug is the precedent for verifying transport in a real browser).

- [ ] **Step 4: Commit any fixes + open the exercir PR.**
```
git add -A
git commit -m "chore(pack-football-ui): tactical-board S5-A/B gate fixes"
git push -u origin feat/tactical-board-scene5
gh pr create --base main --title "feat: tactical-board Scene 5 (S5-A brick wiring + S5-B read-only live board)" --body "Adds <db-pitch> consumption + the read-only live tactical board at /coach/board (full-pitch render + SSE re-render). Depends on @de-braighter/design-system-angular@1.5.0 (design-system PR). Spec: docs/superpowers/specs/2026-05-26-tactical-board-scene5-design.md"
```

---

## Self-review notes (addressed)

- **Spec coverage (S5-A + S5-B):** `<db-pitch>` brick + publish + wire (A1–A4); full-pitch template + view types (B1); pure layout (B2); SSE consumer re-rendering on FormationChanged/SubstitutionMade (B3, B4); read-only full-pitch render at `/coach/board` (B4, B5); a11y `role="img"` + hidden list + aria-live (B4); gate + browser smoke incl. transport check (B6). Stages S5-C (writes), S5-D (annotations), S5-E (snapshots) are **out of this plan** (follow-on plans per the spec's build order).
- **Cross-repo ordering:** A1–A2 (design-system brick + version) → A3 (publish, credentialed) → A4 (exercir consumes) → B1–B6. A4's tarball fallback unblocks S5-B dev if publishing is gated on a token.
- **Type consistency:** `TacticalBoardView`/`BoardSlot` (B1) consumed by `layoutTacticalBoard` (B2) and the scene (B4); `MatchEvent` (B3) consumed by `onMatchEvent` (B4); `FULL_PITCH_4_3_3`/`FULL_PITCH` (B1) consumed by the layout (B2); `DbPitch` (`db-pitch`, A1) consumed by the scene (B4) + smoke test (A4).
- **Mirror-pointers** (live-convention-dependent): the design-system component-test harness (A1), the page-vs-scene data-sourcing split + how the active match is read from `CoachStore` (B4/B5), and the exact nav-entry shape in `fc-workspace.types.ts` (B5) — each names the file to mirror.
- **Deferred/flagged:** SSE auth headers cross-origin (native EventSource can't set headers — relies on same-origin/proxy in dev; a token/cookie scheme is a follow-on if the live stream needs auth beyond the dev proxy); multi-formation slot templates (4-3-3 only in v1).

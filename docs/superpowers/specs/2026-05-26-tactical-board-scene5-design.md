# Tactical-board (ADR-160 Scene 5) — live match-day board (design)

- **Date:** 2026-05-26
- **Status:** design (brainstormed; pending implementation plan)
- **Domain:** `de-braighter/domains/exercir` (`pack-football` + `pack-football-ui` + `pack-football-api`) **and** `de-braighter/layers/design-system` (`@de-braighter/design-system-angular`)
- **Tracker:** [de-braighter/exercir#93](https://github.com/de-braighter/exercir/issues/93)
- **Builds on:** the shipped drill-board editable canvas (Scene 4) + the desk-side coach lineup/match-prep surfaces.
- **Governing ADRs:** [ADR-160](https://github.com/de-braighter/specs) §"Scene 5 — Tactical board" (the surface), [ADR-138](https://github.com/de-braighter/specs) (four-layer visual-editor + GestureInterpreter), [ADR-162](https://github.com/de-braighter/specs) (live-telemetry SSE — Scene 5 is a consumer), [ADR-177](https://github.com/de-braighter/specs) (**SVG canonical**, Konva retired), [ADR-168](https://github.com/de-braighter/specs) (design-system bricks-everywhere, `db-*`), [ADR-176](https://github.com/de-braighter/specs) (promote-to-shared-layer on 2nd consumer), [ADR-033 §4](https://github.com/de-braighter/specs) (tier immutability).

## 1. Summary

Build the **live match-day tactical board** (ADR-160 Scene 5) at a new `/coach/board` route: render the current lineup/formation on a **full pitch** (100×120), **consume the live SSE match event stream** (ADR-162) to re-render on `FormationChanged.v1` / `SubstitutionMade.v1`, and let the coach, during the match window, **substitute, change formation, draw play-pattern annotations, and capture per-minute formation snapshots**.

This is a **two-repo feature**: it introduces a shared **`<db-pitch>` SVG brick** in `@de-braighter/design-system-angular` (the documented `<fc-pitch>` extraction point, ADR-160 + ADR-172, realised as SVG per ADR-177), and builds Scene 5 in `pack-football-ui` consuming it. Multi-editor collaboration (Yjs/Hocuspocus) is **deferred to v2** per ADR-160.

The match-day **write use-cases already exist** (`apply-substitution`, `apply-formation-change`, `apply-lineup`, `apply-captain`, `get-match-prep`) and emit events; what is **new** is: the live **SSE UI consumer**, dedicated **HTTP endpoints** for the match-day use-cases, **play-pattern annotation** persistence, and **per-minute snapshot** persistence.

## 2. Decisions locked (from the 2026-05-26 brainstorm)

| Decision | Choice |
|---|---|
| v1 scope | **Full authoring board** — live render + live SSE + substitution/formation writes + play annotations + per-minute snapshots. Multi-editor collab → v2. |
| Renderer | **SVG** — ADR-177 is authoritative (confirmed in design-system code: the Konva `db-visual-editor` brick is retired). |
| Pitch primitive | **`<db-pitch>` SVG brick in `@de-braighter/design-system-angular`** — cross-repo, published, added as a **new exercir dependency**. (Founder chose the design-system path over a pack-local helper, accepting the publish + new-cross-repo-dependency cost; exercir consumes zero design-system bricks today.) |
| Substitution/formation writes | **New dedicated HTTP endpoints** wrapping the existing use-cases (no HTTP exists yet). |
| Annotation/snapshot writes | The **existing generic `POST /pack-football/edits`** (metadata-patch + subtree-insertion). |
| Mount | New **`/coach/board`** route, sibling to `/coach/lineup` (desk-side XI) + `/coach/match` (read-only prep). |
| Conflict model | Live SSE owns **lineup/formation** (committed); coach **annotations** are a separate **local overlay** that never conflicts; the coach's own subs echo back via SSE and reconcile by `entryId`. |
| Pitch consolidation | The 3 existing pitches (drill 100×60, formation 100×64, coach-lineup 105×68) are **not** migrated onto `<db-pitch>` in this slice — tracked follow-up. |

## 3. Scope

**In:** full-pitch live render; live SSE consumer; substitution + formation-change from the board; play-pattern annotations (run-arrow / pass-arrow / zone-highlight); per-minute formation snapshots; the `<db-pitch>` brick + cross-repo wiring; full keyboard a11y; the new match-day HTTP endpoints.

**Out (deferred):** multi-editor collaboration (Yjs/Hocuspocus — ADR-160 v2); consolidating the existing 3 pitches onto `<db-pitch>`; the FC-Cockpit player-event telemetry feed (ADR-162's own scope); offline/replay buffering beyond SSE `Last-Event-ID`.

## 4. Build order (decomposition)

One Scene 5 spec, five independently shippable/PR-able stages:

- **S5-A — `<db-pitch>` brick + cross-repo wiring.** New SVG brick in `design-system-angular` (`frame: mini|half|full`, chrome-only, `<ng-content>` overlay projection); publish a new minor; add `@de-braighter/design-system-angular` (+ `design-system-core` peer) to exercir; smoke-import it. *(design-system repo + exercir.)*
- **S5-B — read-only live board.** `layoutTacticalBoard` (pure) + `CoachTacticalBoardComponent` rendering the lineup on `<db-pitch frame="full">`; `MatchEventClient` (EventSource) consuming `/pack-football/live/match/:id/events`, re-rendering on `FormationChanged.v1` / `SubstitutionMade.v1`; new `/coach/board` route.
- **S5-C — in-match writes.** New `pack-football-match-day.controller.ts` (substitution + formation-change endpoints wrapping the existing use-cases); `MatchDayClient`; board interactions (drag bench→pitch = substitute; formation picker = formation-change); optimistic + SSE reconcile.
- **S5-D — play-pattern annotations.** New `PlayAnnotationSchema` (domain + UI mirror + parity guard); annotation drawing (reuse drill-editor arrow/zone authoring + undo/redo); persist at `metadata.visualEditor.plays[]` via `POST /edits` metadata-patch.
- **S5-E — per-minute snapshots.** New `snapshot-formation` use-case + `kind_ref='match_minute_N'` child-node insertion via `POST /edits` subtree-insertion; snapshot timeline UI.

## 5. Architecture

### 5.1 `<db-pitch>` brick (design-system)
Standalone OnPush SVG component in `libs/design-system-angular/src/lib/pitch/pitch.component.ts`, mirroring the existing `db-pitch-diagram` conventions: `db-` selector, signal `input()`, token colours (`var(--bg-sunken)` grass, `var(--rule)` lines), `@maturity experimental`, barrel-exported from `src/index.ts`. Input `frame: 'mini' | 'half' | 'full'` → viewBox `0 0 100 60` / `0 0 100 64` / `0 0 100 120` and the corresponding chrome (mini = outline + halfway; half/full = penalty boxes, 6-yard boxes, centre circle, goals). Renders **field chrome only**; consumers project overlays as child SVG via `<ng-content>` into the same coordinate space. No interaction, no domain types — a pure primitive.

### 5.2 Scene 5 (pack-football-ui) — four layers (ADR-138)
- **Generation (pure):** `generation/tactical-board-layout.ts` → `layoutTacticalBoard(match, liveState, viewport): TacticalBoardView` — 11 lineup slots positioned per a **full-pitch (100×120, vertical, attacking-up) formation template**, bench rail, projected annotations + zones, derived `summaryAriaLabel`. No Angular. *(NEW template: the existing `FORMATIONS_4_3_3` is the 100×64 horizontal desk frame and is not reused as-is; the full-pitch slot coordinates are a new constant in this layer.)*
- **Scene component:** `coach/ui/coach-tactical-board.component.ts` (`CoachTacticalBoardComponent`, standalone OnPush) hosts `<db-pitch frame="full">` and projects slot/arrow/zone `<g>` overlays; interactive — drag bench→pitch (substitute), move slot (positional tweak), draw annotations, snapshot. Reuses formation-scene's pointer+keyboard drag + `aria-grabbed`, and the drill-editor's annotation authoring (arrow/zone draw, undo/redo).
- **State:** `state/tactical-board.store.ts` (`TacticalBoardStore`, plain signal class) — committed lineup/formation (seeded from plan-tree, updated by SSE), local annotation working-state with undo/redo, pending sub/formation writes (optimistic), `saveStatus`. Live lineup and local annotations are separate signals (the conflict model).
- **Transport:** existing `SubstrateClient` (`/edits`) for annotations/snapshots; new `MatchDayClient` (substitution/formation endpoints, mirrors `DrillCatalogClient` conventions — header trio, `x-request-id`, typed failures); new `MatchEventClient` (EventSource wrapper; `Last-Event-ID` reconnect; parses the SSE envelope; emits typed match events).

### 5.3 Boundary
`pack-football-ui` continues to **not** import the NestJS `pack-football` barrel — match-day wire shapes + `PlayAnnotation` live in the `wire-schemas.ts` mirror, drift-guarded by `wire-schemas-parity.spec.ts`. The design-system dependency is the published `@de-braighter/design-system-angular` (peer-dep, like substrate-contracts/runtime). Nx tags: `scope:pack-football` (domain) → `scope:design-system` (lower layer) is an allowed direction.

## 6. Data flow & conflict model

1. Board loads → reads the `match_simulation` plan-node (`metadata.lineup` 11 UUIDs, `captainId`, `opponent`) via the existing `CoachStore`/`SubstrateClient`; renders on the full pitch.
2. **Live:** `MatchEventClient` opens `GET /pack-football/live/match/:id/events` (reconnect via `Last-Event-ID`); on `FormationChanged.v1` / `SubstitutionMade.v1` → update committed lineup/formation → re-render.
3. **Substitution** (drag bench→pitch): `MatchDayClient.substitute(...)` → **new** `POST /pack-football/matches/:id/substitution` → `apply-substitution` use-case (emits `SubstitutionMade.v1`). The coach's own SSE stream echoes the event back; reconcile is **idempotent by `entryId`** (no double-apply).
4. **Formation change** (picker): `MatchDayClient.changeFormation(...)` → **new** `POST /pack-football/matches/:id/formation` → `apply-formation-change` (emits `FormationChanged.v1`).
5. **Annotations:** draw run/pass arrow or zone → local working-state (undo/redo) → explicit save → `SubstrateClient.applyEdit` **metadata-patch** on `metadata.visualEditor.plays` (existing `/edits`).
6. **Snapshots:** `snapshot-formation` → `/edits` subtree-insertion of a `kind_ref='match_minute_N'` child node carrying the frozen lineup (S5-E use-case validates).

**Conflict model:** the SSE stream is the single source of truth for **lineup/formation** (committed plane); **annotations** are a separate local overlay (authoring plane) that never conflicts with lineup events. A sub the coach initiates is reconciled against its echoed `SubstitutionMade.v1` by `entryId`.

## 7. New domain contracts

- **`PlayAnnotationSchema`** (NEW — `pack-football/src/domain/football-event.ts` + UI mirror in `wire-schemas.ts`): `{ id: string; kind: 'run-arrow' | 'pass-arrow' | 'zone-highlight'; minute?: number; … geometry }` where arrows carry `{x1,y1,x2,y2}` and zones `{x,y,w,h}` in **full-pitch normalized coords** (`x∈[0,100]`, `y∈[0,120]`). Persisted at `metadata.visualEditor = { sceneKind:'pack-football.tactical-board.v1', schemaVersion:'pack-football.tactical-board.v1', plays: PlayAnnotation[] }`. Drift-guarded mirror + parity spec, like the drill `DrillDiagram`.
- **Per-minute snapshot node** (NEW): a `phase` plan-node `kind_ref='match_minute_N'` under the `match_simulation` parent, `metadata.lineup` frozen. A small `snapshot-formation` use-case + repo method validates + inserts via **subtree-insertion** (the `/edits` tree-edit path); **no new domain event in v1** — the snapshot is plan-tree state, visible via the tree read. The only stage (S5-E) with no pre-built use-case.

Both follow the established Zod + parity-mirror + boundary discipline.

## 8. Accessibility

Full keyboard parity (WCAG 2.5.7), reusing the drill-editor + formation-scene patterns:
- Slot move / substitute: focusable slots (`role="button"`, `tabindex`, `aria-grabbed`); Space/Enter pick-up-drop; arrow nudge; bench↔pitch substitution via keyboard.
- Annotation draw: arrow two-anchor + zone create/move/resize via keyboard (as in the drill editor).
- Live events announced via `aria-live="polite"` ("Auswechslung Minute 63: … für …", "Formation 4-3-3 → 4-4-2").
- `<db-pitch>` carries the chrome with `aria-hidden`; the board's `role="application"` + `aria-describedby` help text covers shortcuts; the rendered result keeps a `role="img"` summary + visually-hidden ordered description derived from the layout.

## 9. Error handling

- New endpoints map failures→HTTP like the drill controller (forbidden→403, not-found→404, invalid-input→400, exhaustive `never`→500). **CORS:** the new POST endpoints must be in the API CORS `methods` allow-list (the drill-board PUT-CORS bug — caught only in a browser — is the cautionary precedent; the browser smoke-test must exercise every new verb).
- `MatchEventClient` handles SSE disconnect/reconnect (`Last-Event-ID`) and surfaces a "live feed reconnecting" state; the board stays usable (annotations are local) while the feed is down.
- Save failures preserve local annotation working-state (no data loss), with retry — as in the drill editor.

## 10. Testing

- **Pure:** `tactical-board-layout`, annotation ops, store (undo/redo, sub/formation pending + SSE reconcile by entryId).
- **Component:** pointer + **keyboard** interaction (slot move, substitute, annotation draw, snapshot); `aria-grabbed`; `aria-live` announcements.
- **SSE:** `MatchEventClient` with a mock `EventSource` (event parse, reconnect, reconcile).
- **API:** new substitution/formation endpoints (success + each failure→status); CORS methods check.
- **Contracts:** `PlayAnnotation` wire-schema parity + drift guards; snapshot-node use-case tests.
- **Brick:** `<db-pitch>` spec in design-system (frame→viewBox/chrome; overlay projection).
- **Gate:** `npm run ci:local` + Sonar QG OK in **both** repos; **browser smoke-test** the live flow (open the board, drive a substitution, draw an annotation, snapshot; verify SSE re-render + no CORS errors on the new verbs).

## 11. Cross-repo & release notes

S5-A lands first: build/publish `@de-braighter/design-system-angular` (new minor), then wire exercir's dependency. Subsequent stages (S5-B…E) are exercir-only and consume the published brick. The Scene 5 spec/plan live in the workbench repo; the brick PR is in the design-system repo; Scene 5 PRs are in exercir. Each repo is PR-gated; local gate is the bar (remote Actions billing-blocked).

## 12. Future / not-foreclosed

- Multi-editor collaboration (Yjs/Hocuspocus, `kernel.scene_collaboration_state`) — ADR-160 v2 trigger.
- Consolidate drill/formation/coach-lineup pitches onto `<db-pitch>` (issue follow-up).
- Promote additional pitch overlays (player badge eyecatcher) into the design-system once a 2nd pack consumes them.

## 13. References

- ADR-160 §"Scene 5" + "Shared pitch primitive"; ADR-138 §6/§6.4/§7.3; ADR-162 (SSE); ADR-177 (SVG canonical); ADR-168 (`db-*` bricks); ADR-176 (promotion rule).
- Domain (exists): `libs/pack-football/src/in-ports/{apply-substitution,apply-formation-change,apply-lineup,apply-captain,get-match-prep}.use-case.ts` (+ services); events in `domain/football-event.ts` (`SubstitutionMade.v1`, `FormationChanged.v1`, `LineupSubmitted.v1`, `CaptainAssigned.v1`).
- UI prior art: `generation/formation-scene.component.ts` + `formation-template.ts` (100×64, 4-3-3 slots); `coach/ui/coach-lineup-page.component.ts` + `coach-lineup-pitch.component.ts` (desk-side XI); `coach/ui/coach-match-prep-page.component.ts`; `interaction/gesture-interpreter.ts`; `state/editor-store.ts` (`extractMatches`/`Match`, optimistic overlay); `data/substrate-client.ts`; the shipped drill editor (annotation authoring + a11y patterns).
- Live SSE endpoint (exists): `apps/pack-football-api/src/app/pack-football-live-telemetry.controller.ts` (`GET /pack-football/live/match/:id/events`).
- Design-system: `layers/design-system/libs/design-system-angular/` (`db-pitch-diagram` brick to mirror; `src/index.ts` barrel; tokens in `design-system-css`).
- Route wiring: `libs/pack-football-ui/src/lib/shell/fc-workspace.routes.ts` + `fc-workspace.types.ts`.

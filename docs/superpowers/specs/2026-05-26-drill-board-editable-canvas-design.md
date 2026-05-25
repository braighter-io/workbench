# Drill-board editable canvas (Scene 4 authoring) — design

- **Date:** 2026-05-26
- **Status:** design (brainstormed; pending implementation plan)
- **Domain:** `de-braighter/domains/exercir` → `pack-football` + `pack-football-ui` + `pack-football-api`
- **Builds on:** drill-board v1 (read-only Drill-Bibliothek), shipped in PRs #88/#90/#91 (closes exercir#87). Seed/handoff: `docs/superpowers/plans/2026-05-26-drill-board-editable-canvas-handoff.md`.
- **Governing ADRs:** [ADR-160](https://github.com/de-braighter/specs/blob/main/adr/adr-160-pack-football-visual-editor-fourth-fifth-scenes-drill-diagram-tactical-board.md) §"Scene 4" (drill-diagram editor + catalog-mutation contract), [ADR-033 §4](https://github.com/de-braighter/specs) (vendor-tier immutability / coach-tier fork), [ADR-176](https://github.com/de-braighter/specs) (pack vs kernel; promote-on-2nd-consumer), [ADR-177](https://github.com/de-braighter/specs) (SVG renderer canonical, reverse Konva), [ADR-138](https://github.com/de-braighter/specs) (four-layer visual-editor + gesture-interpreter pattern).

## 1. Summary

Build the **Scene 4 drill-diagram editor** — the authoring half of S-21 that v1 deliberately deferred because it is the a11y-heavy, interactive-SVG slice. A coach opens a drill in the Drill-Bibliothek, enters edit mode, and:

- **places / moves / deletes dots** (player, defender, keeper, cone, ball),
- **draws / deletes arrows** (pass, run, dribble, shot),
- **sets / clears the zone**,

with **full keyboard accessibility parity** (WCAG 2.5.7 — every pointer gesture has a keyboard alternative). Edits accumulate in a **local working diagram** with **undo/redo**; an explicit **Speichern** flushes the whole `DrillDiagram` in a single call. Editing a **vendor** drill **lazily forks** to a tenant-tier copy on first save; tenant drills edit in place.

This is a **pure pack feature** (ADR-176): the diagram lives in the Intervention catalog row's `metadata.diagram`; editing is a **catalog mutation**, not a substrate `tree-edit`. Renderer is **SVG** (ADR-177).

The backend write path **already exists and is tested** (`ForkTemplateUseCase`, `UpdateDrillDiagramUseCase`, wired in `pack-football.module`). This slice adds **two HTTP endpoints** plus the **entire UI editing layer**.

## 2. Decisions locked (from the 2026-05-26 brainstorm)

| Decision | Choice |
|---|---|
| Edit scope | **Full Scene 4 in one decomposed slice** — all element ops + fork + save + keyboard parity. No deferred a11y debt. |
| Save model | **Explicit save + local undo/redo** — working-diagram signal with a snapshot stack; dirty indicator; one whole-diagram PUT per save. |
| Fork timing | **Lazy — fork on first save** of a vendor drill. Tenant drills edit in place. No orphan forks on cancel. |
| Editor architecture | **Separate `DrillBoardEditorComponent`**, reusing the pure `layoutDrillBoard` + `pitch.ts` + extracted shared glyph styles. The read-only `DrillBoardSceneComponent` viewer is left untouched. |
| Renderer | **SVG** — ADR-177 supersedes ADR-160's 2026-05-22 Konva amendment; v1 shipped SVG and the editor extends it. |
| `<fc-pitch>` promotion | **Deferred** — keep `pitch.ts` local; extract the design-system brick when the tactical-board (Scene 5, different 100×120 frame) lands. See §8. |

## 3. Scope

**In:**
- Edit-mode toggle from the Bibliothek detail pane; a `DrillBoardEditorComponent` rendering an interactive SVG pitch.
- The eight ADR-160 Scene 4 gestures, each as a pure op over `DrillDiagram`: `add-dot(kind,x,y)`, `move-dot(id,x,y)`, `set-dot-kind(id,kind)`, `remove-dot(id)`, `draw-arrow(kind,x1,y1,x2,y2)`, `remove-arrow(id)`, `set-zone(x,y,w,h)`, `clear-zone()`.
- Pointer interaction (click-to-place / drag-to-move / drag-to-draw / drag-to-zone) **and** full keyboard interaction (toolbar tool selection, Space/Enter pick-up-drop, arrow-key nudge, Delete/Backspace).
- Local undo/redo over the working diagram; dirty indicator; explicit Save / Cancel-with-discard-confirm.
- Vendor→tenant **lazy fork** on first save, then re-select the forked drill.
- Two HTTP endpoints wrapping the existing use-cases; UI client methods; wire-schema mirrors + parity-spec rows for the new write shapes.
- Live-region announcements for editing operations.

**Out (deferred — unchanged from v1):**
- The **tactical-board** (ADR-160 Scene 5 — live match-day formation surface, 100×120 frame).
- A shared **`<fc-pitch>`** design-system/pack primitive (extract when Scene 5 is real — §8).
- Card-gallery thumbnails for the browse view.
- Full i18n/translations table for drill names (S-5).
- Multi-editor collaboration; server-side per-gesture history (persistence stays whole-diagram).

## 4. Architecture

Four layers, mirroring the established `generation/` + interaction pattern (ADR-138 §6), but **simpler** than formation-scene: drill editing is a catalog mutation persisted **whole-diagram on explicit save**, so there is **no per-gesture network dispatch** and no substrate `tree-edit`.

```
DrillBoardEditorComponent  (standalone, OnPush, interactive SVG + toolbar)
  ├─ state        → DrillEditorStore (signals: workingDiagram, selection, tool,
  │                                   undo/redo stacks, saveStatus, sourceKey/forkedKey/tier)
  ├─ pure render  → layoutDrillBoard() + pitch.ts + drill-board-style.ts (shared glyphs)
  ├─ gestures     → pointer + keyboard handlers → drill-diagram-ops (pure) mutate workingDiagram
  └─ persistence  → DrillCatalogClient.forkDrill() / .saveDrillDiagram() → pack-football-api
```

**Key divergence from formation-scene:** formation-scene dispatches each edit through `GestureInterpreter → SubstrateClient.applyEdit → EditorStore` optimistic overlay (substrate `tree-edit`). The drill editor instead keeps a **local working `DrillDiagram`** and calls the network **only on Save** (fork? then update). The ADR-160 gestures are therefore a **local editing vocabulary** realised as pure functions, not substrate verbs.

**Boundary (load-bearing):** `pack-football-ui` must **not** import the `@de-braighter/pack-football` (NestJS) barrel — esbuild rejects the server pull. The UI uses the `DrillDiagram` **mirror** in `pack-football-ui/src/lib/data/wire-schemas.ts`, drift-guarded by `wire-schemas-parity.spec.ts`. The new write request/response shapes follow the same mirror discipline.

All code lives in `pack-football-ui` (editing UI), `pack-football-api` (two endpoints). **No `pack-football` domain change is required** — the use-cases already exist. **No kernel/substrate/design-system change.**

## 5. Components

All paths under `domains/exercir/`.

| Component | Path | Responsibility |
|---|---|---|
| Shared glyph styles | `libs/pack-football-ui/src/lib/generation/drill-board-style.ts` | **Extracted** from `drill-board-scene.component.ts`: `DOT_COLORS`, `DOT_STROKE`, `ARROW_STROKE`, `ARROW_DASH`, `ARROW_WIDTH`, `conePoints()`, `dotKinds`/`arrowKinds`. Imported by viewer **and** editor so glyphs are identical. |
| Editor types | `libs/pack-football-ui/src/lib/generation/drill-editor.types.ts` | `DrillTool` union (`select` \| `add-{player\|defender\|keeper\|cone\|ball}` \| `arrow-{pass\|run\|dribble\|shot}` \| `zone`), `DrillSelection` (`{ kind: 'dot'\|'arrow'\|'zone'; id?: string }`), small helpers. |
| Diagram ops (pure) | `libs/pack-football-ui/src/lib/generation/drill-diagram-ops.ts` | **Pure** functions over `DrillDiagram`: `addDot`, `moveDot`, `setDotKind`, `removeDot`, `addArrow`, `removeArrow`, `setZone`, `clearZone`. Mint ids; **clamp** coords to `[0,100]×[0,60]` (zone `w,h>0`). No Angular. Return new immutable diagrams (snapshot-friendly). |
| Editor store | `libs/pack-football-ui/src/lib/state/drill-editor.store.ts` | Signals: `workingDiagram`, `selection`, `tool`, `undoStack`/`redoStack`, `saveStatus` (`idle\|dirty\|saving\|saved\|error`), `sourceKey`, `forkedKey`, `tier`, `errorReason`. Methods: `begin(entry)`, `apply(op)` (pushes undo snapshot, sets `dirty`), `undo()`, `redo()`, `select()`, `setTool()`, `markSaving/markSaved/markError`, `reset()`. |
| Editor component | `libs/pack-football-ui/src/lib/generation/drill-board-editor.component.ts` | Standalone, OnPush. Interactive SVG (reuses `layoutDrillBoard` + `pitch.ts` + `drill-board-style`), `role="toolbar"` palette, dirty banner, Save/Cancel. Pointer + keyboard handlers translate to store ops. Emits save requests via `@Output`/callback to the panel. |
| Editor panel | `libs/pack-football-ui/src/lib/drills/drill-editor-panel.component.ts` | Hosts the editor inside the Bibliothek detail pane: **Bearbeiten** toggle, vendor fork banner, wires Save→`DrillCatalogClient` (fork? → update), re-selects forked drill, surfaces save errors. |
| Data client (extend) | `libs/pack-football-ui/src/lib/drills/drill-catalog.client.ts` | Add `forkDrill(sourceKey)` (`POST …/fork`) and `saveDrillDiagram(drillKey, diagram)` (`PUT …/diagram`), reusing the existing `dispatch` (already accepts method/body) + header trio + `x-request-id`. |
| Wire schemas (extend) | `libs/pack-football-ui/src/lib/data/wire-schemas.ts` | Mirror `ForkResponse` (`{ templateId, forkedKey, sourceKey }`) and `UpdateDrillDiagramResponse` (`{ drillKey, updatedAt }`); reuse mirrored `DrillDiagramSchema` for the request body. Add parity-spec rows in `wire-schemas-parity.spec.ts`. |
| Public API (extend) | `libs/pack-football-ui/src/index.ts` | Export `DrillBoardEditorComponent`, `DrillEditorStore`, editor types, and the new ops if consumed across the lib boundary. |
| HTTP endpoints (extend) | `apps/pack-football-api/src/app/pack-football-drills.controller.ts` | `POST /pack-football/drills/:drillKey/fork` → `ForkTemplateUseCase`; `PUT /pack-football/drills/:drillKey/diagram` → `UpdateDrillDiagramUseCase`. Derive `CoachPrincipal` from the tenant/pack context the global `TenantPackContextGuard` validates. |

> **Endpoint-path note:** ADR-160 §Scene 4 (written pre-v1) names the write endpoint `POST /pack-football/intervention-catalog/:drillCode/diagram`. We **deliberately** mount the new routes under the **`drills` resource** the v1 controller already owns (`/pack-football/drills/:drillKey/{fork,diagram}`) for consistency with the shipped read endpoint — same use-cases, same gating, one controller. The ADR's contract (catalog mutation, not tree-edit; whole-diagram payload; vendor-immutability) is honoured; only the URL surface is harmonised with v1.

`DrillTool`/`DrillSelection` (editor types) are consumed by the editor component and store; `drill-diagram-ops` are consumed by the store; the mirrored write schemas by the client.

## 6. Data flow — edit & lazy fork

1. Coach selects a drill, clicks **Bearbeiten** → `DrillEditorStore.begin(entry)` seeds `workingDiagram` from `entry.diagram` (or an empty `pack-football.drill-diagram.v1` diagram if none), records `sourceKey`, `tier`, clears undo/redo, `saveStatus=idle`.
2. If `tier === 'vendor'`, the panel shows a banner: *"Beim Speichern wird eine eigene Kopie für dein Team erstellt."*
3. Each gesture (pointer or keyboard) calls `store.apply(op)`: a pure `drill-diagram-ops` fn produces the next diagram; the previous is pushed to the undo stack; redo stack cleared; `saveStatus=dirty`. A live region announces the op.
4. **Speichern** (`saveStatus=saving`):
   - Vendor & not yet forked → `client.forkDrill(sourceKey)` → `forkedKey`; target = `forkedKey`. (Store remembers `forkedKey` so a second save in the same session reuses it.)
   - Tenant (or already forked this session) → target = current/forked key.
   - `client.saveDrillDiagram(target, workingDiagram)` → on 200: `markSaved`, re-fetch the catalog, re-select the (possibly new forked) drill so the viewer + editor reflect the persisted row. Backend emits `DrillCatalogUpdated.v1`.
5. **Abbrechen** with unsaved changes → confirm-discard dialog; on confirm `store.reset()`; **no network call, no orphan fork**.

## 7. Accessibility (WCAG 2.5.7 — the heavy part)

Extends formation-scene's proven keyboard pattern (`aria-grabbed`, `tabindex`, Space/Enter pick-drop, Escape-cancel) to a full editor. **Every pointer gesture has a keyboard equivalent** (WCAG 2.5.7 Dragging Movements; WAI-ARIA "Move Item in List").

- **Focusable elements:** each dot/arrow/zone is a `<g role="button" tabindex="0" [attr.aria-label] [attr.aria-grabbed]>`. The toolbar is `role="toolbar"` with roving tabindex; visible focus rings on all controls.
- **Move:** `Space`/`Enter` picks up the selected element (`aria-grabbed=true`); arrow keys nudge by **1 normalized unit** (`Shift`+arrow = **5 units**); `Space` drops; `Escape` cancels and restores. Pointer drag is the equivalent gesture, never the only one.
- **Add:** toolbar buttons select an "add" tool; **activating** an add tool inserts the element at **pitch centre** (50,30), selected and ready to nudge. Pointer users instead click an empty spot to place at the cursor.
- **Draw arrow:** select an arrow tool → `Space` on the **first anchor** (a dot, or an empty point) → `Space` on the **second anchor** creates the arrow (two-point pick/drop, confirmed in the brainstorm). Pointer users click-drag from start to end. `Escape` after the first anchor cancels.
- **Set zone:** zone tool → `Space` to start, arrow keys size/position a default rect, `Space` to confirm; pointer users drag a rectangle. `clear-zone` via a toolbar button / `Delete` on a selected zone.
- **Delete:** `Delete`/`Backspace` removes the selected element.
- **Announcements:** an `aria-live="polite"` region announces each op ("Spieler hinzugefügt", "Pass-Pfeil von … nach …", "gespeichert", "Eigene Kopie erstellt"). The rendered result keeps the read-only `role="img"` summary + visually-hidden ordered list (derived from the working diagram via `layoutDrillBoard`).

## 8. `<fc-pitch>` promotion — deferred (flagged per handoff)

The editor reuses `pitch.ts` as-is, making it a 2nd consumer alongside the read-only scene. Per ADR-176's promote-on-2nd-consumer rule this *could* trigger extraction, **but** both consumers live in `pack-football-ui` and share the **same 100×60 mini-pitch frame** — extracting a design-system `<fc-pitch>` brick now adds cross-layer churn for no behavioral gain. The genuine trigger is the **tactical-board (Scene 5)**, which needs a **different coordinate frame (100×120)** and is the divergent second context the promotion rule targets. **Decision: keep `pitch.ts` local; extract `<fc-pitch>` when Scene 5 lands.** (Confirmed in brainstorm.)

## 9. Error handling

- **API failure → HTTP mapping** (mirroring `pack-football-plan-tree.controller.ts`'s envelope posture): `forbidden`→403, `source-not-found`→404 (fork), `drill-not-found`/`not-a-drill`→404 (update), `invalid-input`→400, `forbidden-vendor-tier`→409 Conflict (defensive — lazy-fork-on-save should prevent the UI ever sending an update against a vendor key).
- **Client failures** reuse `DrillCatalogClientError` (`http-error`/`network-error`/`schema-mismatch`/`cancelled`).
- **Save errors keep the working diagram intact** (`saveStatus=error`, `errorReason` shown, retry affordance) — **no data loss**, the coach can retry or keep editing.
- **Invariants:** `drill-diagram-ops` clamp coords and mint valid ids so the working diagram **always satisfies `DrillDiagramSchema`**; the client also validates before sending (defensive parity with the server-side Zod gate).

## 10. Testing

- **Pure ops** (`drill-diagram-ops`): exhaustive unit tests per gesture incl. coordinate clamping, id minting, immutability, zone constraints.
- **Editor store**: undo/redo correctness, `saveStatus` transitions, fork-on-first-save targeting (vendor→forkedKey; tenant→in-place; second save reuses forkedKey).
- **Editor component**: pointer **and keyboard** interaction tests — place/move/delete dots, draw/delete arrows, set/clear zone via keys; `aria-grabbed`; live-region announcements; toolbar roving tabindex. (Uses pack-football-ui's working component-test harness, not the blocked dsa/analog harness.)
- **Client**: `forkDrill` + `saveDrillDiagram` (stubbed fetch); header trio + `x-request-id`; error surfaces.
- **Wire-schema parity**: new `ForkResponse` / `UpdateDrillDiagramResponse` parity-spec rows against the canonical `pack-football` shapes (source-text drift guard, as v1 did for `DrillDiagramSchema`).
- **API**: both endpoints — success + each failure→status mapping; `CoachPrincipal` derivation from context.
- **Gate**: `npm run ci:local` green + `npm run sonar:coverage && npm run sonar:scan` Quality Gate OK (new code needs lcov-covered tests). **Browser smoke-test the editing flow** before "done": `nx serve pack-football-api` (:3100) + `nx serve pack-football-visual-editor` (:4200), route `/t/b6c5d8e2-1234-4abc-9def-fc1a55e1a55e/p/football/coach/drills` — edit a vendor drill (place/move/delete/draw/zone via mouse **and** keyboard), Save → confirm fork + persisted diagram + re-selection; verify the accessibility tree (toolbar, `aria-grabbed`, live region).

## 11. Build order (decomposition)

1. Extract `drill-board-style.ts` from the read-only scene (refactor; viewer keeps rendering identically).
2. `drill-diagram-ops.ts` (pure) + `drill-editor.types.ts` + exhaustive op tests.
3. `drill-editor.store.ts` (working diagram, undo/redo, save status, fork targeting) + tests.
4. `drill-board-editor.component.ts` — pointer interaction + SVG + toolbar; component tests.
5. Keyboard a11y parity on the editor (pick/drop, nudge, draw-arrow two-anchor, delete, live region) + a11y tests.
6. Two HTTP endpoints (`POST …/fork`, `PUT …/diagram`) + `CoachPrincipal` derivation + controller tests.
7. `DrillCatalogClient.forkDrill` / `.saveDrillDiagram` + wire-schema mirrors + parity rows + client tests.
8. `drill-editor-panel.component.ts` — wire edit toggle, fork banner, save/cancel, re-selection into the Bibliothek; public-API exports.
9. Full gate (`ci:local` + Sonar) + browser smoke-test.

## 12. References

- Read-only v1: `libs/pack-football-ui/src/lib/generation/drill-board-scene.component.ts`, `drill-board-layout.ts`, `drill-board.types.ts`, `pitch.ts`; `libs/pack-football-ui/src/lib/drills/drill-bibliothek.component.ts`, `drill-catalog.client.ts`.
- Boundary: `libs/pack-football-ui/src/lib/data/wire-schemas.ts` + `wire-schemas-parity.spec.ts`.
- Interactive prior art: `libs/pack-football-ui/src/lib/generation/formation-scene.component.ts` (pointer + keyboard drag, `aria-grabbed`, hit-testing), `libs/pack-football-ui/src/lib/interaction/gesture-interpreter.ts`, `libs/pack-football-ui/src/lib/state/editor-store.ts`.
- Write path (already built): `libs/pack-football/src/in-ports/fork-template.use-case.ts` (`FORK_TEMPLATE_USE_CASE`) + `application/fork-template.service.ts`; `in-ports/update-drill-diagram.use-case.ts` (`UPDATE_DRILL_DIAGRAM_USE_CASE`) + `application/update-drill-diagram.service.ts`; `out-ports/manifest-intervention-catalog.repository.ts` (`forkTemplate`, `updateDrillDiagram`, `tenantTierKeys` gate).
- Contract: `libs/pack-football/src/domain/football-event.ts` (`DrillDiagramSchema`, `DrillCatalogUpdatedV1Schema`); ADR-160 §"Scene 4"; ADR-033 §4.
- API pattern to mirror: `apps/pack-football-api/src/app/pack-football-plan-tree.controller.ts` (POST body validation + failure→HTTP envelope); `pack-football-drills.controller.ts` (the v1 GET, extended here).

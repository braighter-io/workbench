# Kernel Mechanism Registry + Ontology-Gate Enforcement — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the `kernel.mechanism` registry (ADR-190), wire the typed `mechanismRef` onto `kernel.effect_declaration` (ADR-186 D2), and ship the typed contracts + charter-checker rules that make the ADR-187/189 ontology gates enforceable.

**Architecture:** `kernel.mechanism` is the **first implemented kernel catalog** (ADR-127 inv.4 mould): a vendor-tier, RLS-free, append-only versioned table. The kernel validates `id`/`version`/referential-integrity/version-pin; the prose (`hypothesis`/`expectedEffects`/`citations`) is opaque JSON at the metadata boundary. It follows the proven port + in-memory-adapter + Prisma-adapter + shared-contract-test pattern from `consent-engine`. The gate contracts (recommendation DTO, `ai-draft|human-approved` discriminant) are pure types; enforcement is charter-checker rules over pack code.

**Tech Stack:** TypeScript (ESM, `.js` import extensions), Zod, Prisma 5.22 (multiSchema, `kernel` schema), NestJS (`SubstrateModule.forRoot`), Vitest, plain-Symbol DI tokens.

**Scope:** This plan covers substrate slices **A (contracts)**, **B (registry runtime + migration)**, **C (gate contracts + checker rules)**. Slice **D** — the pack-football consuming surface that makes the gates *fire* (vs merely *enforceable*) — is a **separate follow-on plan** (`domains/exercir`, different subsystem). Until D lands, the gates are built-and-armed but dormant; this is the deliberate, ADR-187-disclosed "pinned ≠ enforced yet" state.

**Path note:** substrate repo root is `layers/substrate/`. The Explore pass returned a few paths without that prefix — **confirm exact lib paths** (`layers/substrate/libs/substrate-{contracts,runtime}/`) against the working tree before editing.

**Gate per PR:** `npm run ci:local` in `layers/substrate/` (`nx run-many -t build lint && nx run-many -t test --parallel=1`) + Sonar. Known friction (memory): nx/vitest4 executor mismatch (`nx test` exit-1-no-summary vs green `vitest run`; fresh install fixes); verifier worktree daemons can orphan + lock the main checkout's nx db (EBUSY) → `nx reset`. Never bypass pre-push hooks.

---

## File Structure

**PR-1 — Contracts (`layers/substrate/libs/substrate-contracts/`)**
- Create `src/mechanism/mechanism.ts` — `Mechanism` type + Zod schema; `MechanismRef` branded string (`mechanismId@semver`); parse/format helpers.
- Create `src/mechanism/mechanism-registry.port.ts` — `MechanismRegistry` out-port interface + `MECHANISM_REGISTRY` Symbol + typed error envelope.
- Create `src/mechanism/mechanism-registry.contract.spec.ts` — shared contract suite factory (run by both adapters).
- Create `src/mechanism/index.ts` — barrel.
- Modify `src/index.ts` — re-export `./mechanism`.
- Modify `src/testing.ts` — re-export the contract suite.
- Modify `package.json` — add `./mechanism` subpath export.

**PR-2 — Registry runtime + migration (`layers/substrate/`)**
- Modify `prisma/schema/kernel.prisma` — add `model Mechanism` (`@@map("mechanism")`, `@@schema("kernel")`, no `tenantPackId`); add `mechanismRef String? @map("mechanism_ref")` to `model EffectDeclaration`.
- Create `prisma/migrations/<ts>_kernel_mechanism_registry/migration.sql` — `kernel.mechanism` (append-only, no RLS, UNIQUE `(mechanism_id, version)`); `ALTER TABLE kernel.effect_declaration ADD COLUMN mechanism_ref TEXT`.
- Create `libs/substrate-runtime/src/mechanism/in-memory-mechanism-registry.ts` — in-memory adapter.
- Create `libs/substrate-runtime/src/mechanism/prisma-mechanism-registry.ts` — Prisma adapter (append-only + referential-integrity enforcement).
- Create `libs/substrate-runtime/src/mechanism/in-memory-mechanism-registry.spec.ts` + `prisma-mechanism-registry.contract.spec.ts` — run the shared suite against both.
- Modify `libs/substrate-runtime/src/composition-root/substrate.module.ts` — bind `MECHANISM_REGISTRY` (default in-memory; Prisma when `prismaClient` provided).

**PR-3 — Gate contracts + checker rules**
- Create `layers/substrate/libs/substrate-contracts/src/gates/recommendation.ts` — `Recommendation` DTO (non-null `mechanismRef`) + `AuthorityStatus = 'ai-draft' | 'human-approved'` discriminant + Zod.
- Modify `src/index.ts` + `package.json` (`./gates` subpath).
- Modify `D:/development/projects/de-braighter/.claude/agents/charter-checker.md` — add INV-3 (recommendation DTO carries non-null `mechanismRef`→`reviewedBy` mechanism), INV-8 (state surfaced to humans carries confidence + interpretation limits), and the `ai-draft|human-approved` authority check to the checklist.

---

## Definitions used across tasks

```typescript
// MechanismRef wire form: "<mechanismId>@<semver>", e.g. "mech_manageable_pressure_exposure@1.0.0"
// Mechanism (registry row) — typed identity, opaque prose payload (ADR-190 D1)
interface Mechanism {
  mechanismId: string;          // kebab/snake id, kernel-validated well-formedness
  version: string;              // semver, append-only (supersedes, never in-place)
  hypothesis: string;           // OPAQUE payload — kernel stores, never parses
  expectedEffects: string[];    // OPAQUE
  confidence: number;           // 0..1 (epistemic confidence in the theory)
  basis: 'literature' | 'expert' | 'tenant' | 'derived' | 'sham';
  citations: string[];          // OPAQUE
  reviewedBy: string | null;    // actor ref; INV-3 requires non-null before a rec may cite it
  supersedes: string | null;    // prior "<id>@<semver>" this version replaces
  declaredAt: string;           // ISO 8601
  retiredAt: string | null;     // soft-delete
}
```

---

## Task 1 — `MechanismRef` branded type + parse/format (PR-1)

**Files:**
- Create: `layers/substrate/libs/substrate-contracts/src/mechanism/mechanism.ts`
- Test: `layers/substrate/libs/substrate-contracts/src/mechanism/mechanism.spec.ts`

- [ ] **Step 1: Write the failing test**

```typescript
import { describe, it, expect } from 'vitest';
import { parseMechanismRef, formatMechanismRef, MechanismRefSchema } from './mechanism.js';

describe('MechanismRef', () => {
  it('round-trips a well-formed ref', () => {
    const ref = formatMechanismRef('mech_pressure', '1.2.0');
    expect(ref).toBe('mech_pressure@1.2.0');
    expect(parseMechanismRef(ref)).toEqual({ mechanismId: 'mech_pressure', version: '1.2.0' });
  });

  it('rejects a ref without a semver', () => {
    expect(MechanismRefSchema.safeParse('mech_pressure').success).toBe(false);
    expect(MechanismRefSchema.safeParse('mech_pressure@1.0').success).toBe(false);
  });
});
```

- [ ] **Step 2: Run it, verify it fails** — `cd layers/substrate && npx vitest run libs/substrate-contracts/src/mechanism/mechanism.spec.ts` → FAIL (module not found).

- [ ] **Step 3: Implement `mechanism.ts` (refs + schema)**

```typescript
import { z } from 'zod';

const SEMVER = /^\d+\.\d+\.\d+$/;
const MECH_ID = /^[a-z][a-z0-9_]*$/;

export const MechanismRefSchema = z
  .string()
  .regex(/^[a-z][a-z0-9_]*@\d+\.\d+\.\d+$/, 'mechanismRef must be "<id>@<semver>"');
export type MechanismRef = z.infer<typeof MechanismRefSchema>;

export function formatMechanismRef(mechanismId: string, version: string): MechanismRef {
  return `${mechanismId}@${version}` as MechanismRef;
}
export function parseMechanismRef(ref: string): { mechanismId: string; version: string } {
  const parsed = MechanismRefSchema.parse(ref);
  const [mechanismId, version] = parsed.split('@');
  return { mechanismId, version };
}

export const MechanismSchema = z.object({
  mechanismId: z.string().regex(MECH_ID),
  version: z.string().regex(SEMVER),
  hypothesis: z.string().min(1),
  expectedEffects: z.array(z.string()),
  confidence: z.number().min(0).max(1),
  basis: z.enum(['literature', 'expert', 'tenant', 'derived', 'sham']),
  citations: z.array(z.string()),
  reviewedBy: z.string().nullable(),
  supersedes: z.string().nullable(),
  declaredAt: z.string(),
  retiredAt: z.string().nullable(),
});
export type Mechanism = z.infer<typeof MechanismSchema>;
```

- [ ] **Step 4: Run it, verify it passes** — same command → PASS.
- [ ] **Step 5: Commit** — `git add layers/substrate/libs/substrate-contracts/src/mechanism && git commit -m "feat(contracts): MechanismRef + Mechanism schema"`

---

## Task 2 — `MechanismRegistry` out-port + error envelope (PR-1)

**Files:**
- Create: `layers/substrate/libs/substrate-contracts/src/mechanism/mechanism-registry.port.ts`
- Create: `layers/substrate/libs/substrate-contracts/src/mechanism/index.ts`
- Modify: `src/index.ts`, `src/testing.ts`, `package.json`

- [ ] **Step 1: Define the port** (no test — interface only; the contract suite in Task 3 is its test)

```typescript
import type { Mechanism, MechanismRef } from './mechanism.js';

export const MECHANISM_REGISTRY = Symbol.for('@de-braighter/substrate-contracts/MECHANISM_REGISTRY');

export type MechanismRegistryError =
  | { kind: 'not-found'; ref: string }
  | { kind: 'version-conflict'; ref: string }       // re-declaring an existing (id,version)
  | { kind: 'validation-failed'; message: string };

export type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };

export interface MechanismRegistry {
  /** Append a new mechanism version. Fails version-conflict if (id,version) exists (append-only). */
  declare(mechanism: Mechanism): Promise<Result<Mechanism, MechanismRegistryError>>;
  /** Resolve a pinned ref to its exact version. Fails not-found if absent or retired. */
  resolve(ref: MechanismRef): Promise<Result<Mechanism, MechanismRegistryError>>;
  /** Referential-integrity check used by INV-3 / effect-declaration validation. */
  exists(ref: MechanismRef): Promise<boolean>;
}
```

- [ ] **Step 2: Barrel + exports** — `index.ts` re-exports `./mechanism.js` + `./mechanism-registry.port.js`; add to `src/index.ts`; add `"./mechanism"` subpath to `package.json` exports (mirror the `./inference` entry).
- [ ] **Step 3: Build to verify wiring** — `cd layers/substrate && npx nx build substrate-contracts` → success.
- [ ] **Step 4: Commit** — `git commit -am "feat(contracts): MechanismRegistry out-port + MECHANISM_REGISTRY token"`

---

## Task 3 — Shared contract suite (PR-1)

**Files:**
- Create: `layers/substrate/libs/substrate-contracts/src/mechanism/mechanism-registry.contract.spec.ts`
- Modify: `src/testing.ts` (re-export `runMechanismRegistryContract`)

Mirror `consent-engine`'s contract-suite factory pattern (`prisma-consent-receipt.repository.contract.spec.ts` + `src/testing.ts`).

- [ ] **Step 1: Write the suite factory**

```typescript
import { describe, it, expect, beforeEach } from 'vitest';
import type { MechanismRegistry } from './mechanism-registry.port.js';
import { formatMechanismRef, type Mechanism } from './mechanism.js';

const fixture = (over: Partial<Mechanism> = {}): Mechanism => ({
  mechanismId: 'mech_pressure', version: '1.0.0', hypothesis: 'h', expectedEffects: [],
  confidence: 0.5, basis: 'expert', citations: [], reviewedBy: 'actor_1',
  supersedes: null, declaredAt: '2026-05-29', retiredAt: null, ...over,
});

export function runMechanismRegistryContract(makeRegistry: () => MechanismRegistry): void {
  describe('MechanismRegistry contract', () => {
    let reg: MechanismRegistry;
    beforeEach(() => { reg = makeRegistry(); });

    it('declares then resolves a pinned ref', async () => {
      await reg.declare(fixture());
      const got = await reg.resolve(formatMechanismRef('mech_pressure', '1.0.0'));
      expect(got.ok).toBe(true);
    });

    it('is append-only: re-declaring (id,version) is a version-conflict', async () => {
      await reg.declare(fixture());
      const again = await reg.declare(fixture({ hypothesis: 'rewritten' }));
      expect(again.ok).toBe(false);
      if (!again.ok) expect(again.error.kind).toBe('version-conflict');
    });

    it('a new version coexists with the old (no in-place rewrite)', async () => {
      await reg.declare(fixture({ version: '1.0.0' }));
      await reg.declare(fixture({ version: '1.1.0', supersedes: 'mech_pressure@1.0.0' }));
      expect((await reg.resolve(formatMechanismRef('mech_pressure', '1.0.0'))).ok).toBe(true);
      expect((await reg.resolve(formatMechanismRef('mech_pressure', '1.1.0'))).ok).toBe(true);
    });

    it('resolve of an absent ref is not-found', async () => {
      const got = await reg.resolve(formatMechanismRef('mech_absent', '1.0.0'));
      expect(got.ok).toBe(false);
    });
  });
}
```

- [ ] **Step 2: Re-export from `testing.ts`** — `export { runMechanismRegistryContract } from './mechanism/mechanism-registry.contract.spec.js';`
- [ ] **Step 3: Commit** — `git commit -am "test(contracts): MechanismRegistry shared contract suite"`

---

## Task 4 — In-memory adapter (PR-1 → first green adapter)

**Files:**
- Create: `layers/substrate/libs/substrate-runtime/src/mechanism/in-memory-mechanism-registry.ts`
- Create: `layers/substrate/libs/substrate-runtime/src/mechanism/in-memory-mechanism-registry.spec.ts`

- [ ] **Step 1: Failing spec wiring the shared suite**

```typescript
import { runMechanismRegistryContract } from '@de-braighter/substrate-contracts/testing';
import { InMemoryMechanismRegistry } from './in-memory-mechanism-registry.js';
runMechanismRegistryContract(() => new InMemoryMechanismRegistry());
```

- [ ] **Step 2: Run, verify fail** — `npx vitest run libs/substrate-runtime/src/mechanism/in-memory-mechanism-registry.spec.ts` → FAIL.
- [ ] **Step 3: Implement the in-memory adapter** — a `Map<string, Mechanism>` keyed by `<id>@<version>`; `declare` rejects existing key with `version-conflict`; `resolve` returns `not-found` when absent or `retiredAt != null`; `exists` mirrors `resolve`.
- [ ] **Step 4: Run, verify pass.**
- [ ] **Step 5: Commit** — `git commit -am "feat(runtime): in-memory MechanismRegistry adapter (contract green)"`

> **PR-1 boundary:** open PR with Tasks 1–4. Verifier wave + `ci:local`. The contracts package + in-memory adapter are self-contained and useful (the gates can type-check against them) before the DB lands.

---

## Task 5 — Prisma model + migration (PR-2)

**Files:**
- Modify: `layers/substrate/prisma/schema/kernel.prisma`
- Create: `layers/substrate/prisma/migrations/<ts>_kernel_mechanism_registry/migration.sql`

- [ ] **Step 1: Add the Prisma model + the `mechanismRef` column**

```prisma
model Mechanism {
  mechanismId     String   @map("mechanism_id")
  version         String
  hypothesis      String
  expectedEffects Json     @map("expected_effects")
  confidence      Float
  basis           String
  citations       Json
  reviewedBy      String?  @map("reviewed_by")
  supersedes      String?
  declaredAt      DateTime @default(now()) @map("declared_at") @db.Timestamptz()
  retiredAt       DateTime? @map("retired_at") @db.Timestamptz()

  @@id([mechanismId, version])
  @@index([mechanismId], map: "idx_mechanism_id")
  @@map("mechanism")
  @@schema("kernel")
}
// In model EffectDeclaration, add:
//   mechanismRef String? @map("mechanism_ref")   // logical ref "<id>@<semver>"; no @relation (ADR-027 inv.5)
```

- [ ] **Step 2: Generate the migration** — `cd layers/substrate && npx prisma migrate dev --name kernel_mechanism_registry --create-only` (create-only so we can harden the SQL by hand).
- [ ] **Step 3: Harden `migration.sql`** — append-only, **no RLS** (vendor-tier shared catalog, unlike `plan_node`); add a `mechanism_ref` column to `effect_declaration`:

```sql
CREATE TABLE "kernel"."mechanism" (
    "mechanism_id"     TEXT NOT NULL,
    "version"          TEXT NOT NULL,
    "hypothesis"       TEXT NOT NULL,
    "expected_effects" JSONB NOT NULL,
    "confidence"       DOUBLE PRECISION NOT NULL,
    "basis"            TEXT NOT NULL,
    "citations"        JSONB NOT NULL,
    "reviewed_by"      TEXT,
    "supersedes"       TEXT,
    "declared_at"      TIMESTAMPTZ NOT NULL DEFAULT now(),
    "retired_at"       TIMESTAMPTZ,
    CONSTRAINT "mechanism_pkey" PRIMARY KEY ("mechanism_id","version"),
    CONSTRAINT "mechanism_confidence_range" CHECK ("confidence" >= 0 AND "confidence" <= 1)
);
CREATE INDEX "idx_mechanism_id" ON "kernel"."mechanism" ("mechanism_id");
-- NOTE: deliberately NO row-level security: kernel.mechanism is a vendor-tier shared catalog
-- (ADR-190 D3 / ADR-127 inv.4), read by all tenants, not pack-owned rows.
DO $$ BEGIN
  IF EXISTS (SELECT 1 FROM pg_roles WHERE rolname = 'app') THEN
    GRANT SELECT ON "kernel"."mechanism" TO app;            -- read-all
    GRANT INSERT ON "kernel"."mechanism" TO app;            -- append-only (no UPDATE/DELETE grant)
  END IF;
END $$;

ALTER TABLE "kernel"."effect_declaration" ADD COLUMN "mechanism_ref" TEXT;
```

- [ ] **Step 4: Apply + verify** — `npx prisma migrate dev` (applies); `npx prisma generate`. Verify table exists and `app` has no UPDATE grant (append-only at the grant level).
- [ ] **Step 5: Commit** — `git add prisma && git commit -m "feat(kernel): kernel.mechanism registry table + effect_declaration.mechanism_ref (ADR-190/186)"`

---

## Task 6 — Prisma adapter + contract test (PR-2)

**Files:**
- Create: `layers/substrate/libs/substrate-runtime/src/mechanism/prisma-mechanism-registry.ts`
- Create: `layers/substrate/libs/substrate-runtime/src/mechanism/prisma-mechanism-registry.contract.spec.ts`

- [ ] **Step 1: DB-gated contract spec** (mirror `prisma-consent-receipt.repository.contract.spec.ts`: `describe.skipIf(!process.env.SUBSTRATE_DATABASE_URL)`, `beforeAll` connect, `afterAll` truncate).

```typescript
import { runMechanismRegistryContract } from '@de-braighter/substrate-contracts/testing';
import { PrismaMechanismRegistry } from './prisma-mechanism-registry.js';
// describe.skipIf(!process.env.SUBSTRATE_DATABASE_URL) wrapping runMechanismRegistryContract(
//   () => new PrismaMechanismRegistry(prisma)); truncate kernel.mechanism between cases.
```

- [ ] **Step 2: Run, verify fail.**
- [ ] **Step 3: Implement `PrismaMechanismRegistry`** — `declare` does `create` and maps the Prisma `P2002` unique-violation to `version-conflict` (note: if any path uses raw SQL, also duck-type `P2010` + SQLSTATE `23505` per the raw-unsafe-error-codes memory — the ORM `create` path here yields `P2002`); `resolve` does `findUnique({ where: { mechanismId_version } })`, returns `not-found` if null or `retiredAt != null`; `exists` mirrors. No tenant GUC needed (vendor catalog, no RLS).
- [ ] **Step 4: Run with DB** — `SUBSTRATE_DATABASE_URL=... npx vitest run libs/substrate-runtime/src/mechanism/prisma-mechanism-registry.contract.spec.ts` → PASS (same suite as in-memory).
- [ ] **Step 5: Commit** — `git commit -am "feat(runtime): Prisma MechanismRegistry adapter (contract green vs DB)"`

---

## Task 7 — Composition-root wiring (PR-2)

**Files:**
- Modify: `layers/substrate/libs/substrate-runtime/src/composition-root/substrate.module.ts`

- [ ] **Step 1: Bind the token** — add to `forRoot` providers, defaulting to in-memory, Prisma when a client is supplied (mirror the `AUDIT_EVENT_REPOSITORY` binding):

```typescript
{
  provide: MECHANISM_REGISTRY,
  useFactory: () => options?.prismaClient
    ? new PrismaMechanismRegistry(options.prismaClient as PrismaClient)
    : new InMemoryMechanismRegistry(),
},
```

Add `MECHANISM_REGISTRY` to the import from `@de-braighter/substrate-contracts` and to module `exports`.

- [ ] **Step 2: Build + module smoke test** — `npx nx build substrate-runtime` and run the existing composition-root spec (if present) to confirm `forRoot()` resolves the new provider.
- [ ] **Step 3: Commit** — `git commit -am "feat(runtime): wire MECHANISM_REGISTRY into SubstrateModule.forRoot"`

> **PR-2 boundary:** open PR with Tasks 5–7. Verifier wave + `ci:local`. `reviewer` must confirm the greenfield claim (no prior mechanism storage) holds at merge time — charter-checker's ADR-190 implementation flag.

---

## Task 8 — Gate contracts: recommendation DTO + authority discriminant (PR-3)

**Files:**
- Create: `layers/substrate/libs/substrate-contracts/src/gates/recommendation.ts`
- Create: `layers/substrate/libs/substrate-contracts/src/gates/recommendation.spec.ts`
- Modify: `src/index.ts`, `package.json` (`./gates` subpath)

- [ ] **Step 1: Failing test** — a recommendation with a null/missing `mechanismRef` fails parse; an `ai-draft` rec parses but is flagged non-authoritative.

```typescript
import { describe, it, expect } from 'vitest';
import { RecommendationSchema } from './recommendation.js';

describe('Recommendation gate contract', () => {
  it('rejects a recommendation without a mechanismRef (INV-3)', () => {
    expect(RecommendationSchema.safeParse({ id: 'r1', authority: 'human-approved' }).success).toBe(false);
  });
  it('accepts a mechanism-backed recommendation', () => {
    const r = { id: 'r1', mechanismRef: 'mech_pressure@1.0.0', authority: 'ai-draft' as const, body: '...' };
    expect(RecommendationSchema.safeParse(r).success).toBe(true);
  });
});
```

- [ ] **Step 2: Run, verify fail.**
- [ ] **Step 3: Implement**

```typescript
import { z } from 'zod';
import { MechanismRefSchema } from '../mechanism/mechanism.js';

export const AuthorityStatusSchema = z.enum(['ai-draft', 'human-approved']);
export type AuthorityStatus = z.infer<typeof AuthorityStatusSchema>;

export const RecommendationSchema = z.object({
  id: z.string(),
  mechanismRef: MechanismRefSchema,          // NON-nullable — INV-3: a rec must cite a mechanism
  authority: AuthorityStatusSchema,          // INV/189 D3: ai-draft is non-authoritative until flipped
  body: z.string(),
});
export type Recommendation = z.infer<typeof RecommendationSchema>;
```

- [ ] **Step 4: Run, verify pass; barrel + `./gates` subpath export; build.**
- [ ] **Step 5: Commit** — `git commit -am "feat(contracts): recommendation DTO + ai-draft|human-approved authority gate types (ADR-187/189)"`

---

## Task 9 — charter-checker enforcement rules (PR-3)

**Files:**
- Modify: `D:/development/projects/de-braighter/.claude/agents/charter-checker.md`

This is an agent-definition edit (no TDD; it changes how the verifier reasons). Add an "Ontology-integrity gates (ADR-187/189)" checklist block instructing the agent, on any PR touching a subject-affecting pack, to assert:

- [ ] **INV-3** — every recommendation surfaced to a user is typed as `Recommendation` (from `@de-braighter/substrate-contracts/gates`) carrying a non-null `mechanismRef` that resolves to a registry mechanism whose `reviewedBy` is non-null. BLOCK if a recommendation DTO omits `mechanismRef` or cites an unreviewed mechanism.
- [ ] **INV-8** — any State/posterior rendered to a human carries its confidence + interpretation limits (no bare label). Flag bare-label surfaces.
- [ ] **189 authority** — AI-authored content is typed `authority: 'ai-draft'` and is non-authoritative until a human action flips it to `'human-approved'`. BLOCK an `ai-draft` value presented as decision-grade.
- [ ] **Step 2: Self-check the wording** — the rules must say the kernel never sees "recommendation"; these are checks over **pack** code (ring boundary, ADR-187 D2). 
- [ ] **Step 3: Commit** (in the workbench repo) — `git commit -am "feat(charter-checker): INV-3/INV-8 + ai-draft authority gate rules (ADR-187/189)"`

> **PR-3 boundary:** contracts PR in `layers/substrate`; the charter-checker edit is a separate workbench PR (different repo). Both are dormant-but-armed until slice D (a pack authors a `Recommendation`).

---

## Out of scope — Slice D (separate follow-on plan) — ⏸ PARKED 2026-05-29

Pack-football's recommendation surface (`domains/exercir`) that authors mechanisms into `kernel.mechanism`, emits `Recommendation` DTOs through the gate, and exercises INV-3/189 end-to-end. This is what flips the gates from *armed* to *firing* and creates the second-consumer demand the registry was promoted ahead of. It is a distinct subsystem (pack repo, UI, pack roles) and gets its own plan: `docs/superpowers/plans/<date>-pack-football-recommendation-surface.md`.

### Status: PARKED (2026-05-29) — do not start yet

Parked by founder decision after PR-1/2/3 merged. **Two blockers, both must clear first:**

1. **Unpublished dependency.** `domains/exercir` consumes `@de-braighter/substrate-{contracts@^0.5.0, runtime@^0.6.0}`. The mechanism registry, `MECHANISM_REGISTRY`, and the `@de-braighter/substrate-contracts/gates` DTOs only merged to substrate `main` (unpublished, ≥0.7). Slice D cannot `import { MECHANISM_REGISTRY }` / `Recommendation` until substrate **cuts a release** and exercir **bumps to it**.
2. **Concurrent migration (another session).** Pack-football is being actively migrated onto the substrate in another session — in-flight branches (no PRs yet): `feat/event-sourcing-rewire`, `feat/rewire-plan-tree-to-kernel`, `chore/bump-substrate-arc2`, `fix/api-app-substrate-dep-bump`, `docs/migration-map-plan-tree-done`. The dep-bump branches overlap blocker #1 directly. **Do not write `domains/exercir` code until that migration lands** (collision risk).

**Unblock checklist (resume slice D only when all true):**

- [ ] Substrate cuts a release (≥0.7) containing the mechanism registry + `gates` DTOs.
- [ ] exercir bumps `@de-braighter/substrate-{contracts,runtime}` to that release.
- [ ] The football substrate-migration branches are merged to exercir `main`.

### Readiness findings (explorer, 2026-05-29 — true at park time, re-verify on resume)

Once unblocked, slice D is a **small vertical** — exercir is already substrate-wired and hexagonal:

- **Substrate wiring present**: `apps/pack-football-api/src/app/app.module.ts` calls `SubstrateModule.forRoot({...})`; inference backbone wired in `libs/pack-football/src/inference/inference-backbone.providers.ts`. Adding `MECHANISM_REGISTRY` follows the existing DI pattern.
- **Hexagonal layout**: `libs/pack-football/src/{domain,in-ports,application,out-ports,out-adapters,api,manifest}`; ~37 use-cases. Slice D adds a new in-port + service + HTTP controller + DTO (contracts in `libs/pack-football-contracts/`).
- **Recommendation surface is greenfield** — zero `recommend`/`Recommendation`/`advice` hits in pack-football (reverse-planner was POC-only, never shipped).
- **`subjectSensitivity` is net-new** — `SubjectSchema` is `person|team` (no sensitivity field, no consent/age-gate plumbing). Slice D adds the pack's `subjectSensitivity: 'developmental-minor'` declaration; full consent plumbing defers to a later slice.
- **Gate**: `npm run ci:local` (`nx run-many -t build lint && nx run-many -t test --parallel=1`), Vitest 4; schema-parity spec at `libs/pack-football/src/events/schemas-parity.spec.ts` if a kernel event is emitted.
- **Est.** ~2–3 day vertical once the prerequisites clear; the dominant historical cost was the prerequisite, not the slice.

---

## Self-Review

**Spec coverage** — ADR-190 (kernel.mechanism table, append-only, vendor-tier, no RLS, opaque prose): Tasks 5/6. ADR-186 D2 (`mechanismRef`): Task 5 (column) + Task 8 (typed gate use); the *EffectDeclaration TS contract* field is deferred (the contract doesn't exist yet — noted in Architecture). ADR-186 D3 (version-pin reproducibility as kernel guarantee): enforced by Task 5's append-only grants + Task 6's `version-conflict`. ADR-187 INV-3/INV-8 + ADR-189 authority: Tasks 8/9. Catalog pattern (ADR-127 inv.4): Tasks 1–7 establish it (first kernel catalog).

**Placeholder scan** — no TBDs; every code step has real code. The one honest deferral (EffectDeclaration TS contract field) is called out, not hidden.

**Type consistency** — `Mechanism`/`MechanismRef`/`MechanismRegistry`/`MECHANISM_REGISTRY`/`Recommendation`/`AuthorityStatus` are used identically across Tasks 1→9. `mechanismRef` wire form `"<id>@<semver>"` consistent (Task 1 schema, Task 5 column, Task 8 DTO).

**Known risk** — `nx test` vitest4 executor mismatch (run `npx vitest run` directly to disambiguate; fresh install if `nx test` exits 1 with no summary); worktree daemon EBUSY → `nx reset` (both in memory).

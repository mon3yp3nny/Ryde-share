# Architecture Research

**Domain:** Multi-tenant, API-first, configurable ride-coordination engine
**Researched:** 2026-06-02
**Confidence:** HIGH (core patterns); MEDIUM (GCP-specific tooling choices)

---

## Standard Architecture

### System Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                          API Gateway Layer                           │
│  ┌───────────────┐  ┌──────────────────┐  ┌──────────────────────┐  │
│  │  Anon Token   │  │  API Key / JWT   │  │  Rate Limit          │  │
│  │  Middleware   │  │  Auth Middleware  │  │  Middleware          │  │
│  └───────┬───────┘  └────────┬─────────┘  └──────────┬───────────┘  │
└──────────┼───────────────────┼────────────────────────┼─────────────┘
           │                   │                        │
           └───────────────────┴────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────┐
│                     Ride-Coordination Module                         │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                     Public Module API (Facade)                  │  │
│  │  createDrive / joinDrive / offerSeat / claimSeat /             │  │
│  │  assignSeat / getCoverageView / getDrive / listDrives          │  │
│  └───────────────────────────┬────────────────────────────────────┘  │
│                              │                                       │
│  ┌───────────────┐  ┌────────▼────────┐  ┌─────────────────────┐    │
│  │  Drive        │  │  Seat           │  │  Config/Policy      │    │
│  │  Domain       │  │  Domain         │  │  Domain             │    │
│  │  Service      │  │  Service        │  │  Service            │    │
│  └───────┬───────┘  └────────┬────────┘  └──────────┬──────────┘    │
│          │                   │                       │               │
│  ┌───────▼───────────────────▼───────────────────────▼────────────┐  │
│  │                     Domain Repository Layer                     │  │
│  └───────────────────────────┬────────────────────────────────────┘  │
└──────────────────────────────┼──────────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────────┐
│                         Infrastructure Layer                         │
│                                                                      │
│  ┌─────────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │  PostgreSQL         │  │  Token Store    │  │  Rate Limit     │  │
│  │  (Cloud SQL)        │  │  (Cloud         │  │  Store          │  │
│  │  Shared Schema +    │  │  Memorystore    │  │  (Memorystore   │  │
│  │  RLS / tenant_id    │  │  / Redis)       │  │  / Redis)       │  │
│  └─────────────────────┘  └─────────────────┘  └─────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

| Component | Responsibility | What It Owns |
|-----------|---------------|--------------|
| API Gateway Layer | Auth, rate limiting, routing, anon token parsing | Request lifecycle before business logic |
| Module Public Facade | The single stable contract: all use-case entry points | Public method signatures and DTOs |
| Drive Domain Service | Create/read/update drives, ownership, state | Drive aggregate, Stop aggregate |
| Seat Domain Service | Seat offers, claims, driver assignments, capacity enforcement | Seat entity, capacity policy enforcement |
| Config/Policy Domain Service | Read and validate drive-level config (knobs), inherit org defaults | DriveConfig entity, default resolution |
| Domain Repository Layer | DB abstraction; always filters by tenant_id | SQL queries, JSONB access patterns |
| Infrastructure / Persistence | PostgreSQL on Cloud SQL with shared schema | Tables, RLS policies, indexes |
| Token Store (Redis) | Anon participant token nonce tracking, API key lookup cache | Short-lived tokens, rate limit counters |

---

## Recommended Project Structure

```
src/
├── api/                        # HTTP entry points (Express/Hono router)
│   ├── middleware/             # Auth, rate-limit, tenant-context middleware
│   │   ├── anon-token.ts       # Validates signed participant tokens
│   │   ├── api-key.ts          # Validates API consumer keys
│   │   └── rate-limit.ts       # Token-bucket enforcement via Redis
│   └── routes/                 # Route definitions — thin, delegate to module facade
│       └── drives.ts
│
├── modules/
│   └── ride-coordination/      # The self-contained module
│       ├── index.ts            # PUBLIC: exports only the facade + DTOs
│       │
│       ├── facade.ts           # Module facade — the ONLY public entry point
│       │                       # (other modules / HTTP layer import this only)
│       │
│       ├── domain/             # Business logic — zero framework deps
│       │   ├── drive.ts        # Drive aggregate + domain rules
│       │   ├── stop.ts         # Stop / waypoint value object
│       │   ├── seat.ts         # Seat entity
│       │   ├── participant.ts  # Participant entity
│       │   └── drive-config.ts # DriveConfig value object + validation
│       │
│       ├── services/           # Orchestrates domain + repos
│       │   ├── drive.service.ts
│       │   ├── seat.service.ts
│       │   └── config.service.ts
│       │
│       ├── repositories/       # DB queries; always tenant-scoped
│       │   ├── drive.repo.ts
│       │   ├── seat.repo.ts
│       │   └── participant.repo.ts
│       │
│       └── dto/                # Input/output shapes for the facade
│           ├── create-drive.dto.ts
│           └── coverage-view.dto.ts
│
├── shared/                     # Truly cross-cutting code
│   ├── tenant-context.ts       # Request-scoped tenant_id propagation
│   ├── errors.ts               # Domain error types
│   └── token.ts                # Anon token sign/verify utilities
│
└── infra/
    ├── db/                     # DB connection, migrations, seeds
    └── cache/                  # Redis client, token/rate-limit helpers
```

### Structure Rationale

- **`modules/ride-coordination/index.ts` exports only `facade.ts` and DTOs.** Nothing else is importable from outside. This is enforced by barrel exports — internal services, domain objects, and repositories are never re-exported.
- **`domain/` has zero framework or ORM dependencies.** Pure TypeScript classes/functions. This makes domain logic unit-testable without booting the app.
- **`shared/tenant-context.ts` uses AsyncLocalStorage** to propagate `tenant_id` through the request lifecycle without threading it through every function call — repositories read it directly.
- **`api/` is thin by design.** Route handlers parse/validate HTTP input and call one facade method. No business logic lives here.

---

## Architectural Patterns

### Pattern 1: Module Facade as the Only Public Contract

**What:** The ride-coordination module exposes a single `RideCoordinationFacade` class (or object) with use-case-specific methods. All external callers (HTTP routes, future embedding apps, tests) interact through this surface only. Nothing inside `modules/ride-coordination/` is importable from outside except what `index.ts` re-exports.

**When to use:** Always — for any module. This pattern makes the module embeddable: a consuming app imports the facade and gets the full engine.

**Trade-offs:** Adds a thin indirection layer. Worth it: internal refactors never break external callers, and the public surface stays auditable.

```typescript
// modules/ride-coordination/facade.ts
export class RideCoordinationFacade {
  constructor(private drives: DriveService, private seats: SeatService) {}

  async createDrive(tenantId: string, input: CreateDriveDto): Promise<DriveDto> { ... }
  async offerSeat(tenantId: string, driveId: string, input: OfferSeatDto): Promise<SeatDto> { ... }
  async claimSeat(tenantId: string, driveId: string, input: ClaimSeatDto): Promise<SeatDto> { ... }
  async getCoverageView(tenantId: string, driveId: string): Promise<CoverageViewDto> { ... }
}

// modules/ride-coordination/index.ts — ONLY public exports
export { RideCoordinationFacade } from './facade';
export type { CreateDriveDto, DriveDto, CoverageViewDto, ... } from './dto';
```

### Pattern 2: Drive Config as an Immutable Value Object (JSONB + Typed Schema)

**What:** Each drive carries a `config` column (JSONB) validated against a TypeScript schema on write. V1 has four named knobs. Adding a fifth knob later is a schema migration + code addition, not a structural change. The config type is a discriminated union or Zod schema that remains backward-compatible.

**When to use:** For any entity with a variable, extensible set of behavioral knobs. JSONB beats EAV (entity-attribute-value) tables for read performance and beats nullable columns for extensibility.

**Trade-offs:** JSONB is not easily indexed for arbitrary key queries (but drive configs are loaded by drive ID, not queried by config key, so this is fine). Schema validation at the application boundary is mandatory.

```typescript
// domain/drive-config.ts
export interface DriveConfig {
  meetingPoint: 'required' | 'optional' | 'hidden';
  seatSelectionMode: 'self-pick' | 'driver-assigns' | 'both';
  requiredRiderInfo: 'name-only' | 'name-and-contact';
  capacityBehavior: 'hard-cap' | 'waitlist';
  // Future knobs added here without schema migration to add columns
}

// Default used when tenant/drive doesn't override
export const DEFAULT_DRIVE_CONFIG: DriveConfig = {
  meetingPoint: 'optional',
  seatSelectionMode: 'both',
  requiredRiderInfo: 'name-only',
  capacityBehavior: 'hard-cap',
};
```

### Pattern 3: Shared Schema Multi-Tenancy with `tenant_id` + Async Context Propagation

**What:** Every tenant-scoped table has a `tenant_id UUID NOT NULL` column. All queries are automatically scoped by reading `tenant_id` from `AsyncLocalStorage` set in the auth middleware. Repositories never accept `tenant_id` as a parameter — they read it from context, making accidental cross-tenant leaks structurally impossible.

**When to use:** Default for all new multi-tenant SaaS. Schema-per-tenant is an anti-pattern at any non-trivial tenant count.

**Trade-offs:** Requires discipline: async context must be set before any DB call. Background jobs must set tenant context explicitly. Benefit: single migration path, no catalog bloat, operational simplicity.

```typescript
// shared/tenant-context.ts
import { AsyncLocalStorage } from 'node:async_hooks';

const store = new AsyncLocalStorage<{ tenantId: string }>();

export const TenantContext = {
  run: (tenantId: string, fn: () => unknown) => store.run({ tenantId }, fn),
  get: (): string => {
    const ctx = store.getStore();
    if (!ctx) throw new Error('No tenant context — request setup bug');
    return ctx.tenantId;
  },
};

// repositories/drive.repo.ts
async findById(driveId: string): Promise<Drive | null> {
  const tenantId = TenantContext.get(); // tenant_id is never passed explicitly
  return db.query('SELECT * FROM drives WHERE id = $1 AND tenant_id = $2', [driveId, tenantId]);
}
```

---

## Data Model

### Entity Overview

```
Organization (Tenant)
  └── has many Drives
        ├── has one DriveConfig (JSONB, embedded)
        ├── has many Stops (ordered; v1 = exactly 2: origin + destination)
        ├── has many Seats (offered by drivers)
        │     └── has many SeatAssignments → Participant
        └── has many Participants (everyone who joined, needing a seat)
```

### Core Tables

```sql
-- Tenant / Organization
organizations (
  id          UUID PRIMARY KEY,
  slug        TEXT UNIQUE NOT NULL,       -- used in API key lookup
  name        TEXT NOT NULL,
  created_at  TIMESTAMPTZ DEFAULT now()
)

-- Drive (the core aggregate)
drives (
  id             UUID PRIMARY KEY,
  tenant_id      UUID NOT NULL REFERENCES organizations(id),
  title          TEXT,                    -- optional human label
  event_id       UUID,                    -- optional linked event (future)
  status         TEXT NOT NULL DEFAULT 'open',  -- open | closed | cancelled
  config         JSONB NOT NULL,          -- DriveConfig value object
  created_at     TIMESTAMPTZ DEFAULT now(),
  updated_at     TIMESTAMPTZ DEFAULT now(),
  -- index:
  INDEX drives_tenant_idx (tenant_id),
  INDEX drives_event_idx (tenant_id, event_id)
)

-- Stop / Waypoint  ← extensibility lives here
stops (
  id          UUID PRIMARY KEY,
  drive_id    UUID NOT NULL REFERENCES drives(id) ON DELETE CASCADE,
  tenant_id   UUID NOT NULL,              -- denormalized for RLS
  position    INTEGER NOT NULL,           -- 0 = origin, N-1 = destination
  label       TEXT,                       -- "Home ground", "Away stadium"
  address     TEXT,
  lat         NUMERIC(9,6),
  lng         NUMERIC(9,6),
  -- v1 always has exactly 2 rows per drive (position 0 and 1)
  -- v2 adds intermediate positions with no schema change
  UNIQUE (drive_id, position)
)

-- Seat (offered by a driver)
seats (
  id              UUID PRIMARY KEY,
  drive_id        UUID NOT NULL REFERENCES drives(id),
  tenant_id       UUID NOT NULL,
  driver_token    TEXT NOT NULL,          -- opaque token identifying the anonymous driver
  driver_display  TEXT,                   -- display name
  capacity        INTEGER NOT NULL,
  status          TEXT NOT NULL DEFAULT 'open',  -- open | full | cancelled
  meeting_point   TEXT,                   -- only relevant if config.meetingPoint != hidden
  created_at      TIMESTAMPTZ DEFAULT now()
)

-- Participant (anyone who joined as needing a seat)
participants (
  id               UUID PRIMARY KEY,
  drive_id         UUID NOT NULL REFERENCES drives(id),
  tenant_id        UUID NOT NULL,
  token            TEXT UNIQUE NOT NULL,  -- the signed anon token value (for lookup)
  display_name     TEXT NOT NULL,
  contact_info     TEXT,                  -- null if config.requiredRiderInfo = 'name-only'
  needs_seat       BOOLEAN NOT NULL DEFAULT true,
  created_at       TIMESTAMPTZ DEFAULT now()
)

-- Seat Assignment (who is in which seat)
seat_assignments (
  id              UUID PRIMARY KEY,
  seat_id         UUID NOT NULL REFERENCES seats(id),
  participant_id  UUID NOT NULL REFERENCES participants(id),
  drive_id        UUID NOT NULL,          -- denormalized for query efficiency
  tenant_id       UUID NOT NULL,
  status          TEXT NOT NULL DEFAULT 'confirmed',  -- confirmed | waitlisted | cancelled
  assigned_at     TIMESTAMPTZ DEFAULT now(),
  UNIQUE (seat_id, participant_id)
)

-- API Consumer (authenticated callers)
api_consumers (
  id          UUID PRIMARY KEY,
  tenant_id   UUID NOT NULL REFERENCES organizations(id),
  name        TEXT NOT NULL,
  key_hash    TEXT UNIQUE NOT NULL,       -- bcrypt/argon2 hash of the raw API key
  rate_limit  INTEGER NOT NULL DEFAULT 1000,  -- requests per minute
  created_at  TIMESTAMPTZ DEFAULT now()
)
```

### Single-Stop → Multi-Stop Extensibility Path

This is the critical design decision. The `stops` table is the extensibility mechanism:

**V1 (now):** Every drive is created with exactly 2 stops inserted atomically — `position: 0` (origin) and `position: 1` (destination). Application code validates this constraint. No multi-stop logic is built.

**V2 (multi-stop):** Add rows with intermediate `position` values. The `seat_assignments` table gets an optional `origin_stop_position` and `destination_stop_position` — riders can board and alight at different stops. The only schema change is two nullable columns on `seat_assignments`. No existing rows are invalidated.

**V3 (return trip):** Add a `leg` column to `stops` (INTEGER, default 0). A second leg gets `leg: 1` with its own origin/destination stops. Coverage view becomes per-leg.

```
V1 stops:   [{ position: 0, label: 'Home' }, { position: 1, label: 'Away Ground' }]

V2 stops:   [{ position: 0 }, { position: 1, label: 'Pickup Stop' }, { position: 2 }]
            seat_assignments gets: origin_stop_position = 0, destination_stop_position = 2

V3 stops:   [{ leg: 0, position: 0 }, { leg: 0, position: 1 },
             { leg: 1, position: 0 }, { leg: 1, position: 1 }]
```

**No rewrite required** — each evolution is additive: new rows, new nullable columns, new application logic. Existing drives are unaffected.

---

## Identity Architecture

### Two Principal Types

| Principal | Identity | Lifetime | Used For |
|-----------|----------|----------|---------|
| Anonymous Participant | Signed JWT in URL/QR | Short-lived (configurable, e.g. 7 days) | Join drive, claim/offer seat — no account required |
| API Consumer | Hashed API key → JWT session | Long-lived key, short-lived session token | Authenticated callers, rate-limited, tenant-scoped |

### Anonymous Participant Token

A participant joins by following a signed URL. The URL contains a `pt` (participant token) query param — a signed JWT with:

```
{
  sub: "<participant_id>",      // UUID of the participant row
  tid: "<tenant_id>",           // which org
  did: "<drive_id>",            // which drive this token is scoped to
  role: "participant" | "driver",
  exp: <unix timestamp>,        // e.g. 7 days from issue
  jti: "<nonce>",               // unique token ID (stored in Redis for revocation)
}
```

Signed with `RS256` using a GCP-managed key (Secret Manager stores the private key; public key is loaded at startup). The token is stateless enough for signature-only validation, but the `jti` nonce is checked against Redis to enable revocation (important for seat-cancel flows).

**Token issuance:** Token is created when the participant first accesses the drive link and confirms their name. If they lose the link, a new token is issued (new `jti`, old one revocable).

### API Consumer Auth Flow

```
1. Consumer sends:  POST /auth/token  with API key in Authorization: Bearer header
2. Server:          Hash key → lookup api_consumers → verify → issue short-lived JWT (15 min)
                    JWT contains: { sub: consumer_id, tid: tenant_id, scope: "api", exp }
3. Consumer uses:   Authorization: Bearer <short-lived JWT> on all subsequent requests
4. Rate limiting:   Token bucket in Redis keyed by consumer_id, refilled per minute
```

API keys are never stored in plaintext — only argon2id hashes. Raw keys are shown once at creation.

### Middleware Stack (Request Order)

```
Request
  → parse-tenant-from-host-or-header    (resolves tenant_id for every request)
  → auth-router                         (branch: anon token vs API key vs unauthenticated)
      ├── anon-token-middleware          (validates pt param JWT, sets participant context)
      └── api-key-middleware             (validates Bearer JWT, sets consumer context)
  → rate-limit-middleware               (token bucket check, consumer_id keyed)
  → tenant-context.run(tenantId, ...)   (establishes AsyncLocalStorage scope)
  → route handler → facade
```

Anonymous participants are **not** rate-limited individually (they're scope-limited to one drive by their token). Rate limiting applies only to API consumers.

---

## Data Flow

### Participant Joins Drive

```
User scans QR / clicks link (contains drive_id + invite_token)
    ↓
GET /drives/:driveId?invite=<invite_token>
    ↓ invite_token validated (HMAC-signed, short-lived, drive-scoped)
    ↓
POST /drives/:driveId/join  { displayName, contactInfo? }
    ↓
[ParticipantService.join()] → creates participant row → issues participant JWT
    ↓
Response: { participantToken: "<JWT>" }
    ↓
Client stores token locally; all subsequent actions pass it as Bearer
```

### Coverage View Calculation

Coverage is a **pure read query** — no separate materialized state. Calculated on demand:

```
GET /drives/:driveId/coverage
    ↓
[DriveService.getCoverageView(driveId)]
    ↓
SELECT
  COUNT(*) FILTER (WHERE needs_seat = true)               AS needs_ride_count,
  SUM(s.capacity) - COUNT(sa.id) FILTER (confirmed)       AS available_seats,
  array_agg(DISTINCT p.id) FILTER (WHERE sa.id IS NULL)   AS uncovered_participants
FROM participants p
LEFT JOIN seat_assignments sa ON sa.participant_id = p.id AND sa.status = 'confirmed'
LEFT JOIN seats s ON sa.seat_id = s.id
WHERE p.drive_id = $1 AND p.tenant_id = $2 AND p.needs_seat = true
    ↓
CoverageViewDto: {
  needsRide: 12,
  covered: 9,
  shortBy: 3,           -- 0 means "everyone covered"
  availableSeats: 2,
  uncoveredParticipants: [{ id, displayName }, ...]
}
```

This is cheap at MVP scale. At high scale (1000+ participants per drive), a materialized counter updated on seat assignment events is the optimization — but that's a deferred concern.

### API Consumer Calls Drive Engine

```
API Consumer (e.g. handball-manager, future)
    ↓
POST /drives  Authorization: Bearer <JWT>
    Body: { title, config: { meetingPoint: 'required', ... }, stops: [origin, dest] }
    ↓
api-key-middleware validates JWT → sets tenant context
    ↓
RideCoordinationFacade.createDrive(tenantId, dto)
    ↓
DriveService: validate config schema → persist drive + stops (atomic transaction)
    ↓
Response: DriveDto with invite link / QR token embedded
```

---

## Multi-Tenancy Pattern

**Chosen pattern: Shared schema with `tenant_id` on every table.**

Rationale:
- Schema-per-tenant is "a trap" at any non-trivial tenant count — DDL migrations across N schemas become unmanageable (source: ClickHouse Engineering, 2026).
- Database-per-tenant is appropriate only for regulated/white-label workloads — overkill here.
- Shared schema with `tenant_id` has lowest operational overhead, single migration path, and is the recommended default for new SaaS in 2026.

Enforcement layers (defense in depth, not just relying on one):
1. **Application layer:** `TenantContext.get()` injected into every repository call; no query runs without tenant scope.
2. **PostgreSQL RLS (defense in depth):** `CREATE POLICY tenant_isolation ON drives USING (tenant_id = current_setting('app.current_tenant_id')::uuid)` — set via `SET LOCAL app.current_tenant_id = '...'` before each transaction. This catches any query that bypasses the application layer (migrations, ad-hoc queries).
3. **Integration tests:** Every repository test asserts that fetching entity X from tenant A is invisible to tenant B.

---

## Scaling Considerations

| Scale | Architecture Adjustments |
|-------|--------------------------|
| 0-50 orgs, ~100 drives/day | Single Cloud Run service, Cloud SQL (shared schema), Memorystore Redis for tokens/rate-limit. All on one deployment. |
| 50-500 orgs, ~1k drives/day | Add Cloud SQL read replica for coverage view queries. Add Cloud CDN for static assets if frontend exists. No structural change needed. |
| 500+ orgs, high write volume | Coverage counter materialization (update counter on seat claim, read counter instead of aggregation). Connection pooling via PgBouncer or Cloud SQL Proxy in transaction mode. |
| Embed in handball-manager | handball-manager imports `RideCoordinationFacade` directly (in-process) OR calls the REST API. No structural change to ryde-share needed. |

### Scaling Priorities

1. **First bottleneck:** PostgreSQL connection count. Cloud Run scales horizontally; each instance opens DB connections. PgBouncer or Cloud SQL Proxy (transaction mode) is the fix.
2. **Second bottleneck:** Coverage view aggregation at large participant counts. Replace with a Redis counter or materialized `drive_coverage_cache` table updated on assignment events.

---

## Build Order (Component Dependencies)

Dependencies flow strictly downward. Each layer can be built and tested before the next.

```
Phase 1 — Foundation (no shortcuts possible)
  ├── Database schema: organizations, drives, stops, participants, seats, seat_assignments
  ├── TenantContext (AsyncLocalStorage propagation)
  ├── Anon token sign/verify (shared/token.ts)
  └── Base repository layer (tenant-scoped queries)

Phase 2 — Drive Core (depends on Phase 1)
  ├── DriveService: createDrive, getDrive, listDrives
  ├── StopService: createStops (always 2 for v1)
  ├── ConfigService: validate DriveConfig, apply defaults
  └── Facade: createDrive, getDrive exposed

Phase 3 — Seat & Participant (depends on Phase 2)
  ├── ParticipantService: join (issues anon token), get participant
  ├── SeatService: offerSeat, claimSeat, assignSeat (respects config.seatSelectionMode)
  ├── Capacity enforcement (hard-cap vs waitlist per config.capacityBehavior)
  └── Facade: offerSeat, claimSeat, assignSeat exposed

Phase 4 — Coverage View (depends on Phase 3)
  ├── CoverageService: getCoverageView (pure aggregation query)
  └── Facade: getCoverageView exposed

Phase 5 — API Consumer Auth + Rate Limiting (can parallel-track with Phase 3/4)
  ├── api_consumers table + key hashing
  ├── Token exchange endpoint (POST /auth/token)
  └── Rate limit middleware (Redis token bucket)

Phase 6 — HTTP Layer (depends on Phases 1-5 being complete)
  ├── Express/Hono routes (thin wrappers over Facade)
  ├── Auth middleware stack (anon token, API key, rate limit)
  ├── Request/response DTOs and validation (Zod)
  └── OpenAPI spec generation (from Zod schemas)
```

**Key sequencing insight:** Phases 1-4 can be built and fully unit/integration tested **without any HTTP layer**. The facade is callable directly in tests. This validates the core engine before adding HTTP concerns. Rate limiting (Phase 5) is independently testable against the Redis middleware. HTTP routing (Phase 6) then becomes a thin integration exercise.

---

## Anti-Patterns

### Anti-Pattern 1: Leaking Module Internals

**What people do:** Import DriveRepository or DriveService directly from outside the module, bypassing the facade.
**Why it's wrong:** Creates invisible coupling; internal refactors break external callers; module cannot be extracted or replaced.
**Do this instead:** Enforce barrel exports — `modules/ride-coordination/index.ts` re-exports only the facade and DTOs. Use eslint-plugin-boundaries or an architectural fitness function in CI.

### Anti-Pattern 2: Schema-Per-Tenant

**What people do:** Create a separate PostgreSQL schema per tenant for "isolation."
**Why it's wrong:** At 50+ tenants, DDL migrations across N schemas become a deployment nightmare. PostgreSQL catalog bloats. Schema drift happens. This pattern doesn't scale operationally.
**Do this instead:** Shared schema with `tenant_id` + RLS. Simple, maintainable, secure when enforced at multiple layers.

### Anti-Pattern 3: Nullable Column Sprawl for Config Knobs

**What people do:** Add `meeting_point_required BOOLEAN`, `seat_selection_mode TEXT`, etc. as separate nullable columns on the `drives` table each time a new knob is needed.
**Why it's wrong:** Every new knob is a migration + nullable column. 10 knobs = 10 nullable columns of unclear semantics. No schema evolution story.
**Do this instead:** JSONB `config` column with typed schema. New knobs are additive to the schema definition; old rows get default values on read. One migration for the column; zero migrations for new knobs.

### Anti-Pattern 4: Hard-Coding Single Start→Destination

**What people do:** Add `origin_address TEXT` and `destination_address TEXT` columns directly on `drives`.
**Why it's wrong:** Multi-stop is explicitly in the extensibility requirements. These columns can never grow into legs/stops without a rewrite. Migrating address data out of the drive row into a normalized stops table later is painful.
**Do this instead:** `stops` table from day one with `position` ordering. V1 inserts exactly 2 rows. The table is the extension point, not the drives row.

### Anti-Pattern 5: Coverage View as Cached/Materialized State (Premature)

**What people do:** Add a `covered_count` column to `drives` and update it on every seat assignment.
**Why it's wrong:** Introduces synchronization complexity early. Any bug in the update logic causes coverage state drift. Not needed at MVP scale.
**Do this instead:** Calculate coverage on demand via aggregation query. Materialize only when query time becomes a real problem (measurable at scale).

---

## Integration Points

### External Services

| Service | Integration Pattern | Notes |
|---------|---------------------|-------|
| GCP Cloud SQL (PostgreSQL) | Direct connection via Cloud SQL Auth Proxy or private IP | Use transaction-mode PgBouncer at scale |
| GCP Cloud Memorystore (Redis) | Redis client (ioredis) for token nonces and rate-limit counters | Single Redis instance fine for MVP |
| GCP Secret Manager | Load private signing key at startup; rotate without redeploy | Never store signing key in env vars or code |
| GCP Cloud Run | Single-service deployment of the modular monolith | Scales to zero; cost-effective at MVP scale |

### Internal Boundaries

| Boundary | Communication | Notes |
|----------|---------------|-------|
| HTTP routes ↔ Ride module | Facade method calls (in-process) | Routes import only from `modules/ride-coordination/index.ts` |
| Ride module ↔ DB | Repository layer (SQL via pg/postgres.js) | Always tenant-scoped via TenantContext |
| Auth middleware ↔ Token store | Redis via shared cache client | Nonce check on every anon token request |
| Future handball-manager ↔ Ride module | REST API (API consumer key pattern) | Can also embed as in-process module if co-deployed |

---

## Sources

- ClickHouse Engineering: [How to architect multi-tenant SaaS on Postgres](https://clickhouse.com/resources/engineering/multi-tenant-saas-postgres-architecture) — schema comparison, shared schema recommendation (HIGH confidence)
- Milan Jovanovic: [Internal vs Public APIs in Modular Monoliths](https://www.milanjovanovic.tech/blog/internal-vs-public-apis-in-modular-monoliths) — facade pattern, module contracts (HIGH confidence)
- Red Gate: [Creating a Data Model for Carpooling](https://www.red-gate.com/blog/creating-a-data-model-for-carpooling/) — stop/waypoint table pattern (MEDIUM confidence, adapted)
- Clerk: [How to Design Multi-Tenant SaaS Architecture](https://clerk.com/blog/how-to-design-multitenant-saas-architecture) — tenant isolation patterns (MEDIUM confidence)
- AWS Blog: [Multi-tenant data isolation with PostgreSQL Row Level Security](https://aws.amazon.com/blogs/database/multi-tenant-data-isolation-with-postgresql-row-level-security/) — RLS implementation (HIGH confidence)
- Modular Monolith DDD (Kamil Grzybek): [Modular Monolith Domain-Centric Design](https://www.kamilgrzybek.com/blog/posts/modular-monolith-domain-centric-design) — bounded context boundaries (HIGH confidence)
- Peter Hrynkow: [Schema-Driven Platforms](https://peterhrynkow.com/ai/architecture/2025/02/01/schema-driven-platforms.html) — JSONB config pattern (MEDIUM confidence)

---

*Architecture research for: multi-tenant, API-first, configurable ride-coordination engine*
*Researched: 2026-06-02*

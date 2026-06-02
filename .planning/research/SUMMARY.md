# Project Research Summary

**Project:** Ryde-share — configurable, multi-tenant, API-first ride-coordination engine
**Domain:** Group-trip / carpool coordination (flagship: sports-club away-game rides), domain-agnostic SaaS
**Researched:** 2026-06-02
**Confidence:** HIGH (stack, features, core architecture, pitfalls); MEDIUM (frontend choice, token revocation specifics)

## Executive Summary

Ryde-share is a backend-centric, API-first coordination engine: organizations create "drives" (one start → one destination per car/group), members offer seats, riders self-pick or are assigned seats (configurable per drive), and a **coverage view answers "is everyone covered?"** The research strongly validates a lean modular-monolith on GCP rather than microservices or a heavy framework — the whole domain is a relational coverage calculation behind a thin HTTP layer, embeddable later by larger apps (the user's future `handball-manager`).

The single biggest product insight: **no competitor surfaces aggregate coverage.** TeamSnap, Caroster, GoKid, SignUpGenius, GroupCarpool all show seat counts or slot states, but none computes "3 riders still need a car — *these* 3." That aggregate status + the uncovered-riders list is the differentiator to own, and it's cheap (a query, not a service). Anonymous link/QR join is the proven adoption pattern; requiring accounts is what kills casual/one-off use.

The dominant risks are not feature risks — they're correctness and compliance risks that must be designed in from Phase 1: **seat-claim race conditions** (need `SELECT FOR UPDATE` atomic allocation), **tenant data leakage** (need Postgres RLS with `SET LOCAL` inside transactions, never `SET`), **anonymous-link abuse** (signed/expiring capability URLs + edge rate limiting), **GDPR/minors** (the flagship handles children's data — Germany's Art. 8 threshold is 16; frame as "service to organizations, minimize PII, scheduled deletion"), and **Cloud Run + Cloud SQL connection exhaustion** under the natural "40 parents click the link at once" burst (need PgBouncer transaction pooling + `min-instances=1`).

## Key Findings

### Recommended Stack

A TypeScript modular monolith on Cloud Run, backed by Cloud SQL Postgres with row-level security, fronted by GCP API Gateway for the two-tier auth model. This keeps the engine small, type-safe end-to-end, and cheap at low volume while scaling cleanly.

**Core technologies:**
- **Google Cloud Run** (compute): stateless HTTP API; `--min-instances=1 --cpu-boost` (~$3/mo) kills cold starts. Chosen over GKE ($200–400/mo node baseline, no benefit) and App Engine.
- **TypeScript + Hono 4.x** (backend framework): tiny, runs on Node + edge, first-class JWT/CORS/validation middleware, RPC client + `@hono/zod-validator` for zero-duplication type safety with Drizzle + Zod.
- **Cloud SQL PostgreSQL 16 + Row-Level Security** (data): coverage is a relational join, so Postgres — not Firestore. RLS shared-schema isolation adds only ~2–4% overhead. AlloyDB rejected (26% pricier, irrelevant advantages at this scale).
- **Drizzle ORM + Zod** (data access + validation): type-safe, light, fast cold starts.
- **GCP API Gateway** (API management): consumption-priced (~$3/M calls); handles anonymous public routes + per-consumer API-key quotas. Apigee rejected (~$2,500/mo min); Cloud Endpoints rejected (self-managed ESP proxy).
- **`jose` HMAC JWTs** (anonymous participation): signed, drive-scoped, short-lived tokens — no accounts, no DB lookup to validate.
- **SvelteKit static PWA** (standalone UI) — *MEDIUM confidence, deferrable*: ~15–25 KB bundle, served from Cloud Storage + CDN (no compute). Because the build is API-first, the UI framework is a late, low-risk, swappable decision.

### Expected Features

**Must have (table stakes):**
- Create a drive (start, destination, optional event info, date/time)
- Offer seats (driver declares capacity)
- Join via shareable link / QR, anonymously (name-only or name+contact)
- Claim a seat (rider self-pick) **and** driver-assign — per-drive configurable mode
- Coverage view: covered / short-by-N **plus the list of who is still uncovered**
- Unclaim / edit your own entry without an account (token/cookie-scoped)

**Should have (competitive differentiators):**
- **Aggregate coverage status + uncovered-riders list** — the gap no competitor fills (primary differentiator)
- Per-drive configurability (meeting point required/optional/hidden; required rider info; capacity behavior) exposed without overwhelming users
- Clean public API as a first-class product surface (embeddable engine)

**Defer (v2+):**
- Notifications / push / reminders (the #1 over-built MVP feature — pull-only is sufficient)
- Multiple pickup/drop-off stops, return/round trips (schema anticipates; not built)
- Cost/fuel splitting, in-app chat, GPS/live tracking, route optimization, recurring drives
- Lasting end-user accounts; active handball-manager integration

### Architecture Approach

A modular monolith exposing a single `RideCoordinationFacade` as the only public contract (HTTP layer, tests, and future embedders import only the facade + DTOs). The domain (drives, seats, participants, coverage) is fully testable without HTTP, so the HTTP layer stays thin and is added last. Multi-tenancy is shared-schema with `tenant_id`/`org_id` on every table, enforced at two layers: application-level tenant context propagation **and** Postgres RLS as a defense-in-depth backstop. Schema-per-tenant was explicitly rejected as a 2026 anti-pattern (migration/catalog bloat).

**Major components:**
1. **Tenant & identity layer** — org/tenant model, anonymous participant tokens (signed, drive-scoped) + authenticated API-consumer credentials with rate limits; tenant-context middleware runs before all business logic.
2. **Ride-coordination domain (the module)** — Drives, Stops, Seats, Participants, per-drive Config; the `RideCoordinationFacade` public surface.
3. **Coverage engine** — pure aggregation query over the domain (no materialized state/cache for v1; optimize only if measured).
4. **HTTP/API layer** — thin Hono app + OpenAPI spec, wired to API Gateway; added after the domain is proven.
5. **Standalone web/PWA UI** — thin client over the public API; built in parallel once the API contract stabilizes.

**Key data-model decision:** a dedicated **`stops` table with a `position` column from day one**. V1 inserts exactly 2 rows per drive (origin + destination); multi-stop (V2) and return legs (V3) add rows/a `leg` column with no breaking migration. This is the cheapest way to honor "single now, multi-stop later without a rewrite" — far better than `origin`/`destination` columns on `drives`.

### Critical Pitfalls

1. **Seat-claim race conditions** — two riders grab the last seat, or driver assigns while a rider self-picks. Use `SELECT FOR UPDATE` atomic allocation from the first booking endpoint; retrofitting is dangerous. *(Phase: participation flow.)*
2. **Tenant data leakage** — convention-based `WHERE tenant_id=?` eventually fails. Use Postgres RLS with `SET LOCAL app.current_tenant_id` **inside a transaction**; `SET` (without LOCAL) leaks across pooled connections — a classic bug. *(Phase: foundation.)*
3. **Anonymous-link abuse** (enumeration, seat-stuffing, contact scraping) — use HMAC-signed, expiring capability URLs (UUIDs as external IDs), edge rate limiting (Cloud Armor, not per-instance counters), and organizer-only scoping of PII fields. *(Phase: foundation + participation.)*
4. **GDPR / minors** — flagship processes children's data; Germany's Art. 8 threshold is 16. Frame as "service to organizations," minimize PII, never surface contact info to other participants, run a scheduled deletion job anchored to drive date, mask PII in logs. *(Phase: foundation policy + operations.)*
5. **Cloud Run + Cloud SQL connection exhaustion** — bursty "40 parents at once" traffic exhausts connections. PgBouncer transaction-pooling + `min-instances=1` must be in place before the first public load. *(Phase: foundation infra.)*
6. **"Everything is a knob" / config drift** — keep the 4 v1 knobs as **typed columns**, not a free-form JSONB blob (see reconciliation). *(Phase: foundation.)*

### Reconciled Decisions (cross-doc tensions)

1. **Config storage → typed columns for v1, JSONB only as a documented escape hatch.** The 4 known v1 knobs (meeting-point mode, seat-selection mode, required rider info, capacity behavior) become explicit, validated, typed columns/enums — this prevents the undocumented "config drift" Pitfalls warned about. Keep a path to a JSONB `extra_config` (or `organizations.default_drive_config`) for genuinely open future knobs, but do not store the known knobs there. (Resolves ARCHITECTURE↔PITFALLS.)
2. **Tokens → HS256 (HMAC via `jose`) for v1.** Symmetric signing is simpler, needs no key distribution, and validates with no DB/Redis lookup. **Do not add Redis/Memorystore for v1** — revoke via an `invalidated_at`/`drive.revoked` column check (cheap, the drive row is already loaded) rather than a `jti` denylist. Revisit RS256 + jti only if/when external embedders need to verify tokens without the shared secret. (Resolves STACK↔ARCHITECTURE.)
3. **Frontend → SvelteKit static PWA, but treated as a deferrable, low-lock-in choice.** Backend/API-first build order means the UI can be built late and swapped cheaply; don't let the framework choice block engine work. (Confirms STACK with a flag.)
4. **PgBouncer + `min-instances=1` are MVP-critical, not deferrable.** Both directly mitigate the connection-exhaustion pitfall under the product's natural burst pattern and cost only a few dollars/month. Establish in the foundation phase.

## Implications for Roadmap

Suggested phase structure (backend/engine-first, vertical coverage slice early, API + UI after the domain is proven):

### Phase 1: Foundation & Tenancy
**Rationale:** Everything depends on correct tenant isolation, infra, and the extensible schema; these are the riskiest-to-retrofit decisions.
**Delivers:** Cloud Run service + Cloud SQL Postgres 16 + PgBouncer + `min-instances=1`; shared-schema with `org_id` + RLS (`SET LOCAL`); `stops` table with `position`; typed Drive config columns; UUID external IDs; tenant-context middleware; signed-token (HS256) infrastructure.
**Addresses:** multi-tenant isolation, extensible data model.
**Avoids:** tenant leakage, connection exhaustion, config drift, painful stop migration.

### Phase 2: Drives, Anonymous Participation & Coverage (core vertical slice)
**Rationale:** Drives, seat offer/claim, and coverage are tightly coupled — coverage can't be tested without all of them. This is the product's core value in one slice.
**Delivers:** create drive (+ optional event info, link/QR), offer seats, rider self-pick + driver-assign (per-drive mode), unclaim/edit-own-entry via scoped token, and the **coverage view (covered/short-by-N + uncovered list)**.
**Uses:** Hono, Drizzle, `jose` tokens, the `RideCoordinationFacade`.
**Avoids:** seat-claim race conditions (`SELECT FOR UPDATE`), anonymous-link abuse (signed/expiring capability URLs, edge rate limiting), PII over-exposure (organizer-only contact fields, join-screen disclosure).

### Phase 3: API Surface & Consumer Auth
**Rationale:** Make the engine a first-class embeddable product; can largely proceed in parallel with Phase 2 (separate tables/infra).
**Delivers:** public OpenAPI contract via API Gateway, API-consumer API keys + per-consumer rate limits/quotas, two-tier auth (anonymous public routes vs authenticated consumers), raw Cloud Run URL never exposed in shared links.
**Implements:** API management + identity layer.

### Phase 4: Standalone Web/PWA UI
**Rationale:** API-first means the UI is a thin, late, swappable client; build once the API contract is stable (parallelizable with Phase 3).
**Delivers:** SvelteKit static PWA for organizer + participant flows (create drive, share link/QR, offer/claim seat, view coverage), served from Cloud Storage + CDN.

### Phase 5: Privacy/Compliance Operations
**Rationale:** GDPR/minors obligations need an operational home once data exists.
**Delivers:** scheduled participant-data deletion anchored to drive date, PII log-masking middleware, data-disclosure notices, retention policy.
**Avoids:** GDPR/minors retention breach risk.

### Phase Ordering Rationale
- Foundation first because RLS, tenancy, the `stops` table, and typed config are the most dangerous to change later.
- The coverage slice (Phase 2) is intentionally a single vertical slice so the core value is demonstrable and testable end-to-end early.
- API and UI follow the proven domain (facade pattern keeps the HTTP layer thin); Phases 3 and 4 can run in parallel.
- Compliance ops (Phase 5) lands once real data flows, but its design constraints (PII minimization, organizer-only contact) are honored from Phase 1–2.

### Research Flags
Phases likely needing deeper research during planning:
- **Phase 1:** Drizzle has no first-party `set_config`/RLS hook — needs a concrete transaction-scoped tenant-context middleware pattern; PgBouncer deployment shape (sidecar vs separate service) needs a decision.
- **Phase 2:** lock the coverage definition edge cases before building (does a driver with no riders count? does a waitlisted rider count as covered? does a rider who hasn't joined count as "needs ride"?); rider self-registration ("I need a ride") UX is an open design question no competitor solves.
- **Phase 5:** confirm German/EU specifics (BDSG/TDDDG, retention windows) during planning.

Phases with standard patterns (lighter research):
- **Phase 3:** API Gateway + OpenAPI quotas are well-documented GCP patterns.
- **Phase 4:** SvelteKit static PWA is a standard, low-risk build.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | Cloud Run / Cloud SQL+RLS / API Gateway verified against official GCP docs + 2025–26 guidance; npm versions checked live |
| Features | HIGH | 8+ comparable products inspected; table stakes converge; coverage gap confirmed across all |
| Architecture | HIGH | Modular monolith, shared-schema+RLS, stops-table extensibility are established 2026 patterns |
| Pitfalls | HIGH | Race conditions, RLS `SET LOCAL`, signed URLs, GDPR Art. 8/age-16, connection pooling all multi-source verified |

**Overall confidence:** HIGH

### Gaps to Address
- **Drizzle ↔ RLS tenant context:** no first-party hook — implement raw `SET LOCAL` in Hono middleware before Drizzle queries; validate in Phase 1.
- **Token revocation granularity:** column-based revocation chosen for v1; revisit if external embedders must verify tokens independently (→ RS256 + jti).
- **Coverage definition edge cases:** must be locked before Phase 2 build (see research flags).
- **Connection-pooler operational shape:** decide PgBouncer deployment topology when connection pressure is real.
- **Frontend choice:** SvelteKit is MEDIUM confidence/deferrable; revisit at Phase 4 if component needs grow.

## Sources

### Primary (HIGH confidence)
- Official GCP docs — Cloud Run (min-instances, cpu-boost), Cloud SQL Postgres 16, API Gateway pricing/quotas, Cloud Armor
- PostgreSQL official docs — Row-Level Security, `SET` vs `SET LOCAL`, `SELECT FOR UPDATE`
- GDPR Art. 8 text; German age-16 threshold (BDSG) — multiple legal sources
- npm registry — Hono 4.x, Drizzle, Zod, jose (versions verified live)

### Secondary (MEDIUM confidence)
- 2025–2026 framework/ORM benchmarks (Hono vs alternatives; Drizzle cold-start/bundle)
- Modular-monolith / DDD facade-pattern literature; ClickHouse Engineering + AWS multi-tenancy guidance (shared-schema + RLS as 2026 default)
- Competitor feature inspection — TeamSnap, Caroster, GoKid, SignUpReady, SignUpGenius, GroupCarpool, Spond

### Tertiary (LOW confidence)
- SvelteKit bundle-size comparisons (2026) — informs frontend choice, validate at Phase 4

---
*Research completed: 2026-06-02*
*Ready for roadmap: yes*

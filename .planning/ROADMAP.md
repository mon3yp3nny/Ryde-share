# Roadmap: Ryde-share

## Overview

Ryde-share is built engine-first and vertically. The foundation locks in the correctness properties that are genuinely dangerous to retrofit — tenant isolation, the extensible stops schema, UUID external IDs, and connection pooling. The core vertical slice delivers the full "is everyone covered?" loop end-to-end so the product's primary value is demonstrable and testable early. Configurability knobs are layered in next, turning the fixed slice into an adaptable engine. A clean public API surface makes the engine embeddable. The standalone web UI is a thin client over the proven API, built last and intentionally late to keep it swappable.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Tenancy & Data Foundation** - Tenant isolation, extensible stops schema, UUID IDs, PgBouncer, and Cloud Run service with Postgres RLS wired up
- [ ] **Phase 2: Core Vertical Slice** - Create a drive, join via link/QR, offer and claim/assign seats, view coverage — the "is everyone covered?" loop working end-to-end with privacy baked in
- [ ] **Phase 3: Configurability** - All four per-drive knobs (meeting point mode, seat-selection mode, required rider info, capacity behavior) exposed and enforced
- [ ] **Phase 4: Public API & Compliance Ops** - OpenAPI contract, API-consumer credentials and rate limits, two-tier auth separation, scheduled data deletion, and PII log masking
- [ ] **Phase 5: Standalone Web UI** - SvelteKit static PWA for organizer and participant flows, served from Cloud Storage + CDN

## Phase Details

### Phase 1: Tenancy & Data Foundation
**Goal**: The deployment is multi-tenant from the first request — every record is tenant-scoped and enforced at the database layer, identifiers are non-enumerable, the stops schema is extensible without future migration, and the service can handle burst traffic without connection exhaustion.
**Mode:** mvp
**Depends on**: Nothing (first phase)
**Requirements**: TEN-01, TEN-02, TEN-03, DRIVE-03
**Success Criteria** (what must be TRUE):
  1. A request authenticated as tenant A cannot read or mutate any record belonging to tenant B, even if the application-layer tenant filter is removed
  2. All externally-visible record identifiers are UUIDs; no sequential integer IDs appear in any API response or shareable link
  3. A drive's start and destination are stored as two rows in a `stops` table with a `position` column; adding a third stop requires no schema migration
  4. The service handles 50 simultaneous connections without exhausting the Cloud SQL connection limit (PgBouncer transaction pooling + `min-instances=1` in place)
**Plans**: TBD

### Phase 2: Core Vertical Slice
**Goal**: The full "is everyone covered?" loop works end-to-end — an organizer creates a drive, participants join via shareable link or QR code without an account, drivers offer seats, riders claim or are assigned seats, and the coverage view shows whether all riders are covered or how many are still short, with participant contact data visible only to the organizer and a data-disclosure notice on the join screen.
**Mode:** mvp
**Depends on**: Phase 1
**Requirements**: DRIVE-01, DRIVE-02, DRIVE-04, PART-01, PART-02, PART-03, PART-04, PART-05, PART-06, PART-07, COV-01, COV-02, COV-03, PRIV-01, PRIV-02
**Success Criteria** (what must be TRUE):
  1. An organizer can create a drive with a start point, destination, and optional event information, and receive a shareable link and QR code
  2. A participant can join a drive via the shareable link without creating an account; the join screen presents a data-disclosure notice before any information is collected
  3. A driver can offer seats (declare available capacity); a rider can claim an open seat (when mode allows) or be assigned by the driver (when mode allows); two riders cannot be granted the same seat under concurrent access
  4. A participant can update or withdraw their own entry using a scoped token they received at join time, without an account
  5. The coverage view shows "covered" or "short by N" plus the list of specific riders who still need a seat; contact information is visible only to the organizer, never to other participants
**Plans**: TBD
**UI hint**: yes

### Phase 3: Configurability
**Goal**: All four per-drive configuration knobs are exposed and enforced — organizers can control meeting point visibility, seat-selection mode, required rider info, and capacity behavior, and the engine enforces the chosen behavior for every operation on that drive.
**Mode:** mvp
**Depends on**: Phase 2
**Requirements**: CFG-01, CFG-02, CFG-03, CFG-04
**Success Criteria** (what must be TRUE):
  1. An organizer can set a drive's meeting point to required, optional, or hidden; participants see or are asked for a meeting point accordingly
  2. An organizer can set a drive's seat-selection mode to rider self-pick, driver-assign, or both; the engine rejects operations that violate the configured mode
  3. An organizer can set a drive to require name only or name + contact; the join form enforces the configured fields and the coverage view reflects the right rider-info level
  4. An organizer can set a drive's capacity behavior to hard cap or waitlist; the engine honors the cap or routes additional riders to the waitlist; the coverage view reflects the waitlist distinction
**Plans**: TBD

### Phase 4: Public API & Compliance Ops
**Goal**: All core engine operations are available through a documented public API; consuming applications authenticate with API keys and are subject to per-consumer rate limits; anonymous participant routes are separated from authenticated consumer routes without exposing the raw Cloud Run URL; participant personal data is deleted on a schedule anchored to the drive date and PII is masked in application logs.
**Mode:** mvp
**Depends on**: Phase 3
**Requirements**: API-01, API-02, API-03, API-04, PRIV-03, PRIV-04
**Success Criteria** (what must be TRUE):
  1. Every core operation (create drive, offer/claim/assign seats, read coverage, configure a drive) is callable via a documented OpenAPI endpoint
  2. An API consumer authenticates with an API key; a consumer without a valid key receives a 401; a consumer without an account cannot impersonate another tenant
  3. A high-volume API consumer hits a rate-limit response after exceeding their quota; different consumers have independent quotas
  4. Shareable participant links do not contain or reveal the underlying Cloud Run service URL
  5. Participant personal data older than the configured retention window (anchored to drive date) is deleted by a scheduled job; PII fields do not appear in application log output
**Plans**: TBD

### Phase 5: Standalone Web UI
**Goal**: A lightweight static web UI lets organizers and participants use all core features without writing API calls — create a drive, share a link or QR code, offer and claim seats, and view coverage — served from Cloud Storage + CDN with no compute cost at rest.
**Mode:** mvp
**Depends on**: Phase 4
**Requirements**: (no new v1 requirements — thin client over the Phase 4 API)
**Success Criteria** (what must be TRUE):
  1. An organizer can create a drive, view its shareable link and QR code, and see the live coverage view using only the web UI (no API client required)
  2. A participant can open the shareable link on a mobile browser, see the data-disclosure notice, join the drive, and claim or be assigned a seat without installing anything
  3. The UI is served as static files from Cloud Storage + CDN; no Cloud Run instance is required to serve the UI assets
**Plans**: TBD
**UI hint**: yes

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4 → 5

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Tenancy & Data Foundation | 0/TBD | Not started | - |
| 2. Core Vertical Slice | 0/TBD | Not started | - |
| 3. Configurability | 0/TBD | Not started | - |
| 4. Public API & Compliance Ops | 0/TBD | Not started | - |
| 5. Standalone Web UI | 0/TBD | Not started | - |

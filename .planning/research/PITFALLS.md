# Pitfalls Research

**Domain:** Multi-tenant, API-first ride-coordination engine — anonymous participants, configurable per-drive behavior, GCP
**Researched:** 2026-06-02
**Confidence:** HIGH (critical pitfalls verified against official docs and multiple independent sources)

---

## Critical Pitfalls

### Pitfall 1: The Configuration Schema Explosion ("Everything Is a Knob")

**What goes wrong:**
You ship four knobs (meeting point visibility, seat selection mode, required rider info, capacity behavior) and the schema is a loose JSON blob or an undifferentiated `config` column. Within two milestones the number of supported keys has silently grown to 20, nobody knows which keys are required vs. optional, the validation is scattered across request handlers, and adding a new consumer (handball-manager) requires reading source code to understand what can be configured. Alternatively, you over-correct early: you build a typed, versioned, validated configuration DSL before you have even one real tenant — and it takes three times as long to ship the first drive.

**Why it happens:**
The product's stated identity is "configurability is the product," which creates psychological pressure to build the configuration system to be maximally general from day one. Early JSONB columns feel cheap and future-proof — until the schema drifts and becomes undocumented.

**How to avoid:**
- Define a strict typed schema for the four v1 knobs — not a JSON blob. Use a Postgres JSONB column **with a JSON Schema constraint** (CHECK with `jsonb_matches_schema`) or a flat set of nullable columns for the four knobs. A flat column approach is actually easier to validate and query than an open JSONB blob.
- Reserve JSONB only for the fields that are genuinely unknown today (e.g., future multi-stop waypoints). For the four known knobs, use explicit typed columns (`seat_selection_mode ENUM`, `capacity_behavior ENUM`, etc.).
- Apply the "rule of three": generalize a configuration axis only after three real tenants have a different value for it. Until then, hard-code the edge case.
- Separate config *validation* (what values are legal) from config *defaults* (what a new drive gets). Encode defaults as constants, not magic nulls.

**Warning signs:**
- A `config` column that is a JSONB blob without a documented schema
- Application code doing `config.get('some_key', None)` without a corresponding validation step at write time
- PR descriptions that say "added X to drive config" without updating a schema definition file

**Phase to address:**
Data model phase (Phase 1/foundation). The schema of `Drive.config` must be locked down before the first API endpoint is built, because changing it later requires a migration that touches every existing drive.

---

### Pitfall 2: Tenant Scoping Miss — The Cross-Tenant Data Leak

**What goes wrong:**
One raw query, one ORM eager-load, or one admin endpoint is written without a `tenant_id` filter. A drive belonging to tenant A is returned in tenant B's coverage view. This is not hypothetical — it is the most common multi-tenancy bug in shared-schema databases. In a ride-coordination engine it exposes real people's names, contact information, and travel plans.

**Why it happens:**
Application-layer tenant scoping by convention fails because it relies on every developer remembering to add the filter on every query. One `findById(driveId)` call — without also checking `tenantId` — is the entire attack surface.

**How to avoid:**
- Enable PostgreSQL Row-Level Security (RLS) on all tables that carry `tenant_id`. Set the application role's `SET LOCAL app.current_tenant_id = $1` on every connection before executing queries. Do **not** use `SET` (persists across session/pool) — only `SET LOCAL` (scoped to the current transaction).
- Use `ALTER TABLE drives FORCE ROW LEVEL SECURITY` — even the table owner is subject to the policy.
- Never grant the application database role `BYPASSRLS`. Use a separate migration-only role that has it.
- All queries through an ORM must pass through a single repository class that injects `tenant_id` automatically. Raw SQL queries are gated behind a code-review checklist item: "does this query include tenant_id in the WHERE clause or rely on RLS?"
- Write a specific integration test that logs in as tenant A, creates data, then queries as tenant B and asserts 0 rows returned.

**Warning signs:**
- A query that looks up a resource by `id` alone (no `tenant_id` join)
- Any endpoint whose URL contains only a drive/seat ID without a tenant discriminator
- Connection pooling (PgBouncer, Cloud SQL connector) without verifying `SET LOCAL` happens inside a transaction, not outside it

**Phase to address:**
Foundation/data model phase. RLS policies must be in place before any data is inserted. Retrofitting RLS onto a populated database requires a migration to backfill `tenant_id` and a period of dual-enforcement.

---

### Pitfall 3: Seat Claim Race Condition — Two Riders Claim the Last Seat

**What goes wrong:**
Two riders open the same drive link simultaneously. Both read `available_seats = 1`. Both POST to `/drives/{id}/seats/claim`. Both get 200 OK. The driver now has two riders assigned to one seat, which breaks the coverage model — the entire value proposition of the product.

A second variant: driver assigns a seat while a rider is mid-flow claiming it. The driver's assignment wins (or doesn't), leaving the system in an inconsistent state.

**Why it happens:**
A non-atomic read-then-write pattern: `SELECT capacity WHERE drive_id = $1` followed by `UPDATE seats SET rider_id = $2 WHERE seat_id = $3 AND rider_id IS NULL`. Without explicit locking, two concurrent transactions pass the check before either commits the update.

**How to avoid:**
- Use `SELECT ... FOR UPDATE` on the seat row (or drive row) at the start of the claim transaction. This is the correct tool for the seat-booking concurrency problem — it is a high-contention, low-conflict scenario where pessimistic locking is cheaper than optimistic retry loops.
- Model the atomic operation as a single UPDATE with a WHERE predicate: `UPDATE seats SET rider_id = $riderId WHERE seat_id = $seatId AND rider_id IS NULL RETURNING *`. If RETURNING returns 0 rows, the seat was already taken — return 409 Conflict to the caller.
- For the driver-assign vs. rider-claim conflict, use an ENUM state machine on the seat: `(open → reserved → confirmed)`. Driver assignment and rider self-pick both transition `open → confirmed` via the same atomic UPDATE. The loser gets a 409.
- Never check seat availability in application code and then write in a separate query. The check and the write must be one atomic database operation.

**Warning signs:**
- Seat availability checked in a service method, then a separate `save()` call
- Duplicate seat assignments appearing in testing under load (even low concurrency)
- Capacity count derived from `COUNT(*)` in application code rather than enforced at the database level with a constraint

**Phase to address:**
Core booking mechanics (Phase 2). The atomic seat-claim pattern must be established before the first rider-facing endpoint ships. Adding locking retroactively to a live system under load is dangerous.

---

### Pitfall 4: Anonymous Link Abuse — Seat Stuffing and Drive Enumeration

**What goes wrong:**
The drive link is a bare UUID or short ID (e.g., `/drives/abc123`). A bad actor (or a script-kiddie from a rival club) can:
1. **Enumerate drives** by iterating IDs and harvesting participant names/contact info
2. **Stuff seats** by claiming all available seats under fake names, locking out legitimate riders
3. **Spam claim/unclaim** to degrade the coverage view ("is everyone covered?" oscillates)

The damage is real because the flagship use case involves children's names and parent contact info — GDPR-reportable exposure.

**Why it happens:**
Anonymous participation is a deliberate design choice (zero friction), but without defenses, the same openness that makes joining easy makes abusing easy. UUIDs alone are not a security mechanism — they are just unguessable, not unforgeable once shared.

**How to avoid:**
- Use **cryptographically signed capability URLs**. The shareable link is not just a drive ID — it is a signed token: `HMAC-SHA256(drive_id + tenant_id + expiry_timestamp, secret)`. The server verifies the signature on every request. Expired or tampered links return 404 (not 401, to avoid enumeration).
- Set link expiry appropriate to the use case: a drive for Saturday's game has a link valid through Saturday + 1 hour. After expiry the link is dead; the organizer can generate a new one.
- Rate-limit the `/drives/{token}/join` and `/drives/{token}/seats/claim` endpoints by IP **and** by token. GCP Cloud Armor or API Gateway can enforce this at the edge before requests hit Cloud Run.
- For the seat-stuffing vector specifically: require the rider to provide a name (which is already a configurable field) and impose a per-IP and per-token rate limit on seat claims (e.g., maximum 2 claims per token per hour from the same IP).
- Do not return drive contents (participant list, contact info) on the drive-join endpoint. Return only what is needed to render the join form. Participant details should require a separate authenticated request by the organizer's client.

**Warning signs:**
- Drive links that are incrementing integers or sequential short IDs
- No rate limit on the join/claim endpoint
- A single API response that returns both drive metadata and all participant PII in one call
- No link expiry in the data model (no `expires_at` column on drive invitations)

**Phase to address:**
Anonymous participation flow (Phase 2). Signed tokens and rate limits must be in place before the first public link is generated. This cannot be bolted on after the URL structure is public — changing URL format breaks all existing links.

---

### Pitfall 5: Privacy/PII Exposure for Children — GDPR Article 8 and German BDSG

**What goes wrong:**
The flagship use case (youth sports clubs, Hamburg handball teams) means participants are frequently minors — players aged 10-16. The organizer collects a player's name and optionally a contact number. This is personal data under GDPR. Specific failure modes:

1. Storing contact info indefinitely after the drive ends
2. No legal basis for processing a minor's contact info (Germany: GDPR Art. 8 sets the age of consent for information society services at **16** — the highest in the EU)
3. Exposing participant lists (including minors' names) to all drive participants, not just the organizer
4. No privacy notice / data processing disclosure at the point of data entry

**Why it happens:**
Sports-club apps are perceived as low-stakes "internal tools" and GDPR compliance is deferred. The anonymous-first design creates a false sense that no PII is being collected — but a name + phone number is absolutely personal data.

**How to avoid:**
- **Data minimization is the primary defense.** Default required rider info to "name only." Contact info is opt-in, configurable per drive, and should default off. Never make contact info visible to other riders — only to the drive organizer/driver who needs to reach them.
- Implement **automatic deletion**: all participant data (names, contact info) is hard-deleted N days after the drive's scheduled date. 7 days is defensible; 30 days maximum. Store a `drive_date` and run a scheduled deletion job (Cloud Scheduler → Cloud Run).
- Show a **data disclosure notice** at the join screen: "Your name [and phone number] will be shared with the driver and organizer for this trip and deleted after [date]." This is the minimum transparency requirement under GDPR Art. 13.
- Do not display contact info in the coverage API response. Contact info is a separate, organizer-only endpoint.
- Because Germany applies Art. 8's 16-year threshold: the service itself should not be an "information society service offered directly to children" — it is offered to clubs/organizers (the API consumer). Riders are participants, not account holders. This framing keeps the service on the side of "offered to organizations" rather than "offered directly to children," which limits direct Art. 8 exposure. Document this design decision explicitly.
- When the "name + contact" option is enabled, include a checkbox: "I consent to my contact details being shared with the driver for this trip." Log the consent timestamp and IP. Delete the consent record with the participant record.

**Warning signs:**
- Participant data still in the database 30 days after a completed drive
- Contact info visible in the coverage view API response (not just organizer-facing endpoints)
- No data disclosure text in the join flow
- No `drive_date` or equivalent field to anchor the deletion schedule

**Phase to address:**
Anonymous participation flow (Phase 2) for the disclosure notice and data minimization defaults. Scheduled deletion job (Phase 3/ops). Organizer-only contact info endpoint (Phase 2).

---

### Pitfall 6: Cloud Run Cold Starts With Cloud SQL Connection Storms

**What goes wrong:**
Cloud Run scales from 0 to N instances on a traffic burst (e.g., a club shares a drive link and 40 parents click it within 30 seconds). Each new Cloud Run instance establishes its own connection(s) to Cloud SQL. Cloud SQL has a hard default of 100 connections on the smallest tier. 40 instances × 2 connections each = 80 connections consumed in seconds. At the next burst with more tenants, the database hits `FATAL: remaining connection slots are reserved for non-replication superuser connections` and drops requests entirely.

**Why it happens:**
Cloud Run's horizontal scaling is a strength for compute, but each instance is connection-pool-isolated. Without a connection pooler between Cloud Run and Cloud SQL, the effective connection limit is `max_instances × pool_size_per_instance`, not a managed pool. This is a well-documented Cloud Run + Cloud SQL failure mode.

**How to avoid:**
- Deploy **PgBouncer** as a sidecar or as a separate Cloud Run service in transaction pooling mode. Transaction pooling is correct here: most requests are short-lived API calls. Session pooling is wrong for Cloud Run because sessions don't persist.
- Alternatively, use **Cloud SQL Managed Connection Pooling** (requires Enterprise Plus edition — evaluate cost vs. complexity tradeoff at project start).
- Set `max_instances` on Cloud Run to a value you have explicitly validated against your Cloud SQL connection limit: `max_instances × connections_per_instance_pool_size ≤ (Cloud SQL max_connections × 0.8)`.
- Set `--min-instances=1` on the API service to eliminate cold-start latency for the first request. At ~$3-10/month in idle memory costs, this is almost always worth it for a user-facing service.
- Configure Cloud Run with `--cpu-boost` to accelerate startup for cold instances.
- On Cloud SQL, increase `max_connections` deliberately (not by default) and set `connection_timeout` on the application pool so a failed connection attempt surfaces as a 503 rather than hanging forever.

**Warning signs:**
- No PgBouncer or equivalent in the architecture diagram
- `max_instances` not set (defaults to 100 in Cloud Run, which will exhaust a default Cloud SQL instance)
- Error logs showing `too many connections` or `connection refused` on Cloud SQL
- P95 latency spikes of 2-10 seconds correlating with scale-out events

**Phase to address:**
Infrastructure setup (Phase 1). Connection pooling is a foundational infrastructure decision — retrofitting PgBouncer into a running system requires redeployment of every service and connection string changes everywhere.

---

### Pitfall 7: Building the Embedding API Before There Is a Real Consumer

**What goes wrong:**
You build a fully generalized, versioned, SDK-compatible embedding API — webhook hooks, event streams, customizable field mappings, a plugin interface — before handball-manager (or any other consumer) has a single concrete integration requirement. The API surface is large, the abstraction is wrong for the actual use case when it eventually arrives, and you have to maintain backward compatibility for an API nobody is using.

The opposite failure: you build "internal-only" APIs with no thought to embedding, then when handball-manager finally wants to integrate, every endpoint requires an authenticated session cookie, all IDs are internal sequential integers, and the data model exposes implementation details.

**Why it happens:**
"API-first and embeddable" is a stated architectural constraint, which is correct. But "embeddable" is interpreted as "build a full SDK now" rather than "design the API so an SDK could be built later."

**How to avoid:**
- Ship a clean, well-documented REST API. That is the embedding surface. Do not build an SDK, webhooks, or event streams until handball-manager (or another real consumer) has a confirmed integration need.
- API-first means: every operation is accessible via HTTP with a stable path, documented request/response schema, and an authentication mechanism. It does not mean: a plugin system, a webhook bus, or a GraphQL federation layer.
- Use stable, external-facing resource identifiers (UUIDs, not sequential integers). This is the one irreversible design choice — once IDs are in external consumers' databases, changing them is breaking.
- Version the API from day one (`/v1/...`). But ship only v1. Don't design v2 until v1 has users.
- Resist adding `embed_options`, `callback_url`, or `custom_field_mapping` parameters until there is a concrete consumer with a concrete requirement.

**Warning signs:**
- API design meetings that reference "what if a consumer wants to..." without a named, real consumer making the request
- A `webhooks` or `events` table in the schema before the first external consumer is connected
- More time spent on API generalization than on the core coverage view feature

**Phase to address:**
API design phase (Phase 1). The "clean REST, UUIDs, versioned" discipline is phase 1. The "no premature generalization" discipline is a continuous guardrail — reference it explicitly in every phase's scope definition.

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| JSONB blob for all drive config | Fast to add new knobs | Schema drift, no validation, unqueryable, breaks refactors | Never — use typed columns for known knobs, JSONB only for genuinely open-ended future extension |
| Application-layer tenant filter (WHERE tenant_id = ?) without RLS | Simpler setup | One missed query leaks cross-tenant data; impossible to audit | Never for tables with PII or business-critical data |
| Sequential integer IDs | Simple debugging | External IDs leak row counts, enable enumeration attacks, cannot be changed once embedded | Never for externally-exposed resource IDs |
| No connection pooler (direct Cloud SQL from Cloud Run) | One fewer component | Connection exhaustion at scale; 503s under burst traffic | Only acceptable for local dev; never in staging or prod |
| Storing participant data indefinitely | No need to build deletion job | GDPR violation; data breach risk accumulates over time | Never — schedule deletion from the start |
| Skip link signing (bare UUID links) | Simpler URLs | Drive enumeration, seat stuffing, PII scraping | Never for public-facing drives; acceptable only for internal organizer-only views |
| Omit data disclosure on join screen | Faster to ship | GDPR Art. 13 violation; regulatory risk | Never — the notice is a 3-line UI addition |

---

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| Cloud SQL via Cloud Run | Direct connection without pooler; no `SET LOCAL` for RLS context | PgBouncer in transaction mode; `SET LOCAL app.current_tenant_id = $1` inside the transaction, not outside |
| Cloud Run + Cloud SQL Auth Proxy | Treating the proxy as a connection pooler (it is not; it is a TLS tunnel) | Use proxy for auth/TLS, PgBouncer for pooling — they are complementary, not alternatives |
| QR code generation | Encoding the raw drive link (which may expire) into a static QR | Encode a link-generation endpoint that issues a fresh signed token on each scan, or regenerate QR on link refresh |
| Rate limiting at the application layer | Per-instance in-memory counters (each Cloud Run instance has its own counter; limits don't aggregate) | Use Cloud Armor (GCP WAF) or API Gateway for edge-level rate limiting; use Redis/Memorystore for distributed in-app counters |
| GDPR deletion | Soft-delete (`deleted_at`) that still returns data in queries | Hard DELETE from participant tables after retention period; soft-delete is insufficient for GDPR erasure |

---

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| Coverage view computed on every request by full table scan | Slow response as drives grow; `SELECT COUNT(*)` on large participant tables | Pre-compute and store `covered_count` / `total_needed` as denormalized columns, updated atomically with seat state changes | ~100 concurrent drives per tenant, or any drive with 50+ participants |
| No pagination on participant lists | Organizer endpoint returns all participants in one response | Cursor-based pagination on all list endpoints from day one | Drives with >50 participants; multiple concurrent organizer views |
| Seat state computed by joining 4 tables in real-time | P95 latency grows with drive size | Maintain a `seat_status` denormalized field on the seat row; update via trigger or in the same transaction as state changes | ~20 concurrent seat-claim attempts on one drive |
| Cloud Run max_instances not capped | Unexpected cost explosion during a viral drive link; Cloud SQL connection exhaustion | Set `max_instances` explicitly; configure budget alerts in GCP | Any burst > (Cloud SQL max_connections / per_instance_pool_size) instances |
| No index on `tenant_id` columns | Full table scan on every tenanted query even with RLS | Index `(tenant_id, id)` and `(tenant_id, drive_id)` composite columns on every join table | ~10 tenants with ~50 drives each |

---

## Security Mistakes

| Mistake | Risk | Prevention |
|---------|------|------------|
| Drive links are bare resource IDs (UUID or integer) | Enumeration attack; all drives in a tenant are discoverable by iterating IDs | Capability URLs: signed HMAC token in the link path, not just a resource ID |
| Signed link secret is a per-drive value stored in the database | Compromise of one drive row reveals all its links | Use a tenant-scoped or global server secret (environment variable); the token proves "this server issued this link for this drive at this time" |
| RLS bypassed by superuser app role | Cross-tenant data exposure via any SQL injection | Application role must not have BYPASSRLS; enforce in infrastructure-as-code |
| Participant PII (name, phone) in API logs | Log aggregation (Cloud Logging) becomes a PII store; GDPR audit target | Mask PII fields in request/response logging middleware; log only resource IDs |
| No CORS policy on the public API | Any website can call the API using a visitor's session context | Explicit CORS allowlist per environment; reject credentialed requests from unlisted origins |
| Drive invitation link re-shared beyond intended audience | Unauthorized participants join a private drive | Implement per-link claim limits and per-link participant caps; allow organizers to revoke/rotate links |

---

## UX Pitfalls

| Pitfall | User Impact | Better Approach |
|---------|-------------|-----------------|
| Coverage view shows raw seat counts but not the answer to "is everyone covered?" | Organizer must do mental arithmetic; the core value is not delivered | Primary UI signal is a single boolean + delta: "Covered" / "Short by 3" — counts are secondary detail |
| Seat claim succeeds but coverage view is stale (eventual consistency display) | Rider claims a seat, refreshes, sees it's still unclaimed — claims again | Serve coverage view with strong read consistency (read from primary, not replica); the volume is low enough that this is fine |
| Expired drive link returns 401 Unauthorized | User doesn't know why they can't join; contacts organizer confused | Expired links return a custom 410 Gone response with a user-readable message: "This link has expired. Ask the organizer for a new one." |
| No confirmation after seat claim | Rider doesn't know if their claim registered | POST to claim returns the full updated drive state, not just 204 No Content |
| "Name required" validation fires after form submit | Minor friction; feels broken on mobile | Client-side validation with inline error before submit |

---

## "Looks Done But Isn't" Checklist

- [ ] **Seat claiming:** The claim endpoint uses `SELECT FOR UPDATE` or equivalent atomic write — verify there is no read-then-write gap in the implementation
- [ ] **Tenant scoping:** A test exists that proves tenant B cannot read tenant A's drives, seats, or participants — not just "we have WHERE tenant_id" in the code
- [ ] **Signed links:** The signature verification path is tested with tampered tokens, expired tokens, and tokens from a different drive — all must return 404
- [ ] **PII deletion:** A scheduled job exists and is tested; verify participant records are absent from the database after the retention period, not merely soft-deleted
- [ ] **Data disclosure:** The join screen renders the disclosure notice before the name/contact form — verify this renders even when JavaScript is slow (server-rendered or inline)
- [ ] **Connection pooling:** Load test with N concurrent Cloud Run instances (simulate auto-scale) and confirm Cloud SQL connection count stays within limits
- [ ] **Rate limiting:** The seat-claim endpoint returns 429 after the configured limit from a single IP — verify this is enforced at the edge, not just per-instance
- [ ] **RLS enforcement:** Verify with `SET ROLE app_user; SELECT * FROM drives;` directly in psql — zero rows should be returned without the tenant context variable set

---

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| Cross-tenant data leak discovered in production | HIGH | Immediate: rotate all API keys, audit Cloud Logging for cross-tenant query patterns, notify affected tenants per GDPR Art. 33 (72-hour deadline). Then: add RLS, audit all queries, regression test suite. |
| Seat double-booking discovered | MEDIUM | Run deduplication query to identify affected drives; notify organizers; add `SELECT FOR UPDATE` to claim path; add unique constraint on (drive_id, rider_id) as second-line defense |
| GDPR erasure request for participant data | LOW (if deletion job exists) / HIGH (if not) | If deletion job exists: verify the job ran for the affected drive, confirm records are gone. If not: emergency manual deletion, data audit, add deletion job immediately, document as incident |
| Cloud SQL connection exhaustion (prod 503s) | MEDIUM | Immediate: scale down Cloud Run max_instances, restart stuck connections. Then: deploy PgBouncer, adjust pool sizes, set explicit max_instances cap |
| Signed link secret leaked | HIGH | Rotate the signing secret; all existing links immediately invalidated (users must request new links from organizers). This is disruptive but contained |

---

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| Config schema explosion | Phase 1 (data model) | Schema review: all four config knobs are typed columns or a schema-validated struct, not a free-form blob |
| Tenant data leakage | Phase 1 (data model + infrastructure) | Cross-tenant integration test returns 0 rows |
| Seat race condition | Phase 2 (core booking mechanics) | Load test: 50 concurrent claims on 1 seat; exactly 1 succeeds |
| Anonymous link abuse | Phase 2 (participation flow) | Link with tampered signature returns 404; expired link returns 410; rate limit test passes |
| GDPR/minors PII | Phase 2 (participation flow) + Phase 3 (ops) | Disclosure renders on join screen; deletion job removes records after retention window |
| Cloud Run cold start + connection storm | Phase 1 (infrastructure) | Load test: scale-out event does not exceed Cloud SQL connection limit |
| Premature embedding API | All phases (continuous) | No webhooks/event-stream/SDK code exists until handball-manager has a named integration requirement |

---

## Sources

- PostgreSQL RLS: [Multi-tenant data isolation with PostgreSQL RLS — AWS](https://aws.amazon.com/blogs/database/multi-tenant-data-isolation-with-postgresql-row-level-security/) | [Mastering PostgreSQL RLS for Multi-Tenancy — Rico Fritzsche](https://ricofritzsche.me/mastering-postgresql-row-level-security-rls-for-rock-solid-multi-tenancy/)
- Seat booking concurrency: [Database Locking in Airline Seat Booking — Developer's Coffee](https://www.developerscoffee.com/blog/understanding-database-locking-in-an-airline-seat-booking-system/) | [PostgreSQL Concurrency Control — Nemanja Tanasković](https://nemanjatanaskovic.com/blog/postgresql-concurrency-control-isolation-levels-locks)
- Advisory locks: [Orchestrating Distributed Tasks with PostgreSQL Advisory Locks — Leapcell](https://leapcell.io/blog/orchestrating-distributed-tasks-with-postgresql-advisory-locks)
- GDPR Art. 8 / Germany age 16: [Art. 8 GDPR — gdpr-info.eu](https://gdpr-info.eu/art-8-gdpr/) | [GDPR Age of Digital Consent — euconsent.eu](https://euconsent.eu/digital-age-of-digital-consent-under-gdpr)
- GDPR children sports: [GDPR for Small Clubs and Societies — DataGuard](https://www.dataguard.com/blog/gdpr-for-small-clubs-and-societies) | [You can't win anything with kids' data — Clifford Chance](https://www.cliffordchance.com/insights/resources/blogs/talking-tech/en/articles/2022/04/you-can-t-win-anything-with-kids-minor-s-data-in-sport.html)
- Cloud Run cold starts + connection pooling: [Cloud Run + PostgreSQL — hoop.dev](https://hoop.dev/blog/the-simplest-way-to-make-cloud-run-postgresql-work-like-it-should/) | [Cloud SQL Managed Connection Pooling — Google Docs](https://docs.cloud.google.com/sql/docs/postgres/managed-connection-pooling) | [Cloud Run min-instances — Cloud Guardian](https://cloudguard.dev/blog/cloud-run-min-instances)
- Noisy neighbor: [Noisy Neighbor Problem in Multi-Tenant Systems — Neon](https://neon.com/blog/noisy-neighbor-multitenant) | [Noisy Neighbor Antipattern — Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/antipatterns/noisy-neighbor/noisy-neighbor)
- Over-engineering configurability: [Avoiding Over-Engineering — Rico Fritzsche](https://ricofritzsche.me/avoiding-over-engineering-focus-on-real-problems-in-software-development/) | [Avoiding Premature Abstractions — Better Programming](https://betterprogramming.pub/avoiding-premature-software-abstractions-8ba2e990930a)
- Signed/capability URLs: [Token-Based URLs — VdoCipher](https://www.vdocipher.com/blog/token-based-urls/) | [Short-Lived Credentials Limitations — token.security](https://www.token.security/blog/short-lived-credentials-token-abuse)
- German data protection (BDSG, TDDDG): [German Data Privacy Laws — Didomi](https://www.didomi.io/blog/germany-data-privacy-protection-laws-everything-you-need-to-know) | [DLA Piper Data Protection Germany](https://www.dlapiperdataprotection.com/index.html?t=law&c=DE)

---
*Pitfalls research for: multi-tenant ride-coordination engine (ryde-share)*
*Researched: 2026-06-02*

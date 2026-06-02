# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-06-02)

**Core value:** A reliable, instantly-readable answer to "is everyone covered?" for a group trip — backed by an engine flexible enough that almost every behavior adapts to each organization's context.
**Current focus:** Phase 1 — Tenancy & Data Foundation

## Current Position

Phase: 1 of 5 (Tenancy & Data Foundation)
Plan: 0 of TBD in current phase
Status: Ready to plan
Last activity: 2026-06-02 — Roadmap created; 28 v1 requirements mapped to 5 phases

Progress: [░░░░░░░░░░] 0%

## Performance Metrics

**Velocity:**
- Total plans completed: 0
- Average duration: -
- Total execution time: 0 hours

**By Phase:**

| Phase | Plans | Total | Avg/Plan |
|-------|-------|-------|----------|
| - | - | - | - |

**Recent Trend:**
- Last 5 plans: -
- Trend: -

*Updated after each plan completion*

## Accumulated Context

### Decisions

Decisions are logged in PROJECT.md Key Decisions table.
Recent decisions affecting current work:

- Foundation: Use Postgres RLS with `SET LOCAL` (not `SET`) inside transactions; never SET without LOCAL on pooled connections
- Foundation: PgBouncer transaction pooling + `min-instances=1` are MVP-critical, not deferrable
- Foundation: Typed columns for the 4 v1 config knobs (not JSONB) to avoid config drift
- Foundation: `stops` table with `position` column from day 1; V1 inserts 2 rows (origin + destination)
- Foundation: HS256 HMAC JWTs via `jose` for anonymous participant tokens; revoke via column check, no Redis/jti denylist for v1
- Architecture: Modular monolith with `RideCoordinationFacade` as sole public contract; HTTP layer added after domain is proven

### Pending Todos

None yet.

### Blockers/Concerns

- Phase 2 (pre-planning): Coverage definition edge cases must be locked before build — does a driver with no riders count? Does a waitlisted rider count as covered? Does a rider who hasn't joined count as "needs ride"?
- Phase 1 (pre-planning): Drizzle has no first-party `set_config`/RLS hook — needs concrete transaction-scoped tenant-context middleware pattern; PgBouncer deployment shape (sidecar vs separate Cloud Run service) needs a decision

## Deferred Items

| Category | Item | Status | Deferred At |
|----------|------|--------|-------------|
| *(none)* | | | |

## Session Continuity

Last session: 2026-06-02
Stopped at: Roadmap created, REQUIREMENTS.md traceability updated, STATE.md initialized — ready to plan Phase 1
Resume file: None

# Ryde-share

## What This Is

Ryde-share is a multi-tenant, API-first, configurable **ride-coordination engine**. Organizations create "drives" — a group trip from a start point to a destination — where members offer seats and others claim or are assigned them, with a clear at-a-glance view of whether everyone who needs a ride is **covered**. The flagship use case is sports clubs coordinating rides to away games (players, parents, trainers offer to drive), but the engine is deliberately domain-agnostic: any kind of organization can use it. It works fully standalone via shareable link/QR drives with anonymous participants, and exposes a clean API so larger applications can embed it.

## Core Value

A reliable, instantly-readable answer to **"is everyone covered?"** for a group trip — backed by an engine flexible enough that almost every behavior (meeting point, seat selection, capacity rules) adapts to each organization's context.

## Requirements

### Validated

<!-- Shipped and confirmed valuable. -->

(None yet — ship to validate)

### Active

<!-- Current scope. Building toward these. MVP. -->

- [ ] An organizer can create a **drive** with a start point and a single destination (one start → one destination per car/group)
- [ ] A drive carries **optional event information** (e.g. the away game it relates to)
- [ ] People can **join a drive via a shareable link and/or QR code**, anonymously (no lasting account)
- [ ] A driver can **offer seats** (declare available capacity) on a drive
- [ ] A rider can **claim an open seat**, and/or a **driver can assign seats** — which mode applies is configurable per drive
- [ ] A clear **coverage view** shows whether everyone who needs a ride has one (covered / short-by-N)
- [ ] **Configurable per drive:** meeting point (required / optional / hidden)
- [ ] **Configurable per drive:** seat selection mode (rider self-picks / driver assigns / both allowed)
- [ ] **Configurable per drive:** required rider info (name only / name + contact)
- [ ] **Configurable per drive:** capacity behavior (hard cap / allow waitlist)
- [ ] A clean **public API** for all core operations, so consuming applications can drive the engine
- [ ] **Multi-tenant:** one deployment serves many independent organizations, isolated from each other
- [ ] **API consumers authenticate** (bring their own identity) and are subject to **rate limits**; individuals do not need accounts

### Out of Scope

<!-- Explicit boundaries. Schema may anticipate these, but they are not built in v1. -->

- **Lasting end-user accounts** — individuals (organizers, drivers, riders) stay anonymous/ephemeral; only high-volume API consumers get accounts/keys. Keeps friction near zero.
- **Multiple pickup / drop-off stops** — v1 is single start → single destination; data model should anticipate multi-stop, but it is not built yet.
- **Return / round trip** — v1 tracks one leg only.
- **Notifications / push / reminders** — v1 is silent, pull-only (users refresh to see current coverage).
- **Custom / extra configurable fields per org** — v1 ships a fixed set of knobs (above); arbitrary custom fields are deferred.
- **Cost / fuel splitting** — deferred.
- **Recurring drives** — deferred.
- **Active handball-manager integration work** — the integration is real but deferred; build API-first so it can plug in later, but no integration work in the MVP.
- **Native mobile app** — platform/stack to be chosen as best technical fit on GCP; not committing to native v1.

## Context

- **The spark:** A concrete need — coordinating who drives to sports-club away games and seeing whether the whole squad is covered — motivated building a general, reusable app.
- **Standalone + embeddable:** Must be fully usable in isolation, but also reachable by bigger applications via API. The user's adjacent `handball-manager` project (`~/Desktop/Development/handball-manager`) is the intended *future* primary API consumer, though ride-sharing is not yet on its agenda.
- **Modular by design:** Ride-sharing is one self-contained module. The architecture should let other modules / use cases join later, or let this single module run on its own.
- **Configurability is the product.** The user repeatedly emphasized that behavior must adapt to context (meeting point required-or-not, self-pick vs. driver-assign, single-stop now / multi-stop later). The design tenet is "flexible engine, simple defaults now, extensible without rewrite."
- **Identity:** Anonymous/ephemeral for individuals; authenticated for consuming applications (relevant mainly for rate limits and for apps that bring their own organizers).
- **Domain-agnostic:** Sports clubs are the flagship, but the engine should not hard-code sports concepts — any organization should fit.

## Constraints

- **Infrastructure**: Runs on **Google Cloud Platform (GCP)** — the specific stack should be chosen as the best technical fit for an API-first, multi-tenant service on GCP (to be settled in research).
- **Architecture**: **API-first and modular** — every core capability must be available via API; the ride-sharing module must be self-contained and embeddable.
- **Tenancy**: **Multi-tenant** from the start — many independent organizations on one deployment, isolated.
- **Identity / friction**: Individuals must be able to participate **anonymously** (link/QR); no lasting end-user accounts. Accounts/keys exist only for high-volume API consumers, with **rate limiting**.
- **Extensibility**: Data model must anticipate multi-stop trips, return legs, and additional config knobs **without requiring a rewrite**, even though v1 doesn't build them.

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| API-first, modular engine (not a monolithic app) | Must run standalone *and* be embeddable by larger apps; reusable across use cases | — Pending |
| Domain-agnostic core, sports clubs as flagship use case | The same need exists for any organization; avoid hard-coding sports concepts | — Pending |
| Anonymous/ephemeral individuals; accounts only for API consumers | Keep participation friction near zero; identity/rate-limits matter only at the consuming-app layer | — Pending |
| Multi-tenant from day one | One deployment serves many organizations, isolated | — Pending |
| Lean MVP: single start → destination per car/group | Ship the core "is everyone covered?" value fast; defer multi-stop | — Pending |
| v1 configurable knobs limited to 4 (meeting point, seat selection mode, required rider info, capacity behavior) | These map to real cross-org variation and directly affect coverage; keep momentum, design schema for more later | — Pending |
| Silent, pull-only for MVP (no notifications) | Coverage is checked on demand; notifications add scope without blocking core value | — Pending |
| Deploy on GCP; specific stack chosen in research | User wants the best technical solution rather than a pre-committed stack | — Pending |
| handball-manager integration deferred but designed-for | It's the real future consumer, but not yet on its agenda; keep the API door open | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd:complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-06-02 after initialization*

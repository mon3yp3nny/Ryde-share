# Requirements: Ryde-share

**Defined:** 2026-06-02
**Core Value:** A reliable, instantly-readable answer to "is everyone covered?" for a group trip — backed by an engine flexible enough that almost every behavior adapts to each organization's context.

## v1 Requirements

Requirements for the initial release. Each maps to a roadmap phase.

### Tenancy & Foundation

- [ ] **TEN-01**: One deployment serves many independent organizations, with data isolated per organization
- [ ] **TEN-02**: Every organization-scoped record is enforced as tenant-isolated at the database layer (not by application convention alone)
- [ ] **TEN-03**: All externally-exposed identifiers are non-enumerable (UUIDs, never sequential IDs)

### Drives

- [ ] **DRIVE-01**: An organizer can create a drive with a single start point and a single destination
- [ ] **DRIVE-02**: A drive can carry optional event information (e.g. the away game it relates to)
- [ ] **DRIVE-03**: A drive's start point and destination are stored as ordered stops, so additional stops can be added later without a data migration
- [ ] **DRIVE-04**: An organizer can view a drive's current state at any time (pull-only; no notifications)

### Participation (anonymous, link/QR)

- [ ] **PART-01**: A drive is joinable via a shareable link and a QR code, without creating an account
- [ ] **PART-02**: A participant joining via link/QR identifies themselves with the info the drive requires (name only, or name + contact)
- [ ] **PART-03**: A driver can offer seats on a drive by declaring available capacity
- [ ] **PART-04**: A rider can claim an open seat themselves when the drive's mode allows self-pick
- [ ] **PART-05**: A driver can assign a rider to a seat when the drive's mode allows driver-assign
- [ ] **PART-06**: A participant can change or withdraw their own entry (unclaim a seat) without an account, via a scoped token
- [ ] **PART-07**: Seat allocation is atomic — two participants cannot be granted the same seat under concurrent access

### Coverage

- [ ] **COV-01**: A drive shows an aggregate coverage status (covered, or short by N riders)
- [ ] **COV-02**: A drive shows the list of which specific riders are still uncovered
- [ ] **COV-03**: Coverage reflects the drive's capacity behavior (hard cap vs. waitlist)

### Configurability (per drive)

- [ ] **CFG-01**: A drive can require, allow-optional, or hide a meeting point
- [ ] **CFG-02**: A drive's seat-selection mode is configurable: rider self-pick, driver-assign, or both allowed
- [ ] **CFG-03**: A drive's required rider info is configurable: name only, or name + contact
- [ ] **CFG-04**: A drive's capacity behavior is configurable: hard cap, or allow a waitlist

### Public API

- [ ] **API-01**: All core operations (create drive, offer/claim/assign seats, read coverage, configure a drive) are available via a clean public API
- [ ] **API-02**: Consuming applications authenticate with API credentials (API keys)
- [ ] **API-03**: API consumers are subject to per-consumer rate limits / quotas
- [ ] **API-04**: Anonymous participant routes and authenticated consumer routes are cleanly separated; the underlying service URL is never exposed in shared links

### Privacy & Compliance

- [ ] **PRIV-01**: Participant contact information is visible only to the organizer, never to other participants
- [ ] **PRIV-02**: The join screen presents a clear data-disclosure notice before info is collected
- [ ] **PRIV-03**: Participant personal data is deleted on a schedule anchored to the drive date (retention minimized; suitable for processing minors' data under EU/German GDPR)
- [ ] **PRIV-04**: Personal data is masked in application logs

## v2 Requirements

Deferred to a future release. Tracked but not in the current roadmap.

### Trips

- **TRIP-01**: A drive can have multiple intermediate pickup/drop-off stops
- **TRIP-02**: A drive can include a return / round trip leg

### Notifications

- **NOTF-01**: Participants are notified when coverage changes (push/email)
- **NOTF-02**: Configurable reminders before departure

### Configurability

- **CFG-05**: Organization-level default drive configuration (new drives inherit org defaults)
- **CFG-06**: Arbitrary custom/extra fields per organization

### Integration

- **INT-01**: First-class integration with handball-manager as an API consumer

## Out of Scope

Explicitly excluded. Documented to prevent scope creep.

| Feature | Reason |
|---------|--------|
| Lasting end-user accounts | Individuals stay anonymous/ephemeral; only high-volume API consumers get credentials. Keeps join friction near zero. |
| Notifications / push / reminders (v1) | The #1 over-built MVP carpool feature; pull-only is sufficient for pre-planned trips. Deferred to v2. |
| Multiple pickup/drop-off stops (v1) | Schema anticipates it (ordered stops), but single start→destination keeps the MVP lean. Deferred to v2. |
| Return / round trips (v1) | One leg only for v1. Deferred to v2. |
| Cost / fuel splitting | Out of core coordination value; commonly requested, deliberately excluded. |
| In-app chat / messaging | Not core to coverage; high maintenance surface. |
| GPS / live location tracking | Privacy-heavy, out of scope for a coordination engine. |
| Route optimization / auto-matching | Research shows auto-assignment "takes as long as starting from scratch" due to special cases; the per-drive self-pick/assign modes cover the real need. |
| Recurring drives | Deferred; not needed to prove core value. |
| Active handball-manager integration | Real but deferred consumer; API is built API-first so it can plug in later (INT-01 in v2). |
| Native mobile app | Standalone UI is a web/PWA; native is not committed for v1. |
| JSONB blob for the known config knobs | The 4 v1 knobs are typed columns to avoid config drift; free-form JSONB is an anti-pattern here. |

## Traceability

Which phases cover which requirements. Populated during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| TEN-01 | Phase 1 | Pending |
| TEN-02 | Phase 1 | Pending |
| TEN-03 | Phase 1 | Pending |
| DRIVE-01 | Phase 2 | Pending |
| DRIVE-02 | Phase 2 | Pending |
| DRIVE-03 | Phase 1 | Pending |
| DRIVE-04 | Phase 2 | Pending |
| PART-01 | Phase 2 | Pending |
| PART-02 | Phase 2 | Pending |
| PART-03 | Phase 2 | Pending |
| PART-04 | Phase 2 | Pending |
| PART-05 | Phase 2 | Pending |
| PART-06 | Phase 2 | Pending |
| PART-07 | Phase 2 | Pending |
| COV-01 | Phase 2 | Pending |
| COV-02 | Phase 2 | Pending |
| COV-03 | Phase 2 | Pending |
| CFG-01 | Phase 3 | Pending |
| CFG-02 | Phase 3 | Pending |
| CFG-03 | Phase 3 | Pending |
| CFG-04 | Phase 3 | Pending |
| API-01 | Phase 4 | Pending |
| API-02 | Phase 4 | Pending |
| API-03 | Phase 4 | Pending |
| API-04 | Phase 4 | Pending |
| PRIV-01 | Phase 2 | Pending |
| PRIV-02 | Phase 2 | Pending |
| PRIV-03 | Phase 4 | Pending |
| PRIV-04 | Phase 4 | Pending |

**Coverage:**
- v1 requirements: 28 total
- Mapped to phases: 28
- Unmapped: 0

---
*Requirements defined: 2026-06-02*
*Last updated: 2026-06-02 after roadmap creation*

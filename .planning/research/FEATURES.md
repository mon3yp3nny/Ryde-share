# Feature Research

**Domain:** Ride-coordination engine (group trip seat management) — flagship use case: sports-club away-game carpooling
**Researched:** 2026-06-02
**Confidence:** HIGH for table stakes and anti-features (grounded in 8+ comparable products); MEDIUM for differentiators (fewer direct precedents)

---

## Competitive Landscape Summary

Products examined before drawing conclusions:

| Product | Model | Key differentiator | Coverage view? |
|---|---|---|---|
| **Caroster** | Event carpool, open-source, no-account | Link-share, driver proposes route, passenger joins | Trips list only; no aggregate "X uncovered" |
| **GoKid** | School/sports carpool, subscription | GPS tracking, team roster integration | Notification when driver needed |
| **TeamSnap Carpool** | Embedded in sports mgmt, assignments tab | Manager can assign or riders self-pick | None — manual check |
| **SignUpReady** | Signup-sheet model, no download | Dashboard shows every slot + what's open | Slot-level only; organizer must count manually |
| **SignUpGenius Carpool** | Signup-sheet model | Highly configurable slots, no-account participant | Slot fill count; no "all covered?" rollup |
| **Carpoolio** | Recurring family carpool, Carma score | Fairness tracking, calendar sync | Per-activity view, not single-event |
| **GroupCarpool** | Event/trip, no-account | Waitlist notifications, browser-only | Implied by slot fill; no explicit status |
| **Whova Ride Sharing** | Conference/event embedded | Attendees self-form groups by arrival time | None |
| **CarpoolOrganiser** | Account-required event carpool | Request-based joining | Not shown |

**Critical observation:** No competitor surfaces an explicit, aggregate "is the whole group covered?" status. Everyone exposes slot counts or seat numbers — organizers must mentally calculate coverage. This is the primary gap ryde-share fills.

---

## Feature Landscape

### Table Stakes (Users Expect These)

Features that must exist or the product feels broken. Missing any one of these causes organizers to fall back to WhatsApp/spreadsheets.

| Feature | Why Expected | Complexity | Notes |
|---|---|---|---|
| Drive creation with start + destination | Carpool is meaningless without a trip | LOW | Single start→destination for v1; schema must anticipate multi-stop |
| Shareable link + QR code | Every comparable product offers this; it is how people recruit participants without accounts | LOW | The link IS the session — carries all context |
| No-account participation | Doodle, Caroster, SignUpGenius, GroupCarpool all make this work; requiring accounts kills adoption for casual group use | LOW-MEDIUM | Token/session-based identity for edits; just name (and optionally contact) captured |
| Driver seat offering (declare capacity) | Core mechanic — if drivers can't post seats, nothing else works | LOW | Integer capacity + optional meeting point; capacity modes (hard cap vs waitlist) already scoped |
| Rider self-claiming of open seats | Expected default from all signup-sheet and event-carpool products | LOW | Driver-assigns mode is differentiator; self-pick is baseline |
| Coverage status view — aggregate | The single most missed feature in all competitors: organizers currently count manually. An explicit "covered: 7 / needs ride: 3" state is the core value prop | MEDIUM | Binary states (covered / short-by-N) suffice for v1; details of uncovered names needed too |
| List of who is in which car | Required for organizer to verify and for riders to know their driver | LOW | Per-drive manifest, per-car passenger list |
| Organizer can see all cars + all unclaimed riders in one view | TeamSnap, SignUpReady both surface this; missing = constant scrolling | LOW | Single-page drive dashboard is the core organizer screen |
| Rider can claim/unclaim a seat before the drive | Any signup system allows this; without it, coordinators must manage by hand | LOW | Need token/cookie for "your own" entry so you can remove yourself |
| Drive event context (name, date, optional notes) | Sports clubs attach rides to a game; conferences attach rides to an event | LOW | Optional event association; free-text name + date is enough for v1 |

### Differentiators (Competitive Advantage)

Features that directly reinforce the core value ("is everyone covered?") or remove friction that all competitors leave on the table.

| Feature | Value Proposition | Complexity | Notes |
|---|---|---|---|
| Explicit aggregate coverage status with "short by N" count | No competitor computes this. An organizer currently has to count uncovered riders manually. A single badge ("3 still need a ride") is the primary UI hook. | MEDIUM | Drive-level computed field: total-riders-needing-a-ride minus total-covered-seats; updates on every change |
| Uncovered-riders list surfaced prominently | Who specifically is uncovered matters. Sports clubs need to chase down those 3 parents. | LOW | Simple list alongside the coverage count; no extra schema needed |
| Configurable seat-selection mode per drive (rider self-pick / driver assigns / both) | No competitor exposes this as a knob. TeamSnap does it one way; Caroster does it another. Different clubs have different norms — a trainer who assigns seats needs different behavior than a casual group. | MEDIUM | Mode is a drive-level config field; UI branches on it |
| Configurable required rider info per drive (name only vs name+contact) | Some contexts (internal club) need minimal info; external events need a contact number. No competitor surfaces this as a per-drive setting. | LOW | Optional field; form renders conditionally |
| Configurable meeting point per drive (required / optional / hidden) | Meeting points matter in some contexts (pick up at school gate) and are irrelevant in others (everyone drives to the same stadium lot). Hiding the field when irrelevant reduces noise. | LOW | Drive-level enum; form field renders conditionally |
| Configurable capacity behavior per drive (hard cap / waitlist) | Hard cap prevents overbooking; waitlist enables backup coordination. GroupCarpool notifies waitlisted riders when a seat opens — this is valued behavior. | MEDIUM | Waitlist requires ordered queue + state transitions |
| API-first design for embedding | No event-carpool product exposes a clean tenant-scoped API. This is the path to being embedded in handball-manager and others. | HIGH | This is architecture as differentiator — surfaces the engine as a platform, not just an app |
| Multi-tenant isolation | Each org sees only its own drives. Caroster is single-tenant per event; no competitor serves many orgs on one deployment with data isolation. | HIGH | Row-level scoping by tenant ID in all queries |
| QR code that encodes the drive join URL | QR codes are listed but rarely first-class. For sports clubs printing programs or posting flyers, a QR that drops someone directly into the drive is high value. | LOW | Just a QR render of the shareable URL — no new backend state |

### Anti-Features (Commonly Requested, Often Problematic)

Features to deliberately not build for MVP — each has a justification grounded in real product failure modes.

| Feature | Why Requested | Why Problematic | Alternative |
|---|---|---|---|
| Push/SMS notifications | GoKid sells this as premium; GroupCarpool sends waitlist emails | Requires sending infrastructure, provider relationships, opt-in flows, unsubscribe paths, and mobile OS permissions. Adds weeks of non-core work. v1 is pull-only and this is sufficient for sports clubs that check the app before game day. | Pull: organizer refreshes coverage view. Add notifications in v1.x once core is validated. |
| Real-time map of driver locations | GoKid premium; consumer rideshare expectation | GPS integration, background location permissions, battery drain, privacy concerns. Group-trip carpool is planned in advance — live tracking is a logistics feature, not a coordination feature. | Static meeting-point field + address on the drive card. |
| GPS route optimization / auto-matching | Routific, corporate carpool tools do this | Rider preferences are too complex for algorithms. Research finding: auto-assignment "takes the same time as starting from scratch" because of special circumstances. Creates distrust when algorithm puts wrong people together. | Manual assignment mode (driver assigns). Let humans do it. |
| In-app chat / messaging | Whova, GoKid, Carpoolio all offer this | Duplicates what organizers already have (WhatsApp, Signal). Creates a new communication silo people won't check. Adds moderation surface. | Contact info field (name+contact mode) lets drivers and riders coordinate off-platform. |
| User accounts for individuals | CarpoolOrganiser requires this; it kills casual participation | Registration kills adoption for one-time events and low-frequency club use. Friction = people stay on spreadsheets. | Session/token-based identity tied to the drive link. Cookie or URL token lets someone "own" their entry without an account. |
| Cost / fuel splitting | Carpoolio Carma Score, many requests | Turns a coordination tool into a payments platform. Payment flows, disputes, reconciliation, tax implications — out of scope complexity for a group-trip coordinator. Drivers in sports clubs are typically volunteers; no one expects payment. | Out of scope. Document that fuel splitting is not a ryde-share concern. |
| Recurring / scheduled drives | Calendar-sync products do this (Carpoolio, Caroster round-trips) | Recurrence adds state machine complexity (cancel one instance, edit all, etc.). Sports club schedules are irregular. Each away game is its own event. | Create a new drive per event. Low friction if drive creation is fast (it should be under 60 seconds). |
| Karma / fairness scoring | Carpoolio's Carma Score | Domain-specific to parent-rotation carpooling; irrelevant for sports-club away game model where the same volunteer drivers appear repeatedly. Adds social friction. | Not applicable to the flagship use case. |
| Multiple pickup stops | Multi-stop route planning | Adds routing complexity, driver confirmation loops, ordering logic. v1 is single start → destination per car. Schema should anticipate it but do not build. | Single meeting point per car. Real multi-stop can come in v2. |
| Waitlist auto-promotion notifications | Nice but requires notification infrastructure | See push notifications above. Waitlist auto-promotion is still useful without notification — organizer manually sees the waitlist and reaches out. | Waitlist queue visible in coverage view; organizer manages offline. |
| Return / round-trip leg | Users expect it for away games | Doubles the coordination surface, adds state coupling between legs. Return trips are often less structured — people leave at different times. | Create two separate drives (there and back). |
| Custom org branding / white-label | Enterprise ask | Design and CSS theming system; out of scope for lean v1. API consumers can render their own UI. | API-first design means consuming apps (handball-manager) bring their own UI. |

---

## Feature Dependencies

```
[Drive creation]
    └──requires──> [Tenant context]
                       └──requires──> [Multi-tenant isolation]

[Drive creation]
    └──enables──> [Shareable link + QR]
                      └──enables──> [Anonymous participant join]
                                        └──enables──> [Seat offering (driver)]
                                                           └──enables──> [Seat claiming (rider)]
                                                                              └──enables──> [Coverage status view]

[Coverage status view]
    └──requires──> [Seat offering (driver)]
    └──requires──> [Rider list on drive]
    └──computes──> [Covered count] + [Uncovered list]

[Configurable seat-selection mode]
    └──modifies──> [Seat claiming (rider)] — self-pick branch
    └──modifies──> [Seat assignment (driver)] — driver-assigns branch

[Waitlist capacity mode]
    └──extends──> [Seat claiming (rider)]
    └──requires──> [Hard cap enforcement] already present as baseline

[API-first design]
    └──requires──> [Multi-tenant isolation]
    └──requires──> [Rate limiting on API consumers]
    └──enables──> [Embedding by handball-manager and others]

[Drive event context]
    └──enhances──> [Drive creation] — optional association
```

### Dependency Notes

- **Coverage status requires rider list on drive:** Riders must explicitly declare they need a ride (by joining the drive as a rider) for coverage to be computable. If riders don't self-register, coverage cannot be determined. This is a UX implication: the join-as-rider flow is as important as the join-as-driver flow.
- **Configurable modes modify core claiming flow:** The seat-selection mode config is a branching condition in the UI and API, not a separate feature. Build the data model first, then branch.
- **Waitlist extends claiming:** Waitlist is an enhancement to the hard-cap baseline. Build hard-cap first; waitlist queue is an additive state machine layer.
- **API-first enables embedding but does not require it:** The consuming app (handball-manager) is deferred. API-first is the architecture constraint, not a feature to be built separately.

---

## MVP Definition

### Launch With (v1)

Minimum to answer "is everyone covered?" for a sports-club away game.

- [x] Drive creation — name, date, start point, destination, 4 config knobs
- [x] Shareable link + QR code generation from drive
- [x] Anonymous join as rider (name, optionally contact) via link
- [x] Anonymous join as driver (seat capacity, optionally meeting point) via link
- [x] Rider self-claiming of open seat (default mode)
- [x] Driver-assigns mode (alternative mode, per-drive config)
- [x] Hard-cap capacity enforcement
- [x] Aggregate coverage status: "covered: N / still need ride: M"
- [x] Uncovered-riders list in coverage view
- [x] Per-car manifest (who is in which car)
- [x] Multi-tenant isolation (org-scoped drives)
- [x] Public API for all core operations
- [x] Rate limiting for API consumers

### Add After Validation (v1.x)

Add once core coverage-coordination flow is proven with real sports clubs.

- [ ] Waitlist capacity mode — add once hard-cap is live and clubs ask for overflow handling
- [ ] Email notifications for waitlist promotion — add once notification infrastructure is justified by usage
- [ ] Drive duplication / "new drive from template" — add once organizers are creating many drives and complain about repeat setup

### Future Consideration (v2+)

Defer until product-market fit is clear.

- [ ] Multi-stop routes — schema should anticipate; do not build
- [ ] Return / round-trip leg coordination — low urgency; two drives workaround acceptable
- [ ] Push / SMS notifications — add when clubs ask for real-time alerting
- [ ] Cost splitting — only if clubs request it; unlikely for volunteer-driver model
- [ ] Custom org branding — only if API consumers need white-label embed

---

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---|---|---|---|
| Drive creation + config knobs | HIGH | LOW | P1 |
| Shareable link + QR | HIGH | LOW | P1 |
| Anonymous join (rider + driver) | HIGH | LOW | P1 |
| Seat offering + claiming | HIGH | LOW | P1 |
| Aggregate coverage status | HIGH | MEDIUM | P1 |
| Uncovered-riders list | HIGH | LOW | P1 |
| Per-car manifest | HIGH | LOW | P1 |
| Multi-tenant isolation | HIGH | MEDIUM | P1 |
| Public API + rate limiting | HIGH | MEDIUM | P1 |
| Driver-assigns mode | MEDIUM | LOW | P1 |
| Hard cap enforcement | MEDIUM | LOW | P1 |
| Waitlist mode | MEDIUM | MEDIUM | P2 |
| Drive duplication / templates | MEDIUM | LOW | P2 |
| Notifications | LOW | HIGH | P3 |
| Real-time location tracking | LOW | HIGH | P3 |
| Cost splitting | LOW | HIGH | P3 |

---

## Competitor Feature Analysis

| Feature | Caroster | TeamSnap | SignUpReady | GoKid | ryde-share approach |
|---|---|---|---|---|---|
| No-account participation | Yes — no registration to join | No — requires TeamSnap account | Yes — no download needed | No — app + account required | Yes — link/QR, name only |
| Aggregate coverage status | No — trips list only | No — manual slot check | Partial — open slots visible, no rollup | Notification-based, not a view | Yes — explicit "covered / short by N" |
| Per-drive configurable seat-selection | No — one mode per platform | Partial — organizer can assign or leave open | No | No | Yes — per-drive enum knob |
| Meeting-point configurability | Address on trip | None | Address per slot | Driver sets it | Yes — required/optional/hidden per drive |
| Driver-assigns mode | No | Partial — organizer can manually assign | No | No | Yes — explicit config mode |
| Waitlist | Yes — waiting list tab | No | No | No | Yes — optional capacity mode |
| API access | No | Yes (limited) | No | No | Yes — first-class API-first design |
| Multi-tenant | No — per-event instances | Yes — team isolation | No | No — per-family | Yes — org-scoped tenancy |
| QR code join | No | No | No | No | Yes — generated from drive link |

---

## Sources

- [Caroster — open-source event carpooling](https://caroster.io/en) and [GitHub](https://github.com/octree-gva/caroster)
- [TeamSnap carpool via Assignments tab](https://helpme.teamsnap.com/article/672-coordinate-carpools-using-the-assignments-tab)
- [SignUpReady sports team carpool sheets](https://www.signupready.com/for/sports-teams)
- [GoKid carpool for parents and sports](https://gokid.mobi/parents/)
- [SignUpGenius carpool sign-up guide](https://www.signupgenius.com/blog/sign-up-guide-carpools.cfm)
- [GroupCarpool — no-account group rideshare sheet](https://groupcarpool.com/)
- [Carpoolio — recurring family carpool](https://thecarpoolio.com/)
- [Whova ride-sharing for events](https://whova.com/blog/ridesharing-whova-event-app/)
- [Why carpool apps fail — Hacker News discussion](https://news.ycombinator.com/item?id=31329476)
- [Caroster best practices for event carpooling](https://caroster.io/en/blog/best-practices-for-event-carpooling)
- [Doodle no-account participation model](https://help.doodle.com/en/articles/9457227-do-i-need-a-doodle-account-to-participate-in-a-group-poll)
- [Kid Hop carpooling for sports teams](https://www.kidplay.tech/rolling-as-a-team-carpooling-for-sports-teams?locale=en)

---

*Feature research for: ride-coordination engine (ryde-share)*
*Researched: 2026-06-02*

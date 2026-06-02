<!-- GSD:project-start source:PROJECT.md -->

## Project

**Ryde-share**

Ryde-share is a multi-tenant, API-first, configurable **ride-coordination engine**. Organizations create "drives" — a group trip from a start point to a destination — where members offer seats and others claim or are assigned them, with a clear at-a-glance view of whether everyone who needs a ride is **covered**. The flagship use case is sports clubs coordinating rides to away games (players, parents, trainers offer to drive), but the engine is deliberately domain-agnostic: any kind of organization can use it. It works fully standalone via shareable link/QR drives with anonymous participants, and exposes a clean API so larger applications can embed it.

**Core Value:** A reliable, instantly-readable answer to **"is everyone covered?"** for a group trip — backed by an engine flexible enough that almost every behavior (meeting point, seat selection, capacity rules) adapts to each organization's context.

### Constraints

- **Infrastructure**: Runs on **Google Cloud Platform (GCP)** — the specific stack should be chosen as the best technical fit for an API-first, multi-tenant service on GCP (to be settled in research).
- **Architecture**: **API-first and modular** — every core capability must be available via API; the ride-sharing module must be self-contained and embeddable.
- **Tenancy**: **Multi-tenant** from the start — many independent organizations on one deployment, isolated.
- **Identity / friction**: Individuals must be able to participate **anonymously** (link/QR); no lasting end-user accounts. Accounts/keys exist only for high-volume API consumers, with **rate limiting**.
- **Extensibility**: Data model must anticipate multi-stop trips, return legs, and additional config knobs **without requiring a rewrite**, even though v1 doesn't build them.

<!-- GSD:project-end -->

<!-- GSD:stack-start source:research/STACK.md -->

## Technology Stack

## Recommended Stack

### Core Technologies

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| TypeScript (Node.js) | 5.x / Node 22 LTS | Backend language | Best-fit for an API-first service: first-class GCP SDK support, massive ecosystem, Hono + Drizzle + Zod share a unified type graph end-to-end. Go is faster at cold start but adds friction when consuming GCP client libraries and ships no ORM comparable to Drizzle. Python adds startup weight. |
| Hono | 4.12.23 | HTTP framework | ~14 KB, runs on Node.js / Bun / Edge runtimes — ideal for Cloud Run where cold-start matters. Built-in JWT, bearer-auth, CORS, secure-headers middleware. RPC client eliminates schema duplication. Zod validator adapter first-party. Faster ergonomics than Fastify for new greenfield projects. |
| Cloud Run (GCP) | — | Compute | Scale-to-zero serverless; no cluster ops; per-request billing fits low-to-medium traffic with unpredictable spikes. GCP-native: IAM, Secret Manager, Cloud SQL Auth Proxy, Artifact Registry all integrate cleanly. Set `--min-instances=1 --cpu-boost` to eliminate cold-start pain at ~$3/month. GKE is over-engineered for a single stateless API service with no GPU or long-lived worker needs. App Engine Flex is deprecated-direction. Cloud Run is the correct GCP tier for this workload. |
| Cloud SQL for PostgreSQL | PostgreSQL 16 | Primary datastore | Relational model fits the drive/seat/rider domain with FK-enforced integrity. RLS (Row Level Security) enables tenant isolation at the DB layer. Schema-based extensibility (JSONB config columns, nullable FK chains) handles multi-stop and config knobs without rewrite. Cloud SQL is ~26% cheaper than AlloyDB and this workload does not need AlloyDB's 4× transaction speed or columnar analytics tier. |
| GCP API Gateway | — | API gateway / rate limiting | Fully managed, serverless, integrates natively with Cloud Run. Accepts an OpenAPI 2.0 spec with `x-google-backend` and `x-google-management` quota stanzas. Tracks quotas per API key — fits the model of "anonymous individuals hit the API ungated; authenticated high-volume consumers get API keys with per-minute quotas." Apigee is enterprise-priced overkill; Cloud Endpoints requires self-managed proxy runtime. |
| SvelteKit | 2.61 / Svelte 5.56 | Standalone web UI / PWA shell | Compiles to minimal JS (15–25 KB client bundle vs 80–90 KB for Next.js). Progressive enhancement built-in; forms work without JS. PWA support first-class. The frontend is a thin API consumer, not a content-heavy SSR app — SvelteKit's lightweight adapter-node output deploys as another Cloud Run service or as static to Cloud Storage + CDN. Smaller Svelte ecosystem is acceptable here because the UI is intentionally thin and the engine is API-first. |

### Supporting Libraries

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| Drizzle ORM | 0.45.2 | PostgreSQL query builder + migrations | Use instead of Prisma: 7.4 KB bundle, near-instant cold start (vs Prisma's WASM engine even in v7), SQL-like syntax keeps RLS `set_config` calls explicit, schema defined in TS (no separate Prisma DSL). Use for all DB access. |
| drizzle-kit | 0.31.10 | Migration tooling | Generates SQL migrations from Drizzle schema definitions. Use for all schema changes. |
| Zod | 4.4.3 | Schema validation + inference | Use at every API boundary: request body, query params, env vars, drive config knobs. `@hono/zod-validator` bridges Hono route handlers. Zod v4 is 14× faster than v3; `zod/mini` (~1.9 KB) available for the frontend bundle. |
| @hono/zod-validator | 0.8.0 | Hono ↔ Zod integration | Use for all route-level request validation. Provides typed `c.req.valid()`. |
| jose | 7.19.0 | JWT sign / verify (JOSE standard) | Use for both API consumer JWTs (HS256 with server secret) and short-lived drive-participation tokens (HS256 signed, 24 h TTL, drive-scoped claims). Pure JS, no native bindings, no cold-start overhead. |
| pg | 8.21.0 | PostgreSQL driver | node-postgres; used by Drizzle under the hood. Configure `max: 5` per Cloud Run instance (Cloud Run default concurrency is 80 req/instance; PgBouncer sidecar or Cloud SQL's built-in PgBouncer handles multiplexing to DB). |
| @google-cloud/secret-manager | latest | Secrets at runtime | Store DB password, JWT signing secret. Cloud Run service account + IAM binding; no `.env` in containers. |
| Cloud SQL Auth Proxy (sidecar) | v2 | Secure DB connectivity | Runs as Cloud Run sidecar container; terminates IAM-authenticated TLS to Cloud SQL. Eliminates need to manage SSL certs or allow public IPs on the DB. |
| Vitest | 4.1.8 | Unit + integration testing | Fast, native TypeScript, compatible with Node.js. Use for domain logic tests and API handler tests with in-memory test doubles. |

### Development Tools

| Tool | Purpose | Notes |
|------|---------|-------|
| Docker / Cloud Build | Container image build + CI | Use Cloud Build triggers on GitHub push; push to Artifact Registry; deploy to Cloud Run. |
| Terraform (or `gcloud` CLI) | Infrastructure as code | Cloud SQL, Cloud Run services, API Gateway config, IAM bindings. Terraform GCP provider is mature; keeps infra reviewable. |
| ESLint + Prettier | Code quality | Standard TypeScript setup; run in CI. |
| `tsx` (or `ts-node`) | Local dev execution | Run Hono server directly without compile step. |

## Installation

# Core backend

# GCP integrations

# Dev dependencies

# Frontend (separate SvelteKit project)

## Architecture Decisions

### Compute: Cloud Run

- Stateless HTTP (no persistent WebSocket, no background workers in v1)
- Variable traffic (sports club use case: spikes around game days)
- Single-region initially

### Database: Cloud SQL PostgreSQL with RLS

### API Gateway: GCP API Gateway

### Anonymous Participation: Signed Drive Tokens

- Invite links are revocable by the organizer (set an `invalidated_at` column on the drive and reject tokens issued before it)
- Session tokens are short-lived; no refresh token needed (re-joining the link re-issues a session)
- HTTPS always (Cloud Run enforces this)
- `jose` uses constant-time comparison for HMAC verification

### Frontend: SvelteKit (thin API consumer)

- **Static adapter** (`@sveltejs/adapter-static`): export as static site, serve from Cloud Storage + Cloud CDN. Zero compute cost. Works because the app is pure client-side after initial load.
- **Node adapter** (`@sveltejs/adapter-node`): deploy as a Cloud Run service if SSR is needed for SEO or initial state hydration. More control, small cost.

## Alternatives Considered

| Recommended | Alternative | Why Not |
|-------------|-------------|---------|
| Cloud Run | GKE | No Kubernetes ops needed for a single stateless service; GKE adds ~$200–400/month node baseline even with Autopilot. Revisit if multi-service orchestration becomes necessary. |
| Cloud Run | App Engine Standard | App Engine is not receiving new major investment; Cloud Run supersedes it for containerized workloads. |
| TypeScript (Hono) | Go | Go has faster cold starts (~100 ms vs ~500 ms for Node.js) and lower memory per instance. Disadvantage: no ORM comparable to Drizzle, weaker GCP SDK ergonomics, and the team context implies TypeScript. The cold-start gap narrows with `--min-instances=1`. |
| TypeScript (Hono) | Python (FastAPI) | FastAPI is excellent but Python cold starts are heavier than Node.js on Cloud Run. No strong reason to diverge from TypeScript if the frontend is also TypeScript (shared types possible). |
| Hono | Fastify | Both are production-quality. Hono wins on bundle size (14 KB vs ~200 KB for Fastify core + plugins), runs on all runtimes, and has cleaner TypeScript ergonomics. Fastify's plugin encapsulation is better for very large teams but adds cognitive overhead for a solo/small team. |
| Hono | NestJS | NestJS is opinionated DI framework suited for teams of 3+ building long-lived large backends. The ceremony (decorators, modules, providers) is disproportionate for a focused single-service API. |
| Cloud SQL PostgreSQL | AlloyDB | AlloyDB is ~26% more expensive with no benefit for this workload. AlloyDB's advantages (4× transaction speed, 100× analytics) are relevant at >10K concurrent users or complex analytical queries. Start with Cloud SQL; AlloyDB is a viable upgrade path with no application code changes (same PostgreSQL wire protocol). |
| Cloud SQL PostgreSQL | Firestore | Drive/seat/rider coverage is a relational join problem. Firestore's document model forces denormalization that breaks referential integrity and complicates coverage calculations. RLS is also impossible on Firestore. |
| RLS shared-schema | Schema-per-tenant | Schema-per-tenant degrades past ~100 schemas, requires per-tenant migration orchestration, and gives no meaningful security advantage over properly implemented RLS. |
| GCP API Gateway | Apigee | Apigee X starts at ~$2,500/month minimum spend. Overkill for a startup. API Gateway is consumption-priced (~$3/million calls). |
| GCP API Gateway | Cloud Endpoints | Endpoints requires deploying and managing the ESP v2 proxy as a sidecar. More operational overhead with no advantage over API Gateway for this use case. |
| GCP API Gateway | Self-rolled rate limiting in Hono | Application-layer rate limiting can be bypassed by attacking multiple instances. Gateway-level enforcement is authoritative regardless of scaling. |
| Drizzle ORM | Prisma | Prisma 7 improved significantly (removed Rust engine), but Drizzle's 7.4 KB bundle and near-instant cold start still win for Cloud Run. Drizzle's SQL-like syntax keeps `set_config` RLS calls explicit rather than hidden. |
| jose (JWT) | jsonwebtoken | `jsonwebtoken` does not support the Web Crypto API and has had CVEs. `jose` implements JOSE standards (RFC 7515/7517/7519) using Web Crypto, is actively maintained, and has no native dependencies. |
| SvelteKit | Next.js | Next.js ships 80–90 KB of React runtime before application code. For a thin drive-coordination UI consumed via link/QR, SvelteKit's 15–25 KB bundle is meaningfully better for mobile load times. React ecosystem advantage is wasted when the UI has no complex component library needs. |
| SvelteKit | Vanilla TypeScript + Vite | SvelteKit gives routing, form actions, and SSR/static generation with almost no boilerplate. For a multi-page PWA, raw Vite would require building the same routing infrastructure manually. |

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| Firebase / Firestore | Document model breaks relational coverage queries; no RLS; vendor lock-in diverges from Cloud SQL investment. | Cloud SQL PostgreSQL |
| Sequelize | Outdated TypeScript support, verbose, slow migration tooling. Active ecosystem has moved to Drizzle/Prisma. | Drizzle ORM |
| TypeORM | TypeScript decorator-based; requires `experimentalDecorators`; known performance issues with complex queries; slower than Drizzle. | Drizzle ORM |
| jsonwebtoken | Lacks Web Crypto API support; historical CVEs; not actively moving toward modern JS standards. | jose |
| Cloud Endpoints | Self-managed ESP v2 proxy; operational overhead for no benefit over API Gateway; grpc-focused origin makes REST use awkward. | GCP API Gateway |
| App Engine | Not receiving new feature investment; Cloud Run is the modern equivalent with better DX and pricing model. | Cloud Run |
| Hardcoded org filter in every query | Application-level tenant filtering is a footgun — one missed WHERE clause leaks data across tenants. | PostgreSQL RLS policies |
| Session-level `set_config` for RLS | In PgBouncer transaction mode, session-level settings persist across connection reuse and leak tenant context to the next request. | Transaction-scoped `set_config($key, $value, true)` |
| `localStorage` for organizer tokens on the standalone UI | Organizer tokens have broader permissions; localStorage is accessible to any JS on the page. | `HttpOnly` cookie for organizer token on standalone UI; localStorage acceptable only for ephemeral participant session tokens. |

## Stack Patterns by Variant

- The handball-manager app authenticates with an API key issued by GCP API Gateway.
- It calls the ryde-share REST API exactly as any other consumer. No special integration layer needed — the API-first design handles this.
- Rate limiting quota assigned to the handball-manager API key is separate from anonymous traffic.
- The DB schema uses a `stops` table with `sequence_index` and nullable `waypoint` geometry (PostGIS extension on Cloud SQL). The `drives` table retains `origin` and `destination` as the primary pair; stops extend it without altering the base structure.
- No API surface changes for consumers who only use single-stop drives.
- Cloud Run scales horizontally automatically. Each new instance gets `min: 5` PG connections. Add a PgBouncer sidecar Cloud Run service or switch to Cloud SQL Enterprise Plus for Managed Connection Pooling.
- AlloyDB upgrade: change the Cloud SQL connection string. Zero application code changes (same PostgreSQL wire protocol).
- Add Cloud Tasks (task queue) for scheduled reminders. Cloud Run handles the worker endpoint.
- Pub/Sub for real-time seat-change events. The current pull-only architecture does not block this; it is additive.

## Version Compatibility

| Package | Compatible With | Notes |
|---------|-----------------|-------|
| hono@4.12.23 | @hono/zod-validator@0.8.0 | Verified compatible; both on v4 API. |
| zod@4.4.3 | @hono/zod-validator@0.8.0 | zod-validator 0.8.0 updated for Zod v4; do not use zod-validator <0.7 with Zod v4. |
| drizzle-orm@0.45.2 | pg@8.21.0 | Use `drizzle-orm/node-postgres` import path. |
| drizzle-orm@0.45.2 | drizzle-kit@0.31.10 | Both must be updated together; minor version mismatch causes migration generation errors. |
| @sveltejs/kit@2.61.1 | svelte@5.56.1 | SvelteKit 2.x requires Svelte 5; Svelte 5 runes API is stable. |
| Cloud Run Node 22 | hono@4 + Drizzle + pg | All packages verified compatible with Node 22 LTS. |

## Sources

- [GCP: Choosing between Apigee, API Gateway, and Cloud Endpoints](https://cloud.google.com/blog/products/application-modernization/choosing-between-apigee-api-gateway-and-cloud-endpoints) — API gateway comparison (official, HIGH confidence)
- [GCP: GKE and Cloud Run concepts](https://docs.cloud.google.com/kubernetes-engine/docs/concepts/gke-and-cloud-run) — compute tier decision (official, HIGH confidence)
- [GCP: Cloud Run min instances](https://docs.cloud.google.com/run/docs/configuring/min-instances) — cold start strategy (official, HIGH confidence)
- [GCP: Cloud SQL Managed Connection Pooling](https://docs.cloud.google.com/sql/docs/postgres/managed-connection-pooling) — Enterprise Plus requirement confirmed (official, HIGH confidence)
- [Bytebase: AlloyDB vs Cloud SQL](https://www.bytebase.com/blog/alloydb-vs-cloudsql/) — cost and performance comparison (MEDIUM confidence, verified against official pricing)
- [Hono docs](https://hono.dev/docs) — feature set, middleware catalog (official, HIGH confidence)
- [Encore: NestJS vs Fastify vs Hono 2026](https://encore.dev/articles/nestjs-vs-fastify-vs-hono) — framework comparison (MEDIUM confidence)
- [Better Stack: Hono vs Fastify](https://betterstack.com/community/guides/scaling-nodejs/hono-vs-fastify/) — benchmark data (MEDIUM confidence)
- [Bytebase: Drizzle vs Prisma 2026](https://www.bytebase.com/blog/drizzle-vs-prisma/) — ORM comparison (MEDIUM confidence)
- [InfoQ: Zod v4 release](https://www.infoq.com/news/2025/08/zod-v4-available/) — v4 confirmed stable August 2025 (HIGH confidence)
- [Rico Fritzsche: PostgreSQL RLS for multi-tenancy](https://ricofritzsche.me/mastering-postgresql-row-level-security-rls-for-rock-solid-multi-tenancy/) — RLS patterns (MEDIUM confidence, consistent with official PG docs)
- [Crunchy Data: RLS for tenants](https://www.crunchydata.com/blog/row-level-security-for-tenants-in-postgres) — transaction-scoped set_config pattern (HIGH confidence, official PG partner)
- [DEV Community: Hono vs Express vs Fastify 2026](https://dev.to/alex_aslam/beyond-express-fastify-vs-hono-which-wins-for-high-throughput-apis-373i) — real-world benchmark at DB query level (MEDIUM confidence)
- [DEV Community: SvelteKit vs Next.js 2026](https://dev.to/paulthedev/sveltekit-vs-nextjs-in-2026-why-the-underdog-is-winning-a-developers-deep-dive-155b) — bundle size and PWA comparison (MEDIUM confidence)
- npm package versions verified via `npm view` at research time (HIGH confidence for versions)

<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->

## Conventions

Conventions not yet established. Will populate as patterns emerge during development.
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->

## Architecture

Architecture not yet mapped. Follow existing patterns found in the codebase.
<!-- GSD:architecture-end -->

<!-- GSD:skills-start source:skills/ -->

## Project Skills

No project skills found. Add skills to any of: `.claude/skills/`, `.agents/skills/`, `.cursor/skills/`, `.github/skills/`, or `.codex/skills/` with a `SKILL.md` index file.
<!-- GSD:skills-end -->

<!-- GSD:workflow-start source:GSD defaults -->

## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:

- `/gsd:quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd:debug` for investigation and bug fixing
- `/gsd:execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->

<!-- GSD:profile-start -->

## Developer Profile

> Profile not yet configured. Run `/gsd:profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->

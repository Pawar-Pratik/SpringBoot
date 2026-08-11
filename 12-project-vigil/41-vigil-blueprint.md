# Lesson 41 — Vigil: the blueprint

> **Read this early** (skim it after Module 2) so you know where you're heading, then return to it whenever you need the full picture. This is the specification — the domain model, the API, the architecture, and the reasoning behind each decision. **The reasoning is the valuable part**: anyone can build CRUD, but being able to explain *why* your schema, boundaries and trade-offs are what they are is what makes this project interview leverage rather than a code sample.

---

## 1. The product

**Vigil is an open-source uptime and incident management platform.** Self-hostable, multi-tenant, and genuinely useful.

**What a user does:**
1. Signs up (email/password, or Google/GitHub), which creates their organization
2. Adds monitors — an HTTP endpoint, a TCP port, an SSL certificate, or a cron heartbeat
3. Vigil checks each on its interval from a pool of workers
4. On failure, Vigil opens an **incident** and escalates through a policy
5. Notifications go out by email, Slack, or webhook
6. A **public status page** shows current state and history to their customers
7. Reports show uptime %, p95 latency, MTTR, and SLA compliance

**The elevator pitch for an interview:** *"A self-hostable uptime monitoring and incident management platform — think a small Better Uptime. Multi-tenant, checks thousands of endpoints on schedule across horizontally-scaled workers, opens and escalates incidents, delivers notifications with retries and circuit breakers, and publishes cached public status pages. It's deployed at vigil.example.dev with a demo login."*

**Why this project rather than another CRUD app:** every technology in it is there because the domain *requires* it, not because it looks impressive. Scheduling, outbound HTTP with resilience, high-volume time-series writes, caching, async notification, multi-tenancy, and horizontal scaling are all forced by the problem. **That means every architectural decision has a real justification you can defend** — which is exactly what a design interview probes.

---

## 2. Domain model

```
┌───────────────┐         ┌──────────────┐         ┌───────────────┐
│ Organization  │◀───────▶│  Membership  │◀───────▶│   AppUser     │
│ (tenant)      │  1    * │ role         │ *     1 │               │
│ · name, slug  │         │ joinedAt     │         │ · email       │
│ · plan        │         │ invitedBy    │         │ · passwordHash│
└───────┬───────┘         └──────────────┘         │ · tokenVersion│
        │ 1                                         └───────┬───────┘
        │                                                   │ 1
        │ *                                                 │ *
┌───────▼────────┐                                  ┌───────▼────────┐
│    Monitor     │                                  │ FederatedIdent │
│ · name, url    │                                  │ provider + sub │
│ · type (enum)  │                                  └────────────────┘
│ · status       │
│ · intervalSecs │      ┌───────────────────┐
│ · config(jsonb)│─────▶│  CheckResult      │   append-only, high volume
│ · nextCheckAt  │  1  *│ · status          │   partitioned by month
│ · @Version     │      │ · responseTimeMs  │   UUIDv7 primary key
└───────┬────────┘      │ · statusCode      │
        │ 1             │ · error           │
        │               │ · checkedAt       │
        │ *             └───────────────────┘
┌───────▼────────┐
│    Incident    │      ┌───────────────────┐      ┌──────────────────┐
│ · status       │─────▶│ IncidentEvent     │      │ EscalationPolicy │
│ · startedAt    │  1  *│ (audit timeline)  │      │ · steps (jsonb)  │
│ · acknowledged │      └───────────────────┘      └──────────────────┘
│ · resolvedAt   │
│ · @Version     │      ┌──────────────────────┐   ┌──────────────────┐
└────────────────┘      │ NotificationChannel  │   │   StatusPage     │
                        │ · type, config(jsonb)│   │ · slug (unique)  │
┌────────────────┐      │ · enabled            │   │ · monitors (M:N) │
│  OutboxMessage │      └──────────────────────┘   └──────────────────┘
│ · topic, key   │
│ · payload      │      ┌──────────────────┐  ┌──────────────┐  ┌──────────┐
│ · publishedAt  │      │  RefreshToken    │  │ ProcessedEvt │  │ ShedLock │
└────────────────┘      │ hash, family, exp│  │ (idempotency)│  └──────────┘
                        └──────────────────┘  └──────────────┘
```

### The modelling decisions, and their justifications

| Decision | Why |
|---|---|
| **`Membership` as an entity, not `@ManyToMany`** | It needs `role`, `joinedAt`, `invitedBy` — a join table can't hold them, and retrofitting is a migration (Lesson 21 §2) |
| **`Monitor.config` as `jsonb`** | Per-type configuration differs (expected status, keyword, TCP port, cert expiry threshold). A rigid column set would be mostly nulls; entity inheritance would be heavier than the problem warrants (Lesson 21 §5) |
| **`CheckResult` has no JPA cascade from `Monitor`** | Millions of rows — a cascade would load them all into the persistence context. Uses DB-level `ON DELETE CASCADE` + a bulk delete (Lesson 21 §3) |
| **`CheckResult` written with `JdbcClient`, not JPA** | Batched, append-only, no entity semantics needed. JPA's per-entity overhead is pure cost here (Lesson 19 §4) |
| **UUIDv7 primary keys** | Non-enumerable (multi-tenant API — no `/monitors/1` walking) but time-ordered, so no B-tree fragmentation on a high-insert table (Lesson 20 §4) |
| **`@Version` on `Monitor` and `Incident`** | Concurrently edited by humans; optimistic locking prevents lost updates and yields a 409 (Lesson 23 §5) |
| **`nextCheckAt` on `Monitor`** | Enables `FOR UPDATE SKIP LOCKED` work claiming — horizontal scaling with no coordinator (Lesson 32 §2) |
| **`org_id` on every table** | Discriminator multi-tenancy: best scale for many small tenants, defended in four layers (Lesson 24 §4) |
| **`OutboxMessage`** | Atomic "state changed" + "notification owed" (Lesson 32 §3) |
| **`ProcessedEvent` + a unique constraint on `(monitor_id, checked_at)`** | Idempotent consumers for at-least-once delivery (Lessons 34, 37) |
| **`tokenVersion` on `AppUser`** | Immediate JWT revocation on password/role change (Lesson 26 §6) |

**Two of these are worth volunteering unprompted in an interview**, because they show genuine depth: the UUIDv7 reasoning (security *and* index locality, a trade-off most people don't know exists) and the `SKIP LOCKED` design (horizontal scaling with no broker or leader election).

---

## 3. Schema highlights

```sql
CREATE TABLE monitor (
    id               UUID PRIMARY KEY,
    org_id           UUID NOT NULL REFERENCES organization(id),
    name             TEXT NOT NULL,
    url              TEXT NOT NULL,
    type             TEXT NOT NULL,
    status           TEXT NOT NULL DEFAULT 'PENDING',
    interval_seconds INT  NOT NULL CHECK (interval_seconds >= 30),
    config           JSONB NOT NULL DEFAULT '{}',
    next_check_at    TIMESTAMPTZ,
    claimed_by       TEXT,
    claimed_at       TIMESTAMPTZ,
    version          BIGINT NOT NULL DEFAULT 0,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT uq_monitor_org_name UNIQUE (org_id, name)
);

-- ★ Index your foreign keys — Postgres does NOT do this automatically (Lesson 24)
CREATE INDEX idx_monitor_org ON monitor (org_id);

-- ★ PARTIAL index: the scheduler only ever queries non-paused monitors that are due.
--   A fraction of the size of a full index, and it stays in cache.
CREATE INDEX idx_monitor_due ON monitor (next_check_at)
    WHERE status <> 'PAUSED';

CREATE TABLE check_result (
    id               UUID NOT NULL,              -- UUIDv7
    monitor_id       UUID NOT NULL,
    org_id           UUID NOT NULL,              -- ★ denormalized for tenant-scoped queries + RLS
    status           TEXT NOT NULL,
    response_time_ms INT,
    status_code      INT,
    error            TEXT,
    checked_at       TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (id, checked_at),
    CONSTRAINT uq_check UNIQUE (monitor_id, checked_at)   -- ★ makes duplicate delivery a no-op
) PARTITION BY RANGE (checked_at);                        -- ★ monthly partitions

-- ★ Composite index: equality column FIRST, then the sort column (Lesson 24)
CREATE INDEX idx_cr_monitor_time ON check_result (monitor_id, checked_at DESC)
    INCLUDE (status, response_time_ms);                   -- ★ covering → Index Only Scan

ALTER TABLE monitor ENABLE ROW LEVEL SECURITY;            -- ★ DB-enforced tenant isolation
CREATE POLICY tenant_isolation ON monitor
    USING (org_id = current_setting('app.current_org', true)::uuid);
```

**Four things in there are worth being able to explain**, because each is a deliberate decision rather than a default:

1. **The partial index** — the scheduler's query is `WHERE next_check_at <= now() AND status <> 'PAUSED'`, so indexing only non-paused rows makes it dramatically smaller and keeps it hot in cache.
2. **`org_id` denormalized onto `check_result`** — it's derivable via `monitor_id`, but joining on every tenant-scoped query and for RLS would be wasteful. **A deliberate, justified denormalization.**
3. **`UNIQUE (monitor_id, checked_at)`** — this is idempotency *enforced by the database*, so at-least-once delivery from the outbox is automatically safe (Lesson 34 §5).
4. **Monthly partitioning** — retention becomes `DROP PARTITION` (instant, no bloat) instead of `DELETE` over millions of rows (slow, generates dead tuples, needs `VACUUM`).

---

## 4. Architecture

### Phase 1 — the modular monolith (build this)

```
com.vigil
├── organization      orgs, users, memberships, invitations, API keys
├── monitor           CRUD, config, ownership, quota enforcement
├── check             ★ the execution engine
│   ├── scheduler     claims due monitors (SKIP LOCKED)
│   ├── executor      HTTP / TCP / SSL / heartbeat  (self-selecting strategy)
│   └── result        storage (JdbcClient, batched) + aggregation
├── incident          lifecycle state machine, escalation
├── notification      channels + dispatch (Slack / email / webhook)
├── statuspage        public pages, cached, rate-limited
├── report            uptime %, p95, MTTR, SLA
├── billing           plans, quotas
├── common            exceptions, audit, security helpers, pagination
└── config            infrastructure wiring
```

**Boundaries enforced by:** package-private repositories and entities, ArchUnit rules, and no cross-feature entity relationships (Lessons 13, 31).

### Phase 2 — the optional split (only after Phase 1 works)

```
              ┌──────────────┐
              │   Gateway    │  auth at the edge, routing, rate limiting
              └──────┬───────┘
     ┌───────────────┼───────────────┐
┌────▼─────┐   ┌─────▼──────┐  ┌────▼──────┐
│vigil-api │   │vigil-checker│  │vigil-notif│
│  3 pods  │   │  50 pods    │  │  3 pods   │
└────┬─────┘   └──────┬──────┘  └─────┬─────┘
  ┌──▼───┐        ┌───▼───┐        ┌──▼───┐
  │config│        │results│        │deliv.│
  │  DB  │        │  DB   │        │ log  │
  └──────┘        └───────┘        └──────┘
       └────────── Kafka ──────────────┘
```

**Justified by measurement, in an ADR** (Lesson 38 §7). And **note what stays together:** orgs, monitors, quotas and incidents are all in `vigil-api`, because creating a monitor must check the plan quota in one transaction. **Avoiding a saga by drawing the boundary correctly is the better engineering choice**, and saying so is stronger than building the saga.

---

## 5. The API surface

```
── Auth ────────────────────────────────────────────────────────────
POST   /api/v1/auth/register                 → 201 + tokens
POST   /api/v1/auth/login                    → 200 + tokens
POST   /api/v1/auth/refresh                  → 200 (rotation + reuse detection)
POST   /api/v1/auth/logout                   → 204
GET    /api/v1/auth/me                       → current user + orgs + roles
GET    /oauth2/authorization/{google|github} → social login

── Organizations ───────────────────────────────────────────────────
GET    /api/v1/orgs
POST   /api/v1/orgs
GET    /api/v1/orgs/{orgId}
GET    /api/v1/orgs/{orgId}/members
POST   /api/v1/orgs/{orgId}/invitations
PATCH  /api/v1/orgs/{orgId}/members/{userId}     change role

── Monitors ────────────────────────────────────────────────────────
GET    /api/v1/orgs/{orgId}/monitors             ?status=&q=&page=&size=&sort=
POST   /api/v1/orgs/{orgId}/monitors             Idempotency-Key supported
GET    /api/v1/orgs/{orgId}/monitors/{id}        ETag / 304
PUT    /api/v1/orgs/{orgId}/monitors/{id}        If-Match → 409 on conflict
DELETE /api/v1/orgs/{orgId}/monitors/{id}        → 204
POST   /api/v1/orgs/{orgId}/monitors/{id}/pause
POST   /api/v1/orgs/{orgId}/monitors/{id}/check  run now (sync)

── Check results & reports ─────────────────────────────────────────
GET    /api/v1/orgs/{orgId}/monitors/{id}/results   ?after=<cursor>&limit=  (cursor paging)
GET    /api/v1/orgs/{orgId}/monitors/{id}/uptime    ?days=30
GET    /api/v1/orgs/{orgId}/reports/sla             ?from=&to=

── Incidents ───────────────────────────────────────────────────────
GET    /api/v1/orgs/{orgId}/incidents            ?status=OPEN
POST   /api/v1/orgs/{orgId}/incidents/{id}/acknowledge
POST   /api/v1/orgs/{orgId}/incidents/{id}/resolve

── Notification channels ───────────────────────────────────────────
GET/POST/DELETE /api/v1/orgs/{orgId}/channels
POST   /api/v1/orgs/{orgId}/channels/{id}/test

── Public (unauthenticated, cached, rate-limited) ──────────────────
GET    /status/{slug}                            server-rendered HTML
GET    /api/v1/public/status/{slug}              JSON, ETag, 30s cache

── Operational ─────────────────────────────────────────────────────
GET    :9090/actuator/health/{liveness,readiness}
GET    :9090/actuator/prometheus
GET    /v3/api-docs  ·  /swagger-ui.html
```

**Design decisions to be able to defend:**

- **Tenant in the path** (`/orgs/{orgId}/...`) — explicit, cacheable, and verified against memberships by a guard. **Still validated server-side** — a path variable is client input (Lesson 28 §3).
- **Cursor pagination for check results, offset for monitors** — millions of append-only rows versus a small set needing page numbers. **Different access patterns, different strategies** (Lesson 16 §4).
- **ETags on monitors and status pages** — the status page is polled by many browsers; 304s cost nothing (Lesson 33 §7).
- **`Idempotency-Key` on POST** — clients retry on timeout (Lesson 34 §5).
- **`If-Match` on PUT → 409** — optimistic locking surfaced properly in HTTP (Lesson 23 §5).
- **`RFC 9457 ProblemDetail` for every error**, with a correlation ID (Lesson 16 §3).
- **Actions as sub-resources** (`/pause`, `/acknowledge`) — a pragmatic departure from REST purism for operations that aren't CRUD. **Being willing to say "I chose pragmatism here, and here's why" is better than defending dogma.**

---

## 6. Key flows

### The check cycle (the heart of the system)

```
① Every 5s, on EVERY instance:
     UPDATE monitor SET next_check_at = now() + interval, claimed_by = :worker
     WHERE id IN (SELECT id FROM monitor
                  WHERE next_check_at <= now() AND status <> 'PAUSED'
                  ORDER BY next_check_at LIMIT 50
                  FOR UPDATE SKIP LOCKED)                    ← ★ disjoint batches, no coordinator
     RETURNING ...

② Dispatch each to a virtual-thread executor          (I/O-bound → virtual threads)

③ Per check: RestClient with connect+read timeouts, a per-target circuit breaker,
   SSRF validation at connection time                  (Lessons 16, 34)

④ Batch-insert results with JdbcClient (reWriteBatchedInserts)
   ON CONFLICT (monitor_id, checked_at) DO NOTHING      ← ★ idempotent

⑤ On a status transition, in the SAME transaction:
   · update monitor.status
   · write an OutboxMessage                             ← ★ atomic (Lesson 32)

⑥ The outbox relay publishes to Kafka (or fires an event in the monolith)

⑦ IncidentService opens/resolves an incident (idempotent — one open incident per monitor)

⑧ NotificationDispatcher delivers with retries, backoff+jitter, circuit breaker, and a DLT
```

**Every numbered step maps to a specific lesson.** That traceability is what makes this project a demonstration of the curriculum rather than a separate exercise.

### The incident state machine

```
        check fails (n consecutive)
   ┌──────────────────────────────▶ OPEN ──────────┐
   │                                 │             │ acknowledge
(none)                        recovery│             ▼
   │                                 │        ACKNOWLEDGED
   │                                 │             │ recovery
   └◀──────── RESOLVED ◀─────────────┴─────────────┘
                  │
                  │ fails again within the flap window
                  └──▶ reopened (same incident, so MTTR isn't gamed)
```

**Rules that matter, and each is a real product decision:**
- **`n` consecutive failures before opening** (default 2) — avoids alerting on a single transient blip. Configurable per monitor.
- **One open incident per monitor** — enforced by a partial unique index, so redelivery can't create duplicates.
- **Acknowledgement stops escalation but not resolution tracking** — a human is on it, but the outage continues until recovery.
- **Reopening within a flap window reuses the incident** — otherwise a flapping endpoint produces dozens of incidents and a meaningless MTTR.

**That last rule is the kind of detail that makes a product feel considered**, and it's a great thing to mention: *"I reuse the incident on a flap rather than opening a new one, because otherwise MTTR looks artificially good and the on-call engineer gets paged repeatedly for one problem."*

---

## 7. Build order

Each phase is independently demonstrable. **Ship each one before starting the next.**

| Phase | Deliverable | Lessons |
|---|---|---|
| **0** | Project skeleton, Docker Compose (Postgres), Actuator | 01, 11–13 |
| **1** | Monitor CRUD: DTOs, validation, `ProblemDetail`, OpenAPI | 14–17 |
| **2** | Persistence: entities, Flyway, repositories, projections | 19–24 |
| **3** | Auth: users, orgs, memberships, JWT + refresh rotation | 25–26 |
| **4** | Authorization: permissions, tenant isolation, RLS, IDOR tests | 28 |
| **5** | **The check engine** — scheduler, executors, results | 32, 34 |
| **6** | Incidents + notifications (events, outbox, async) | 32, 34 |
| **7** | Status pages + caching + rate limiting | 15, 17, 33 |
| **8** | Reports: uptime, p95, MTTR (`JdbcClient` aggregation) | 19, 22 |
| **9** | Testing: unit, Testcontainers, security, ArchUnit, WireMock | 29–31 |
| **10** | Observability: metrics, tracing, dashboards, alerts | 35 |
| **11** | **Deploy** — Docker, CI/CD, a live URL, demo data | 40 |
| **12** | Social login, API keys, invitations, webhooks | 27 |
| **13** | *Optional:* Kafka, service split, gateway | 37–39 |

**Phase 11 is not last by accident** — deploy as early as you can, and redeploy continuously. **A project that has been live for six months is a different artifact from one deployed the night before an interview**, and it shows in the commit history.

---

## 8. What makes this impressive — the specifics

When someone reviews this project, these are what they'll notice. **Do them deliberately.**

**Architecture**
- [ ] `ARCHITECTURE.md` explaining the boundaries and the reasoning
- [ ] ADRs for real decisions (`docs/adr/`) — **including one you'd now decide differently**
- [ ] ArchUnit tests as an executable architecture diagram
- [ ] Package-private repositories — compiler-enforced module boundaries

**Correctness**
- [ ] `SKIP LOCKED` work claiming, with a test proving two workers get disjoint batches
- [ ] Outbox pattern, with a test proving delivery survives `kill -9`
- [ ] Idempotency at every level: HTTP keys, DB constraints, consumer dedupe
- [ ] Optimistic locking surfaced as 409 with `If-Match`

**Security**
- [ ] Cross-tenant tests returning **404** for every endpoint
- [ ] SSRF defence with connection-time IP checks, and tests including a redirect to metadata
- [ ] RLS as a database-enforced backstop
- [ ] Refresh token rotation with reuse detection
- [ ] gitleaks in CI; no secrets in the repo, ever

**Performance**
- [ ] A **before/after table** of measured optimizations (p99 2.4s → 180ms, and how)
- [ ] N+1 regression tests asserting query counts
- [ ] k6 load-test scripts committed, with results in the README

**Operations**
- [ ] **A live URL with a demo login** ← the single highest-impact item
- [ ] A Grafana dashboard screenshot
- [ ] A distributed trace screenshot
- [ ] Zero-downtime deploy demonstrated (requests-failed: 0)
- [ ] External dead-man's-switch monitoring — *because Vigil can't monitor Vigil*

**Honesty**
- [ ] A "Known limitations and what I'd do next" section
- [ ] The microservices retrospective admitting which split wasn't worth it

**That last category is the differentiator.** A project that acknowledges its own limitations reads as engineering; one that claims to be perfect reads as marketing. **Interviewers trust the former.**

---

## 9. Deliberate scope limits — say these out loud

**Being clear about what you did *not* build is a strength, not an admission.** It shows you scoped deliberately rather than ran out of time.

| Not built | Why, and what I'd do |
|---|---|
| A polished SPA dashboard | Backend project. Server-rendered status page + OpenAPI + a minimal dashboard. **Frontend isn't what this demonstrates.** |
| Real payment processing | Plans and quotas are modelled; Stripe integration is a known, well-documented pattern with no new learning here |
| Multi-region checking | Architecturally supported (`region` on the worker); one region deployed for cost |
| SMS/phone escalation | Twilio is another notification channel — the extension point exists, adding it is mechanical |
| ML anomaly detection | Genuinely interesting, entirely out of scope |
| A full audit/history UI | Audit events are written; there's no UI for browsing them |

**Also worth stating: known limitations.**
- Check scheduling drifts by up to the dispatch interval (5s) — acceptable for uptime monitoring, unacceptable for anything financial
- The status page cache means up to 30s of staleness — a deliberate choice, and the right one during an incident
- No cross-region consistency story; single-region Postgres is the availability ceiling

**"Here's what I'd fix first if this had users" is one of the best answers you can give**, because it demonstrates you evaluate your own work the way a senior engineer evaluates a system.

---

## 10. The one-paragraph version

Memorize the substance of this. You'll use it constantly.

> *"Vigil is a self-hostable uptime monitoring and incident management platform — multi-tenant, deployed, with a demo login. Technically the interesting parts are: horizontal scaling of check workers using `SELECT FOR UPDATE SKIP LOCKED` so every instance claims a disjoint batch with no coordinator or broker; a transactional outbox so a state change and its notification are atomic; four layers of tenant isolation ending in Postgres row-level security; and resilience throughout — timeouts, circuit breakers per target, and bulkheads — because the product's entire job is calling unreliable third-party endpoints. It's deliberately a modular monolith with one justified extraction, documented in an ADR, and I have a retrospective explaining which part of that split wasn't worth it."*

---

**Next:** [Lesson 42 — Vigil build log](42-vigil-build-log.md) — phase-by-phase instructions with acceptance criteria, so you always know what "done" means.

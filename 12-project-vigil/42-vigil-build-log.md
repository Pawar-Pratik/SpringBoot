# Lesson 42 — Vigil build log

> **Why this lesson exists:** side projects die in the middle, not at the start, and the build order is what decides whether you keep going. These fourteen phases are sequenced so something runnable exists at the end of every one, so each phase uses only lessons you have already read, and so "done" is an acceptance criterion rather than a feeling.

> **This is your project plan.** Fourteen phases, each independently shippable, each with **acceptance criteria** so you always know what "done" means. Follow it in order — every phase depends on the previous one, and every phase maps to lessons you've already read.
>
> **The rule: do not start a phase until the previous one meets its acceptance criteria.** Half-finished features are how side projects die.

**Time:** 14 phases · roughly 70–100 hours total · **Prereq:** Lesson 41

---

## How to use this

**Each phase has:**
- **Build** — what to write
- **Prove it** — a demonstration you can run, so "done" is objective
- **Commit** — a meaningful commit message (your git history is part of the portfolio)
- **Lessons** — where the material is

**Track it.** Copy this checklist into `PROGRESS.md` and tick as you go:
```
[ ] P0  skeleton        [ ] P5  check engine    [ ] P10 observability
[ ] P1  monitor API     [ ] P6  incidents       [ ] P11 DEPLOY ⭐
[ ] P2  persistence     [ ] P7  status pages    [ ] P12 polish
[ ] P3  auth            [ ] P8  reports         [ ] P13 extras (optional)
[ ] P4  authorization   [ ] P9  testing
```

**Two rules that matter more than the code:**

1. **Commit small and often, with real messages.** An interviewer who opens your repo sees the history. `fix stuff` × 200 tells a story; so does `feat(check): claim due monitors with SKIP LOCKED for horizontal scaling`.
2. **Write the README as you go**, not at the end. Add a section per phase. The README is the *only* file most people will read.

---

## Phase 0 — Skeleton

**Build**
- `start.spring.io`: Maven, Java 21, Boot 3.5.x. Dependencies: Web, Actuator, Validation, Lombok
- Package structure from [Lesson 13](../04-spring-boot/13-project-structure.md) §9 (empty packages with `package-info.java`)
- `compose.yaml` with Postgres 16 + Redis, health-gated
- `application.yml` + `application-local.yml` (gitignored) + `application-local.yml.example` (committed)
- `.gitignore`, `.dockerignore`, `README.md` skeleton, `LICENSE` (Apache 2.0)
- Actuator on port 9090, `health,info` exposed

**Prove it**
```bash
docker compose up -d
./mvnw spring-boot:run
curl localhost:9090/actuator/health          # {"status":"UP"}
```
- [ ] `docker compose up` starts Postgres and Redis, both healthy
- [ ] App starts and `/actuator/health` returns UP
- [ ] `git status` shows no secrets and no `target/`

**Commit:** `chore: project skeleton with compose stack and actuator`
**Lessons:** 01, 11, 12, 13

---

## Phase 1 — Monitor API (in-memory)

**No database yet.** Get the HTTP layer right first, in isolation.

**Build**
- `CreateMonitorRequest`, `UpdateMonitorRequest`, `MonitorResponse`, `MonitorSummaryResponse`, `PagedResponse<T>` — records, in `monitor/dto`
- Bean Validation on every input, including a custom `@SafeUrl` validator with **SSRF checks**
- `MonitorController`: full CRUD + `/pause`, correct status codes, `Location` on create
- `MonitorService` backed by a `ConcurrentHashMap`
- `GlobalExceptionHandler` returning `ProblemDetail` for validation, 404, 409, 415, 406, and a catch-all with an `errorId`
- `CorrelationIdFilter` (`OncePerRequestFilter`, MDC, `finally { MDC.clear(); }`)
- springdoc OpenAPI with `@Operation`, `@ApiResponses`, and `example` values on every field
- ETag on `GET /monitors/{id}`

**Prove it**
```bash
curl -i -X POST localhost:8080/api/v1/monitors -H 'Content-Type: application/json' \
  -d '{"name":"","url":"http://localhost:9090/actuator/env","intervalSeconds":5,"type":"HTTP"}'
```
- [ ] Returns 400 with a `ProblemDetail` listing **three** field errors (blank name, unsafe URL, interval too small)
- [ ] Every error response has the same shape and a `correlationId`
- [ ] No stack trace in any response body
- [ ] `Content-Type: text/plain` → 415; `Accept: application/xml` → 406
- [ ] Swagger UI "Try it out" successfully creates a monitor
- [ ] A 404 request still gets a correlation ID (proves the filter, not an interceptor)
- [ ] `curl -H 'If-None-Match: "<etag>"'` returns 304

**Commit:** `feat(monitor): REST API with validation, problem details and OpenAPI`
**Lessons:** 14, 15, 16, 17

---

## Phase 2 — Persistence

**Build**
- Flyway migrations `V1__…V5__` from [Lesson 41](41-vigil-blueprint.md) §3: organization, app_user, membership, monitor, check_result (partitioned)
- `ddl-auto: validate`, `open-in-view: false`, SQL logging on, `default_batch_fetch_size: 25`
- Entities: `BaseEntity` (`@MappedSuperclass`, audit fields, `@Version`), `Organization`, `AppUser`, `Membership` (join **entity**), `Monitor`
- Every to-one `LAZY`; every enum `EnumType.STRING`; correct `equals`/`hashCode`; `Monitor.config` as `@JdbcTypeCode(SqlTypes.JSON)`
- `MonitorRepository` with `findByIdAndOrgId`, projections, `@EntityGraph`
- `CheckResultRepository extends Repository<...>` (**not** `JpaRepository`) with `JdbcClient` for batched writes
- MapStruct mapper with `unmappedTargetPolicy = ERROR`
- Replace the in-memory store; `@Transactional(readOnly = true)` at class level on services

**Prove it**
- [ ] Drop the database, start the app, Flyway builds everything; `flyway_schema_history` is populated
- [ ] Editing an applied migration fails validation on restart
- [ ] Adding an unmapped entity field fails startup (`ddl-auto: validate` working)
- [ ] `SELECT config->>'expectedStatus' FROM monitor` returns real JSON
- [ ] A projection query's SQL selects **only** the projected columns (check the log)
- [ ] 10,000 batched check-result inserts complete in **under 2 seconds**
- [ ] A `@DataJpaTest` with `em.flush(); em.clear();` proves the tenant-scoped query excludes other orgs

**Commit:** `feat(persistence): JPA entities, Flyway migrations and JdbcClient result writer`
**Lessons:** 19, 20, 21, 22, 23, 24

---

## Phase 3 — Authentication

**Build**
- `VigilUserDetails` (wrapping the entity, **not** implementing it on the entity), `VigilUserDetailsService`
- Delegating `PasswordEncoder`
- RSA keypair from env/file; `JwtEncoder`/`JwtDecoder` via a `JWKSource`
- `TokenService`: 15-min access JWT (`iss`, `aud`, `sub`, `jti`, `org`, `roles`, `tv`) + opaque refresh token, **SHA-256 hashed at rest**, with a `family_id`
- `refresh_token` migration; rotation with **reuse detection** (revoke the family)
- `SecurityConfig` with `oauth2ResourceServer`, `STATELESS`, CSRF disabled (documented why), `.cors()`, deny-by-default
- `JwtAuthenticationConverter` with `setAuthorityPrefix("")`
- Custom `AuthenticationEntryPoint` / `AccessDeniedHandler` returning `ProblemDetail`
- `TokenVersionValidator` against a cached per-user version
- Auth endpoints: register, login, refresh, logout, me

**Prove it**
- [ ] Tamper with a token's payload → 401
- [ ] Expired token → 401 with `WWW-Authenticate`
- [ ] Token with wrong `aud` → 401
- [ ] Present an already-used refresh token → **entire family revoked**, audit logged
- [ ] Bump `tokenVersion` → an unexpired access token stops working immediately
- [ ] `SELECT * FROM refresh_token` shows **only hashes**, never raw tokens
- [ ] Swagger's **Authorize** button works end to end

**Commit:** `feat(auth): JWT with refresh rotation, reuse detection and token versioning`
**Lessons:** 25, 26

---

## Phase 4 — Authorization & tenant isolation

**Build**
- `OrgRole` enum with permission sets; `hasPermission(orgId, permission)`
- `@EnableMethodSecurity`; `@PreAuthorize` checking **permissions**, not roles
- `OrgGuard` bean used as `@orgGuard.hasPermission(#orgId, 'monitors:write')`
- Restructure paths to `/api/v1/orgs/{orgId}/monitors/...`, with `orgId` **verified** against memberships
- `TenantContext` (ThreadLocal, cleared in `finally`) + a filter
- Every query tenant-scoped; a `Specification` with an unconditional tenant predicate
- Postgres RLS on `monitor` and `check_result`, with `SET LOCAL app.current_org` per transaction
- Actuator locked down: separate port, allowlist, `show-details: when-authorized`

**Prove it** — this phase is *defined* by its tests
- [ ] Cross-tenant GET/PUT/DELETE all return **404** (not 403)
- [ ] A list endpoint never returns another tenant's rows
- [ ] Remove the app-level tenant filter → **RLS still blocks it**
- [ ] Remove `TenantContext.clear()` → tenants bleed under concurrent load (then restore it)
- [ ] `monitors:read` cannot DELETE → 403
- [ ] `/actuator/env` requires authentication
- [ ] No response body anywhere contains `passwordHash`

**Commit:** `feat(security): permission model, tenant isolation and row-level security`
**Lessons:** 28

---

## Phase 5 — The check engine ⭐

**This is the heart of the project.** Take your time.

**Build**
- `CheckExecutor` interface with `supports(MonitorType)`; `HttpCheckExecutor`, `TcpCheckExecutor`, `SslCheckExecutor`, `HeartbeatCheckExecutor`
- `CheckDispatcher` injecting `List<CheckExecutor>` (self-selecting strategy)
- `CheckQueueRepository.claimBatch` with **`FOR UPDATE SKIP LOCKED`**
- `CheckScheduler` (`@Scheduled(fixedDelay = 5000)`, **try/catch inside**, no ShedLock)
- `ThreadPoolTaskScheduler` (pool 5, named threads, `ErrorHandler`); virtual-thread `checkExecutor`
- `RestClient` with connect+read timeouts; per-target circuit breaker; SSRF check **at connection time**; redirects validated
- Batched result writes with `ON CONFLICT DO NOTHING`
- `sealed interface CheckOutcome permits Up, Down, Timeout`
- ShedLock for the singleton jobs (purge, daily report)

**Prove it**
- [ ] Run **two instances** against one database: each monitor checked exactly once; work split roughly evenly (`SELECT claimed_by, count(*) FROM monitor GROUP BY claimed_by`)
- [ ] Remove `SKIP LOCKED` → the second worker blocks. Remove `FOR UPDATE` → both check the same monitors
- [ ] A monitor pointed at a 60s-hanging endpoint times out in ~10s (**test this**)
- [ ] A monitor pointed at `http://169.254.169.254/` is rejected
- [ ] A monitor whose target 302s to `localhost` is rejected
- [ ] A persistently failing target opens its circuit; **other monitors are unaffected**
- [ ] With two instances, the purge job logs from only one (ShedLock)
- [ ] Throw inside a `@Scheduled` method → confirm it's cancelled forever (then fix with try/catch)

**Commit:** `feat(check): horizontally-scalable check engine with SKIP LOCKED claiming`
**Lessons:** 32, 34, 18

---

## Phase 6 — Incidents & notifications

**Build**
- `Incident` entity + `incident_event` timeline; partial unique index enforcing one open incident per monitor
- `IncidentService` state machine: open after *n* consecutive failures, acknowledge, resolve, reopen within the flap window
- Domain events: `MonitorWentDownEvent`, `MonitorRecoveredEvent`, `IncidentOpenedEvent`
- `IncidentListener` (plain `@EventListener`, in-transaction — atomic with the status change)
- `outbox_message` table + write inside the business transaction + `OutboxRelay` polling with `SKIP LOCKED`
- `NotificationChannel` interface; `SlackChannel` (HTTP interface `@PostExchange`), `EmailChannel`, `WebhookChannel`
- `NotificationDispatcher` injecting `List<NotificationChannel>`
- `@Retryable` with backoff **+ jitter**, `noRetryFor` 4xx, `@Recover` → dead-letter table
- Bounded `notificationExecutor` with `CallerRunsPolicy` and a `ContextPropagatingTaskDecorator`
- Escalation policy (`jsonb` steps) with a scheduled escalation check

**Prove it**
- [ ] Two consecutive failures open exactly **one** incident; a third doesn't create another
- [ ] Recovery resolves it and records MTTR
- [ ] Flap within the window **reopens** the same incident
- [ ] `@TransactionalEventListener(AFTER_COMMIT)`: roll back after publishing → **no notification fires**. With plain `@EventListener` → it does (log both)
- [ ] `kill -9` between commit and relay → the message publishes on restart
- [ ] A 400 from Slack is **not** retried; a 503 is retried 3× with non-uniform delays
- [ ] Correlation ID appears in logs from inside the `@Async` notification method

**Commit:** `feat(incident): state machine with transactional outbox and resilient notification`
**Lessons:** 32, 34, 10

---

## Phase 7 — Status pages & caching

**Build**
- `StatusPage` entity (unique slug) + a join entity for monitors
- `@Controller` + Thymeleaf `status-page.html` (`th:text` only, never `th:utext`)
- `GET /api/v1/public/status/{slug}` JSON variant with ETag + `Cache-Control`
- Redis `CacheConfig`: per-cache TTLs, JSON serialization, `vigil:v1:` prefix, `transactionAware()`, `CacheErrorHandler`
- **Stale-while-revalidate** for the status page
- Redis-backed rate limiting for public routes (`X-RateLimit-*` headers, `Retry-After` on 429)
- A separate `SecurityFilterChain` for public routes, ordered correctly

**Prove it**
- [ ] Status page loads and renders correctly for an anonymous user
- [ ] `redis-cli KEYS vigil:v1:*` shows readable JSON with a TTL
- [ ] Second request is served from cache (measure with `curl -w "%{time_total}"`)
- [ ] `docker stop redis` → **the page still renders**, `/actuator/health` shows DEGRADED not DOWN
- [ ] 50 concurrent requests just after expiry: without `sync = true`, ~50 computations; with it, 1
- [ ] After logical expiry, the response is still fast and a background refresh happens
- [ ] Exceeding the rate limit returns 429 with `Retry-After`; two app instances share the limit
- [ ] Cache key includes the slug — no cross-tenant leakage (**write this test**)

**Commit:** `feat(statuspage): public status pages with Redis caching and rate limiting`
**Lessons:** 15, 17, 33

---

## Phase 8 — Reports

**Build**
- `JdbcClient` aggregations: uptime %, p95/p99 latency (`percentile_cont`), MTTR, incident count, SLA compliance
- Cursor pagination for `/results` (base64 `(checked_at, id)` cursor)
- `uptime_summary` table maintained incrementally (a CQRS-style read model), refreshed on check results
- A monthly-partition creation job + retention via `DROP PARTITION`

**Prove it**
- [ ] Uptime % matches a hand-calculated figure on seeded data
- [ ] p95 is correct for both odd- and even-length datasets
- [ ] Cursor pagination is stable while rows are being inserted concurrently
- [ ] `EXPLAIN ANALYZE` on the results query shows an **Index Only Scan**
- [ ] A dashboard request that previously made N cross-queries is now **one** local query
- [ ] Retention drops an old partition instantly, not via a slow `DELETE`

**Commit:** `feat(report): uptime, latency percentiles and SLA reporting`
**Lessons:** 19, 22, 24

---

## Phase 9 — Testing

**Build**
- `AbstractIntegrationTest`: singleton Testcontainers Postgres + Redis, `DatabaseCleaner`
- Unit tests: `UptimeCalculator` (parameterized, exhaustive), incident state machine (`@Nested` per state)
- `TestData` builders for every entity
- `Clock` bean injected everywhere; `Clock.fixed` in tests
- `@DataJpaTest`: tenant scoping, unique constraints, `SKIP LOCKED`, N+1 query-count assertions
- `@WebMvcTest`: success, 400 shape, 404, 401, 403, `passwordHash doesNotExist`
- Security suite: cross-tenant 404 for **every** endpoint
- WireMock: 200/500/timeout/redirect-to-metadata/connection-refused
- ArchUnit: 8+ rules including `transactionalMethodsMustBePublic` and `noEntitiesFromControllers`
- OpenAPI snapshot test
- Surefire/Failsafe split (`*Test` vs `*IT`); parallel execution enabled
- The CI pipeline from [Lesson 31](../08-testing/31-advanced-testing-strategy.md) §7

**Prove it**
- [ ] `./mvnw test` completes in **under 30 seconds**
- [ ] `./mvnw verify` runs integration tests against real Postgres
- [ ] Violating any ArchUnit rule fails with a precise message
- [ ] Renaming a DTO field fails the OpenAPI snapshot test with a readable diff
- [ ] The N+1 test fails when you reintroduce a lazy call in a mapper
- [ ] The timeout test fails if you set the read timeout to 0
- [ ] Zero `Thread.sleep` in the test suite (all Awaitility)
- [ ] PIT mutation score on the domain package ≥ 70%
- [ ] Tests pass in parallel and in random order

**Commit:** `test: full suite with Testcontainers, ArchUnit, WireMock and contract snapshot`
**Lessons:** 29, 30, 31

---

## Phase 10 — Observability

**Build**
- Liveness = `ping` only; readiness = `db` (**not** Redis)
- `startupProbe`-compatible config
- `CheckWorkerHealthIndicator` (backlog + last-dispatch age), Redis indicator reporting DEGRADED
- Micrometer + Prometheus registry; business metrics: `vigil.checks.executed`, `vigil.check.duration`, `vigil.checks.overdue`, `vigil.incidents.open`, `vigil.outbox.pending`, `vigil.dead_letters.total`
- `percentiles-histogram: true`; SLO buckets
- Structured JSON logging with `correlationId` + `traceId`
- OTel tracing to Tempo; `@Observed` on `CheckExecutor.execute`
- `docker-compose.observability.yml`: Prometheus, Grafana, Loki, Tempo
- A Grafana dashboard: request rate, error rate, p50/95/99, checks overdue, open incidents, Hikari pool, circuit-breaker state
- Prometheus alert rules: `ChecksBacklogGrowing`, `OutboxStalled`, `HighErrorRate`
- An external dead-man's-switch heartbeat

**Prove it**
- [ ] Put `db` in **liveness**, stop Postgres, and write down the Kubernetes consequence. Then fix it
- [ ] `docker stop redis` does **not** make the app unready
- [ ] Adding `monitorId` as a metric tag explodes `/actuator/prometheus` line count (then remove it)
- [ ] A single trace spans controller → DB → outbox → notification
- [ ] `@Observed` yields both a metric **and** a span
- [ ] Grep logs by correlation ID and reconstruct a full request
- [ ] Stop the dispatcher → `ChecksBacklogGrowing` fires in Alertmanager
- [ ] Stop the app → the external heartbeat alerts you

**Commit:** `feat(observability): metrics, structured logs, tracing and alerting`
**Lessons:** 35

---

## Phase 11 — DEPLOY ⭐

**Do this now, not later.** Everything after this is polish on a live system.

**Build**
- Multi-stage Dockerfile with **layer extraction**, non-root user, `MaxRAMPercentage`, healthcheck
- Full `compose.yaml` bringing up the entire platform
- CI/CD: test → build → Trivy scan → push by digest → deploy → smoke test
- `server.shutdown=graceful` + `timeout-per-shutdown-phase`
- Deploy to Fly.io / Railway / Hetzner
- Custom domain + HTTPS
- Seeded demo data + a public demo login
- A real status page monitoring real public endpoints
- Automated backups, **restore verified**
- `/actuator/info` exposing the git commit

**Prove it**
- [ ] `git clone && cp .env.example .env && docker compose up` works from scratch in **under 5 minutes**
- [ ] Layer extraction: a one-line code change pushes ~400KB, not 60MB (`docker history`)
- [ ] `docker run --rm vigil whoami` is not root
- [ ] Heap size changes with `-m 512m` (proves `MaxRAMPercentage`)
- [ ] Trivy reports no CRITICAL/HIGH CVEs
- [ ] **Zero-downtime deploy: `hey -z 120s` during a deploy → 0 failed requests**
- [ ] The live URL works with HTTPS and a valid certificate
- [ ] Demo login works; demo data is present
- [ ] Actuator is **not** reachable from the public internet
- [ ] A backup restores into a fresh database with data intact
- [ ] Rollback by digest works, and you know how long it takes

**Commit:** `ci: containerized build, deployment pipeline and zero-downtime rollout`
**Lessons:** 40

**🎉 At this point Vigil is a real, live product. Put the URL on your resume.**

---

## Phase 12 — Polish

**Build**
- Social login: Google + GitHub, `FederatedIdentity` keyed on `(provider, sub)`, `email_verified` enforced
- Organization invitations (single-use, expiring tokens)
- API keys for programmatic access (hashed, scoped, revocable)
- Outbound webhooks with HMAC signatures
- `Idempotency-Key` support on POST
- `If-Match` → 409 on `PUT`
- Response compression
- `ARCHITECTURE.md`, `docs/adr/`, a proper `README.md` with screenshots
- k6 load-test scripts committed + a results table
- The performance before/after table

**Prove it**
- [ ] Google login creates a user, links the federated identity, and a second login reuses it
- [ ] Change the user's email in the DB → login still finds the correct user (keyed on `sub`)
- [ ] Unverified email → login rejected (with the test explaining the takeover it prevents)
- [ ] Same idempotency key twice → one monitor, identical responses; different body → 422
- [ ] Stale `If-Match` → 409
- [ ] A webhook receiver can verify the HMAC signature
- [ ] README has: what/why, a live demo link, screenshots, architecture, how to run, and known limitations

**Commit:** `feat: social login, API keys, webhooks and documentation`
**Lessons:** 27, 34, 13

---

## Phase 13 — Optional extras

**Only if you want the distributed-systems material on your CV.** The monolith is already a strong project.

**Build**
- Kafka via KRaft; topics `check.results` (12 partitions, keyed by monitorId), `incident.events`, `notification.requests`, plus `.DLT`s
- Producer: `acks=all`, idempotence, snappy, `linger.ms=10`
- Consumers: `enable-auto-commit=false`, `MANUAL_IMMEDIATE`, `DefaultErrorHandler` + DLT, non-retryable classification, idempotent via `ON CONFLICT`
- Extract `vigil-checker` with its own database — **write the ADR first**
- `vigil-api` consumes `check.results` into its `uptime_summary` read model
- `vigil-gateway` with JWT at the edge, header stripping, per-route rate limits and bulkheads
- Contract tests between services
- `MICROSERVICES-RETROSPECTIVE.md`

**Prove it**
- [ ] Same-key messages land in one partition; different keys spread across all
- [ ] Auto-commit on → kill mid-batch → records silently skipped. Off → redelivered
- [ ] A poison message stalls a partition; with a DLT it doesn't
- [ ] Publish the same event 5× → exactly one row
- [ ] `docker compose up --scale vigil-checker=5` → ~5× backlog throughput, API latency unchanged
- [ ] Header injection (`X-User-Id`) is blocked by stripping **and** by service-side JWT validation
- [ ] A gateway retry on POST creates duplicates; restricted to GET, it doesn't
- [ ] One trace spans gateway → api → kafka → checker → kafka → notifier
- [ ] A slow checker doesn't affect `/api/v1/monitors` (bulkhead)

**Commit:** `feat: kafka event pipeline and checker service extraction`
**Lessons:** 37, 38, 39

---

## The demo script

**Rehearse this.** When someone says "show me your project," you have 5 minutes and you should not be improvising.

```
0:00  "Vigil is a self-hostable uptime and incident management platform.
       It's live at vigil.example.dev — here's the demo login."

0:30  Show the public status page. "This is what a customer's users see.
       It's server-rendered so it works even if our JS CDN is down —
       which matters, because it's a status page."

1:00  Add a monitor pointing at a URL you'll break. Show it go UP.

1:30  Break the target (stop a container). Watch: check fails → incident opens
       → notification delivered. "Two consecutive failures before opening,
       so a single blip doesn't page anyone."

2:30  Show the Grafana dashboard: checks/sec, p95 latency, checks overdue.
       "checks.overdue is the metric that matters — everything else can look
       healthy while the product silently isn't working."

3:00  Show a distributed trace. "Here's one check end to end — you can see
       exactly where the time went."

3:30  Kill Redis live. "The status page still works, just slower.
       Health reports DEGRADED, not DOWN, so we stay in the load balancer."

4:00  Show the ADR for the checker extraction, and the retrospective section
       admitting the notification split wasn't worth it.

4:30  "The technically interesting part is horizontal scaling: every worker
       claims a disjoint batch with SELECT FOR UPDATE SKIP LOCKED, so scaling
       is just adding replicas — no broker, no leader election."
```

**The two moments that land hardest** are killing Redis live (visible graceful degradation) and the retrospective (visible judgment). **Practise both.**

---

## Common failure modes of this project

**Being honest about how side projects die, so yours doesn't:**

| Trap | Fix |
|---|---|
| **Endless refactoring, never shipping** | Phase 11 (deploy) is not optional and not last. Ship at Phase 11 even if Phases 5–8 are rough |
| **Gold-plating the frontend** | This is a backend project. A minimal dashboard is fine. Say so in the README |
| **Adding technologies for the CV** | Every dependency must have a justification in the README. If you can't write one, remove it |
| **Skipping tests "for now"** | Phase 9's acceptance criteria are how the project stays changeable. Skipping them means Phase 12 breaks Phase 5 |
| **No commits for three weeks, then one huge one** | Commit per feature. The history is part of what's assessed |
| **Perfect code, no README** | Most reviewers read only the README and the live demo. Write it as you go |
| **Building Phase 13 before Phase 11** | Microservices on an undeployed monolith is the clearest possible signal of misplaced priorities |

---

**Next:** [Lesson 43 — Polish, host & present it](43-polish-host-and-present.md) — turning the finished project into resume and interview leverage.

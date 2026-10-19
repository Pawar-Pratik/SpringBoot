# Lesson 38 — Microservices fundamentals

> **Why this lesson exists:** Microservices are the most over-applied architecture in software. The valuable skill isn't building them — it's knowing *whether* to, *where* the boundaries go, and how to handle the problems you create by distributing your data. This lesson gives you a defensible position, which is far more impressive in an interview than enthusiasm.

**Time:** ~75 minutes · **Prereq:** Lessons 13, 34, 37

---

## 1. The idea in one sentence

> **Microservices solve an *organizational* problem — letting separate teams deploy separately — and you pay for that with everything a monolith gave you free: one transaction, one stack trace, one deployment, immediate consistency.**

Which makes the whole lesson a question of evidence. Extract a service when you can name the scaling profile, the team, or the isolation requirement that demands it; otherwise a modular monolith buys you the same design discipline at none of the distributed-systems cost.

---

## 2. The honest case against microservices — start here

**Almost every microservices interview question is really testing whether you understand the costs.** A candidate who says "microservices are more scalable and maintainable" has read marketing. A candidate who can list what you *lose* has done the work.

### What a monolith gives you for free, and a microservice takes away

| | Monolith | Microservices |
|---|---|---|
| A cross-entity change | **One ACID transaction** | A saga with compensation (§5) |
| Calling another module | A method call: fast, typed, reliable | A network call: slow, untyped at runtime, can fail |
| Refactoring across a boundary | The compiler finds every caller | **Coordinated deploys across teams** |
| Debugging a request | One stack trace | Distributed tracing, and hope you have it |
| A local dev environment | Run one app | Run 12 services, or mock them |
| Testing an interaction | An integration test | Contract tests + a shared environment |
| A query joining two areas | `JOIN` | Two API calls and an in-memory join |
| Consistency | Immediate | **Eventual** — and your users notice |

**And what you take on:** service discovery, an API gateway, distributed tracing, per-service CI/CD, per-service databases and migrations, network security between services, versioning, and an on-call rotation that spans them.

### The two things microservices genuinely buy you

**1. Independent deployability.** Team A ships without coordinating with team B. **This is the real reason**, and it's an *organizational* benefit, not a technical one.

**2. Independent scalability.** Scale the component that's actually the bottleneck instead of the whole application.

> **Conway's Law:** *"Organizations design systems that mirror their own communication structure."* Microservices are, at heart, a way to let separate teams own separate deployables. **Which is why they don't help a team of four** — you've paid the full distributed-systems cost to solve a coordination problem you don't have.

### The answer to give

> *"I'd start with a modular monolith: feature packages with enforced boundaries, no cross-feature shared tables, and communication through public service interfaces or domain events. That gives me most of the design discipline of microservices with none of the distributed-systems cost. I'd extract a service when there's a concrete reason — a component with genuinely different scaling characteristics, a team that needs to deploy independently, or an isolation requirement — and I'd extract it along the module boundary I already have. Microservices are a solution to an organizational scaling problem; applying them to a small team is paying a large tax for a benefit you can't collect."*

**That paragraph is worth memorizing in substance**, because it's the single most probable microservices question and the answer separates a mid-level from a senior candidate.

### Real signals you should extract a service

| Signal | Example |
|---|---|
| **Genuinely different scaling profile** | Vigil's checkers need 50 replicas; the API needs 3 |
| **Different resource shape** | A CPU-bound PDF renderer next to an I/O-bound API |
| **Independent deploy cadence** | A team shipping 10×/day blocked by a monthly release train |
| **Isolation requirement** | PCI/PII scope reduction, or a noisy-neighbour blast radius |
| **Different technology genuinely needed** | An ML model served in Python |
| **A distinct availability requirement** | A status page that must stay up when everything else is down |

**Bad reasons:** "microservices are modern," "the monolith is messy" (fix the monolith — distribution won't make bad boundaries good, it will make them *expensive*), "we might need to scale someday," "it'll look good on my CV."

**The sharpest version of that point:** *"If you can't draw clean module boundaries inside one codebase, you certainly can't draw them across a network — you'll just have converted method calls into unreliable network calls and kept all the coupling."* That is a genuinely memorable thing to say.

---

## 3. Where the boundaries go

### The one rule: a service owns its data

```
❌ THE #1 MISTAKE — a shared database
   ┌─────────┐  ┌─────────┐  ┌─────────┐
   │ service │  │ service │  │ service │
   │    A    │  │    B    │  │    C    │
   └────┬────┘  └────┬────┘  └────┬────┘
        └───────────┬┴────────────┘
                ┌───▼────┐
                │   DB   │   ← A schema change breaks three services.
                └────────┘      You have a distributed monolith:
                                all the cost, none of the benefit.
```

```
✅ Each service owns its schema; others access it only through its API or its events
   ┌─────────┐        ┌─────────┐        ┌─────────┐
   │    A    │◀─API──▶│    B    │◀─API──▶│    C    │
   └────┬────┘        └────┬────┘        └────┬────┘
     ┌──▼──┐           ┌──▼──┐           ┌──▼──┐
     │ DB  │           │ DB  │           │ DB  │
     └─────┘           └─────┘           └─────┘
```

**A shared database is the definitive sign of a distributed monolith.** You've added network latency, partial failure, and deployment complexity while keeping every coupling you had — because the schema is still a shared, breakable contract between all of them.

### Finding boundaries with DDD-lite

**Bounded context** — an area where a term has one consistent meaning. In Vigil, "monitor" means a configuration object to the API and an executable job to the checker. **That difference in meaning is a boundary hint.**

**Aggregate** — a cluster of objects with one root, treated as a single consistency unit. **Rule: one transaction touches one aggregate.** This is the crucial one, because it's the test that survives the transition to services: if two things must change atomically, they belong in the same aggregate — and therefore the same service.

**The heuristics that actually work:**

| Heuristic | Application |
|---|---|
| **Transactional boundaries** | Things that must change atomically belong together. **The strongest signal.** |
| **Rate of change** | Code that changes together belongs together |
| **Team ownership** | One service, one owning team (never the reverse) |
| **Data access patterns** | Vigil: monitor *config* is read-heavy and rarely written; check *results* are write-heavy and append-only — different shapes, different services |
| **Failure isolation** | What must keep working when the rest doesn't? |

**Anti-heuristics:**
- ❌ **By technical layer.** A "database service," a "business logic service" and a "UI service" is a distributed n-tier app — every feature needs a change in all three. Worst possible split.
- ❌ **By entity.** A `MonitorService`, `OrganizationService` and `UserService` with one table each guarantees chatty cross-service calls for every operation.
- ❌ **Nano-services.** A service per endpoint. Overhead swamps any benefit.

### Vigil's split, and why each boundary is defensible

```
                      ┌──────────────┐
                      │ API Gateway  │   auth at the edge, routing, rate limiting
                      └──────┬───────┘
        ┌────────────────────┼────────────────────┐
   ┌────▼─────┐        ┌─────▼──────┐       ┌────▼──────┐
   │ vigil-api│        │vigil-checker│       │vigil-notif│
   │  3 pods  │        │  50 pods    │       │  3 pods   │
   │          │        │             │       │           │
   │ CRUD,    │        │ executes    │       │ Slack,    │
   │ orgs,    │        │ HTTP/TCP/   │       │ email,    │
   │ auth,    │        │ SSL checks  │       │ webhooks  │
   │ reports  │        │             │       │           │
   └────┬─────┘        └──────┬──────┘       └─────┬─────┘
     ┌──▼───┐             ┌───▼───┐            ┌───▼────┐
     │Postgres│           │Postgres│           │Postgres│
     │ (config│           │(results│           │(delivery│
     │ +orgs) │           │ tsdb)  │           │ log)   │
     └────────┘           └────────┘           └────────┘
                    │            │            │
                    └────── Kafka ────────────┘
```

**Justify each boundary — this is what "defend your architecture" means:**

- **checker is separate because its scaling profile is genuinely different.** 50,000 monitors checked every 60 seconds is ~800 checks/second of pure outbound I/O. The API handles maybe 50 requests/second. **Scaling one shouldn't scale the other**, and a slow customer endpoint must never consume API capacity. This is the strongest boundary in the system.
- **checker's database is separate because the workload is different.** Append-only, high write volume, time-series shaped, aggressively partitioned and pruned. Config data is small, read-heavy, and relational. **Different tuning, different backup policy, different growth curve.**
- **notifier is separate because it talks to unreliable third parties.** A Slack outage must not affect check execution. It needs its own retry state, rate limits, and circuit breakers, and it must be independently scalable during an incident storm.
- **The status page could be separate** — because it must stay up when everything else is down. Arguably it should be a statically-generated page on a CDN. (Genuinely interesting design question, and a good one to raise unprompted.)

**And equally important: state what you did NOT split.** Organizations, users, memberships, monitors, incidents and reports all stay in `vigil-api`, because they're a single consistency domain — creating a monitor checks the org's plan quota, and that must be transactional. **Knowing what to keep together is the harder skill.**

---

## 4. Communication patterns

### Synchronous vs asynchronous

```java
// SYNCHRONOUS — the caller waits and needs the answer
CheckResult result = checkerClient.executeNow(monitorId);      // a user clicked "check now"

// ASYNCHRONOUS — fire an event, don't wait
kafkaTemplate.send("check.results", monitorId, result);         // routine result recording
```

| | Sync (HTTP/gRPC) | Async (Kafka/queue) |
|---|---|---|
| Coupling | **Temporal** — both must be up | Only the broker must be up |
| Latency | Immediate result | Eventual |
| Failure | Propagates to the caller | Absorbed by the queue |
| Use for | Queries; the caller needs the answer | Commands/events; fire and continue |

**The rule: sync for queries, async for state propagation.** And avoid **chained synchronous calls** — A→B→C→D means the availability is `0.99⁴ = 96%` and the latency is the sum. **Every synchronous hop multiplies your failure probability**, which is the arithmetic argument for async.

### Declarative HTTP clients (Lesson 18)

```java
public interface CheckerClient {
    @PostExchange("/internal/checks/{monitorId}/execute")
    CheckResultResponse executeNow(@PathVariable UUID monitorId);
}
```
```java
@Bean
CheckerClient checkerClient(RestClient.Builder builder) {
    RestClient client = builder
            .baseUrl("http://vigil-checker")           // ★ resolved by service discovery
            .requestFactory(ClientHttpRequestFactories.get(
                    ClientHttpRequestFactorySettings.DEFAULTS
                            .withConnectTimeout(Duration.ofSeconds(2))     // ★ mandatory
                            .withReadTimeout(Duration.ofSeconds(5))))
            .build();
    return HttpServiceProxyFactory.builderFor(RestClientAdapter.create(client))
            .build().createClient(CheckerClient.class);
}
```

**Spring's HTTP Interfaces replace Feign** in modern Spring (Lesson 18 §6), and every inter-service client needs timeouts, a circuit breaker and a bulkhead (Lesson 34). **A service-to-service call without a timeout is how one service's slowness becomes a cluster-wide outage.**

### gRPC — know when it wins

| | REST/JSON | gRPC |
|---|---|---|
| Payload | Text, human-readable | Binary Protobuf, ~3–10× smaller |
| Contract | OpenAPI (often after the fact) | **`.proto` is the contract, code generated from it** |
| Streaming | SSE / WebSocket | ✅ Bidirectional, native |
| Browser support | ✅ Native | Needs grpc-web + a proxy |
| Debuggability | ✅ curl | Needs tooling (grpcurl) |

**gRPC for high-volume internal service-to-service traffic where payload size and schema enforcement matter; REST at the edge** where browsers, debuggability, and third-party consumers dominate. **That's the standard, defensible answer.**

### Service discovery

```
Client-side discovery              Server-side discovery
service asks a registry             a load balancer/DNS resolves it
(Eureka, Consul)                    (Kubernetes Service, ALB)
→ the client picks an instance      → the platform picks
→ more control, more client code    → ✅ simpler, language-agnostic
```

**On Kubernetes you don't need Eureka.** `http://vigil-checker` resolves via Kubernetes DNS to a Service that load-balances across ready pods, with health-checking from readiness probes. **Eureka and Spring Cloud Netflix largely solved a pre-Kubernetes problem** — Netflix built them because that infrastructure didn't exist yet.

**Saying "I'd use Kubernetes Services rather than Eureka, because the platform already provides discovery, load balancing and health checking" is a strong currency signal.** A candidate who reaches for Eureka on Kubernetes is working from 2016 tutorials. (Spring Cloud Netflix components are largely in maintenance mode; Eureka survives, Ribbon and Hystrix are replaced by Spring Cloud LoadBalancer and Resilience4j.)

---

## 5. Distributed data — the actual hard part

### The problem, concretely

```java
// In a monolith this is one transaction. Trivially correct.
@Transactional
public void createMonitor(UUID orgId, CreateMonitorRequest req) {
    Organization org = orgRepo.findById(orgId).orElseThrow();
    if (org.monitorCount() >= org.plan().limit()) throw new QuotaExceededException();
    monitorRepo.save(new Monitor(orgId, req));
    org.incrementMonitorCount();
}
```
Split organizations and monitors into separate services and **there is no transaction that spans both.** If the monitor is created and the count update fails, your quota is permanently wrong.

### CAP and PACELC — be precise

**CAP:** under a network **P**artition you must choose **C**onsistency or **A**vailability. **The common misstatement is "pick two of three"** — partitions are not optional in a distributed system, so you're always choosing between C and A *during a partition*.

**PACELC extends it usefully:** during a **P**artition, choose **A** or **C**; **E**lse (normal operation), choose **L**atency or **C**onsistency. **That "else" clause is the one that matters day to day** — most systems aren't partitioned most of the time, and the real ongoing trade is latency versus consistency. **Mentioning PACELC is a nice depth signal.**

### Sagas

**A saga is a sequence of local transactions, each with a compensating action.** No distributed transaction; instead, explicit rollback logic.

**Orchestration** — a coordinator directs each step:
```java
@Service
public class CreateMonitorSaga {

    public void execute(CreateMonitorCommand cmd) {
        ReservationId reservation = null;
        UUID monitorId = null;
        try {
            reservation = orgClient.reserveQuota(cmd.orgId());          // step 1
            monitorId   = monitorClient.create(cmd);                    // step 2
            checkerClient.schedule(monitorId);                          // step 3
            orgClient.confirmQuota(reservation);                        // step 4
        } catch (Exception e) {
            if (monitorId != null)   monitorClient.delete(monitorId);        // compensate 2
            if (reservation != null) orgClient.releaseQuota(reservation);    // compensate 1
            throw new SagaFailedException(e);
        }
    }
}
```
✅ Explicit, visible, debuggable, easy to reason about. ❌ The orchestrator knows everyone (coupling) and is a single point of failure. **Must be persistent** — if the orchestrator dies mid-saga, something has to resume or compensate it.

**Choreography** — each service reacts to events:
```
QuotaReserved → MonitorCreated → CheckScheduled → QuotaConfirmed
      │                │                │
      └─ on failure: QuotaReleased, MonitorDeleted (compensating events)
```
✅ No central coordinator, loosely coupled. ❌ **The overall flow exists nowhere in the code** — you reconstruct it by reading N services. Debugging is genuinely hard.

**The practical answer:** *"Orchestration by default, because the business process is explicit and debuggable. Choreography when you want maximum decoupling and the flow is simple. And with more than ~3 steps I'd use a durable workflow engine — Temporal, or Spring Statemachine with persistence — because hand-rolled sagas fail at exactly the hard part: what happens when the orchestrator crashes halfway."*

**That last clause is the senior insight.** The happy path of a saga is easy; durability and resumption are the whole problem, and mentioning Temporal shows you know it's a solved problem you shouldn't re-solve.

**Compensation is not rollback — and this matters:**
- Some things **cannot** be compensated. You can refund a payment; you cannot un-send an email.
- Compensations can **themselves fail** → you need retries and eventually a human.
- Other users **see intermediate states**. The monitor exists for 200ms before being deleted, and someone's dashboard may have rendered it.
- **Design so the visible intermediate state is acceptable** — e.g. create resources in a `PENDING` state that only becomes `ACTIVE` on saga completion. **That's the practical trick most people miss.**

### Avoiding sagas entirely — usually the better answer

**Most sagas exist because a boundary was drawn in the wrong place.**

```
❌ organizations service + monitors service → every creation needs a saga
✅ ONE service owning organizations, quotas and monitors → one ACID transaction
```

**If two things must change atomically, they belong in the same service.** That's the aggregate rule (§3) applied to service boundaries. **"I'd redraw the boundary rather than write a saga" is frequently the correct and more impressive answer**, and Vigil's split deliberately avoids sagas: orgs, quotas, monitors and incidents are all in `vigil-api` precisely so quota enforcement is one transaction.

### Querying across services

```java
// ❌ API composition: N calls, latency = sum, fails if any service is down
var monitors = monitorClient.list(orgId);
var uptimes  = monitors.stream().map(m -> checkerClient.uptime(m.id())).toList();  // N calls!
```

**Options, and each has a real cost:**

| Pattern | How | Cost |
|---|---|---|
| **API composition** | Call each service, join in memory | N calls; availability = product of all |
| **CQRS read model** | A denormalized view built from events | Eventual consistency; extra storage |
| **Data replication** | Each service keeps a read-only copy of what it needs | Duplication; must handle stale data |
| **BFF** | A backend-for-frontend does the composition | Still N calls, but centralized and cacheable |

**CQRS with an event-built read model is the standard answer for dashboards:**
```java
// vigil-api consumes check.results and maintains its own uptime summary table.
// The dashboard is then ONE local query. No cross-service call at read time.
@KafkaListener(topics = "check.results")
void updateUptimeSummary(CheckResultEvent e) { uptimeSummaryRepo.upsert(e); }
```
**Trade-off to state honestly: the summary is seconds stale, and you've duplicated data.** For an uptime dashboard that's obviously fine. For an account balance it obviously isn't. **Naming where the trade-off breaks is what makes the answer credible.**

---

## 6. Cross-cutting concerns

### Configuration
Kubernetes ConfigMaps and Secrets, or Spring Cloud Config (Lesson 12). **Per-service config, not one shared file** — a shared config file is the same coupling problem as a shared database.

### Identity propagation
```
Client → Gateway (validates the JWT, strips/normalizes headers)
       → service A (trusts the gateway; propagates identity downstream)
       → service B
```
Options: forward the original JWT (simple; every service validates it), or the gateway mints an internal token (better — internal services never see customer credentials). **Never trust an unauthenticated `X-User-Id` header**, and ensure the gateway *strips* client-supplied versions of internal headers — otherwise a client sets `X-User-Id: admin` and you've built an authentication bypass. **That's a real, exploitable design mistake worth naming.**

### Distributed tracing
Non-negotiable (Lesson 35). Without it, debugging a multi-service request is guesswork.

### The database-per-service migration problem
Each service owns its schema and its migrations. But **you must still coordinate breaking changes across services** — expand/contract applies to *API* contracts too (Lesson 24 §4), not just database columns. **Contract tests are what make that safe** (Lesson 31).

---

## 7. The strangler fig pattern

**How you actually migrate, and the answer to "how would you break up a monolith?"**

```
Phase 1: monolith handles everything
         Client ──▶ Monolith

Phase 2: put a facade in front (no behaviour change — do this first, alone)
         Client ──▶ Gateway ──▶ Monolith

Phase 3: extract ONE capability; route just that path to it
         Client ──▶ Gateway ──┬──▶ Monolith
                              └──▶ new-service   (/api/v1/checks/**)

Phase 4: repeat, one capability at a time, with the option to roll back at every step

Phase 5: the monolith shrinks — possibly to nothing, possibly not, and that's fine
```

> **Fun fact:** the name comes from the strangler fig, which grows around a host tree, gradually replacing it, until the original rots away leaving a self-supporting fig. Martin Fowler named the pattern after seeing them in Australia. It's an unusually apt metaphor: the new structure is load-bearing *before* the old one is removed.

**Why this beats a rewrite:** you're never in a state where nothing works, every step is independently deployable and revertible, and you learn the real boundaries from production traffic rather than from a whiteboard. **A big-bang rewrite means a long period with two codebases, no releases, and a hard cutover — which is where rewrite projects die.**

**Extract in this order:**
1. **Something with clear boundaries and few dependencies** — the checker, not the user model
2. **Something with an independent scaling need** — so the benefit is immediate and measurable
3. **Never start with the most tangled core.** You'll fail publicly and the organization will lose faith in the migration.

**And a genuinely useful technique to name: dual-run/shadow traffic.** Route reads to both the monolith and the new service, compare the responses, and only cut over when they agree. **You verify correctness with production traffic before taking any risk.**

---

## 8. Build & break — split Vigil (optional but a strong differentiator)

**Do this only after the monolith is complete and working.** The point is to demonstrate a *justified* split, not to have "microservices" on a CV.

**1. Write the decision document first** — `docs/adr/001-extract-checker-service.md`:
```markdown
# ADR-001: Extract the check executor into a separate service

## Context
50,000 monitors at 60s intervals ≈ 833 checks/second, entirely outbound I/O.
The API serves ~50 req/s. Measured: checker pods need 8× the replicas of API pods,
and a slow customer endpoint consumes threads that the API also needs.

## Decision
Extract vigil-checker with its own database. Communicate via Kafka.

## Consequences
+ Independent scaling: checkers scale on backlog, API on request rate
+ Failure isolation: a slow customer endpoint cannot degrade the API
+ Separate DB tuning: append-heavy time series vs read-heavy relational
− Eventual consistency: check results appear in the API seconds later
− Operational cost: two deployables, two databases, two migration streams
− Local development now requires Docker Compose

## Alternatives considered
- Keep the monolith and use a bulkhead: simpler, but caps checker scale at
  the API's pod count and still shares the connection pool
- Extract organizations/users too: REJECTED — quota enforcement must be
  transactional with monitor creation, so a saga would be required. See §5.
```

**An ADR like that is one of the most persuasive artifacts a portfolio project can contain.** It proves you can reason about trade-offs, not just wire up frameworks. **Interviewers read these.**

**2. Extract `vigil-checker`:**
- Its own Spring Boot app, its own Postgres, its own Flyway migrations
- Consumes `check.requests` (or claims work directly from its own DB with `SKIP LOCKED`)
- Produces `check.results` to Kafka
- No access to the API's database, at all

**3. `vigil-api` consumes `check.results`** and maintains a local `uptime_summary` read model (CQRS). **Note that the dashboard is now one local query rather than a cross-service call** — that's the pattern working.

**4. Docker Compose for the whole system:** api, checker, notifier, postgres ×3, kafka, redis, prometheus, grafana. **One `docker compose up` must bring up the entire platform** — if it doesn't, nobody (including a reviewer) will run your project.

**5. ⭐ Prove the independent scaling.**
```bash
docker compose up --scale vigil-checker=5
```
Seed 10,000 monitors. **Show that 5 checkers process the backlog ~5× faster than 1**, while API latency is unaffected. **Graph it in Grafana and screenshot it.** That graph is the evidence for your ADR, and it's exactly the kind of concrete result that makes an interview memorable.

**6. Prove the failure isolation.** Point 100 monitors at a deliberately hanging endpoint. **Confirm the API stays fast** and that checker capacity degrades gracefully (circuit breakers open — Lesson 34). Then reason about what would have happened in the monolith.

**7. Distributed tracing across services.** Confirm a single trace spans api → kafka → checker → kafka → notifier. **Screenshot the waterfall.** A trace crossing three services and a broker is a genuinely impressive artifact.

**8. Contract tests** between api and checker (Lesson 31), so a breaking change to the event schema fails CI rather than production.

**9. Then write the honest retrospective** — `docs/MICROSERVICES-RETROSPECTIVE.md`:
> *"Was the split worth it? For the checker, yes — measured 5× throughput scaling independent of the API, and verified failure isolation. For notifications, marginal: it needs 3 pods either way, and the main benefit was isolating third-party retry state. If I were building this for a team of three, I'd keep everything as a modular monolith with a bulkhead around checks and revisit at ~10× the load."*

**That paragraph — being willing to say part of your own architecture wasn't worth it — is more impressive than the architecture.** It's evidence of judgment rather than enthusiasm, and it's exactly what distinguishes a senior engineer.

### Explain out loud (60s)

1. What a monolith gives you free that a microservice takes away — five items.
2. The two things microservices genuinely buy, and why one is organizational.
3. Three defensible reasons to extract a service, and three bad ones.
4. Why "a shared database" means you built a distributed monolith.
5. When you'd write a saga, and when you'd move the boundary instead.

---

## 9. Production rules

| Do / Never | Why |
|---|---|
| **Start from a modular monolith and extract along boundaries you already have** | If you can't draw clean boundaries inside one codebase, distributing them just converts method calls into unreliable network calls and keeps the coupling |
| **Never split because the monolith is messy** | Distribution doesn't make bad boundaries good; it makes them expensive and much harder to move later |
| **Give every service its own data** | A shared database is the definitive distributed monolith — the schema is still a breakable contract between everyone who touches it |
| **Keep anything that must change atomically in one service** | That is the aggregate rule applied to boundaries, and it's the single strongest signal you have |
| **Prefer redrawing a boundary over writing a saga** | Most sagas exist because the boundary was wrong, and compensation is strictly harder than the transaction it replaced |
| **Use orchestration for sagas by default** | The business process stays explicit and debuggable; choreography is for simple, maximally decoupled flows |
| **Reach for a durable workflow engine past about three steps** | The happy path is easy — what happens when the orchestrator crashes halfway is the whole problem, and it's already solved |
| **Use sync for queries, async for state propagation** | A chain of four synchronous hops is 96% availability and the sum of the latencies; a queue absorbs the failure instead |
| **Never make a service-to-service call without a timeout, breaker and bulkhead** | It is the mechanism by which one service's slowness becomes a cluster-wide outage |
| **Use Kubernetes Services rather than Eureka on Kubernetes** | The platform already provides discovery, load balancing and health checking; Eureka solved a pre-Kubernetes problem |
| **Choose gRPC for high-volume internal traffic, REST at the edge** | Binary payloads and a generated contract win internally; browsers, curl and third parties win at the boundary |
| **State CAP correctly: partitions aren't optional** | "Pick two of three" is the common misstatement — you choose C or A *during* a partition, and latency vs consistency the rest of the time |
| **Say where an eventual-consistency trade-off breaks** | Seconds-stale is obviously fine for an uptime dashboard and obviously wrong for an account balance; naming the line is what makes the answer credible |
| **Migrate with the strangler fig, never a big-bang rewrite** | Every step stays deployable and revertible, and you learn the real boundaries from production traffic instead of a whiteboard |

---

## 10. Interview traps

**Q1. Would you use microservices for a new project?** *(the most likely question)*
No, by default. I'd start with a modular monolith: feature packages with enforced boundaries (ArchUnit), no cross-feature shared tables, communication via public service interfaces or domain events. Then extract a service for a concrete reason — a genuinely different scaling profile, a team needing independent deploys, or an isolation requirement — along a boundary that already exists. Microservices solve an *organizational* scaling problem; a small team pays the full distributed-systems cost for a benefit it can't collect.

**Q2. What do you lose when you split a monolith?**
ACID transactions across features (→ sagas), fast typed in-process calls (→ network calls that fail), compiler-verified refactoring across boundaries (→ coordinated deploys and contract tests), single-stack-trace debugging (→ distributed tracing), a trivial local environment, and cheap cross-domain queries (→ API composition or CQRS read models). Plus service discovery, a gateway, per-service pipelines, databases and migrations.

**Q3. What are the two real benefits?**
Independent deployability (organizational) and independent scalability (technical). Everything else claimed for microservices — modularity, clear boundaries, testability — is achievable in a monolith with discipline.

**Q4. Why is a shared database an anti-pattern?**
Because the schema becomes a contract shared by every service, so one change breaks several and deploys must be coordinated. You've added latency, partial failure and operational complexity while keeping all your coupling — a distributed monolith, which is strictly worse than either alternative.

**Q5. How do you decide where boundaries go?**
Primarily by **transactional boundaries** — things that must change atomically belong in one service (the aggregate rule). Then rate of change, team ownership, data access patterns, and failure isolation. Never by technical layer (a distributed n-tier app needs a change in every service per feature) and never one-service-per-entity (guarantees chatty cross-service calls).

**Q6. What is a saga, and orchestration vs choreography?**
A sequence of local transactions with compensating actions, replacing a distributed transaction. Orchestration: a coordinator drives the steps — explicit and debuggable, but coupled and a single point of failure that **must be durable**. Choreography: services react to events — decoupled, but the overall flow exists in no single place. I'd default to orchestration, and use a durable workflow engine (Temporal) beyond ~3 steps because crash-resumption is the actual hard part.

**Q7. What's wrong with compensating transactions?**
They aren't rollbacks. Some actions can't be undone (a sent email). Compensations can themselves fail, requiring retries and eventual human intervention. And intermediate states are **visible to other users**. The practical mitigation is designing so the visible intermediate state is acceptable — e.g. creating resources `PENDING` and only activating them on saga completion.

**Q8. How do you avoid sagas?**
Redraw the boundary. Most sagas exist because things that must change atomically were placed in different services. If two entities need one transaction, they belong in one service — that's the aggregate rule applied to boundaries, and it's usually the better answer than writing a saga.

**Q9. How do you query data spanning services?**
API composition (N calls; availability is the product of all services — poor for dashboards), a **CQRS read model** built from events (one local query, seconds stale — the standard answer for dashboards), data replication, or a BFF. State the staleness trade-off: fine for uptime percentages, unacceptable for account balances.

**Q10. Explain CAP, precisely.**
Under a network partition you must choose consistency or availability. It is *not* "pick two of three" — partitions are unavoidable, so the choice is between C and A during one. **PACELC** extends it usefully: during a Partition choose A or C, Else choose Latency or Consistency — and since systems aren't usually partitioned, the latency/consistency trade is the one you make every day.

**Q11. How do you migrate a monolith to services?**
Strangler fig: put a facade/gateway in front with no behaviour change, then extract one capability at a time and route just that path to it, keeping the ability to roll back at every step. Start with something well-bounded that has an independent scaling need — never the tangled core. Use dual-run/shadow traffic to compare old and new responses before cutting over. Avoid big-bang rewrites: they mean a long period with two codebases, no releases, and a risky hard cutover.

**Q12. Do you need Eureka on Kubernetes?**
No. Kubernetes Services provide DNS-based discovery, load balancing, and health checking from readiness probes. Eureka and Spring Cloud Netflix solved a pre-Kubernetes problem; Netflix built them because that platform didn't exist. Reaching for Eureka on Kubernetes is duplicating the platform.

**Q13. REST or gRPC between services?**
gRPC for high-volume internal traffic where payload size and code-generated schema enforcement matter, and for streaming. REST at the edge for browser support, debuggability, and third-party consumers. A common architecture uses both.

**Q14. How does identity propagate between services?**
The gateway validates the JWT and either forwards it or mints a short-lived internal token (better — internal services never handle customer credentials). Critically, the gateway must **strip** client-supplied internal headers; trusting an unauthenticated `X-User-Id` is an authentication bypass.

---

## 11. Retention check

1. Give the two genuine benefits of microservices, and say which is organizational.
2. Why is a shared database strictly worse than either a monolith or true microservices?
3. Give the strongest heuristic for a service boundary, and the two worst.
4. Name three ways a compensating transaction differs from a rollback.
5. When would you redraw a boundary instead of writing a saga?
6. Explain the strangler fig and why it beats a rewrite.

---

**Next:** [Lesson 39 — Gateway & cross-cutting concerns](39-gateway-and-cross-cutting.md) — the edge: routing, auth, rate limiting, and tracing across service boundaries.

# Lesson 39 — Gateway & cross-cutting concerns

> **Why this lesson exists:** Once you have several services, a set of concerns stops belonging to any one of them: routing, authentication, rate limiting, tracing, versioning. Putting these at the edge is how you avoid implementing them N times inconsistently. This lesson also covers the operational realities — service mesh, deployment coordination, and the failure modes of the edge itself.

**Time:** ~60 minutes · **Prereq:** Lessons 17, 25, 35, 38

---

## 1. Why an API gateway

```
WITHOUT a gateway                      WITH a gateway
Client must know:                      Client knows ONE address
 · api.vigil.io:8080                   Gateway handles:
 · checker.vigil.io:8081                · routing
 · notifier.vigil.io:8082               · authentication (once)
Each service implements:                · rate limiting (globally)
 · auth                                 · CORS
 · rate limiting                         · TLS termination
 · CORS                                  · request/response logging
 · TLS                                   · trace initiation
→ N implementations, N inconsistencies  → one place, one policy
```

**The genuine benefits:**
1. **One entry point** — clients don't need to know your topology, so you can re-split services without breaking them
2. **Cross-cutting policy in one place** — auth, rate limiting, CORS, TLS implemented once
3. **Topology decoupling** — extract or merge a service and only the gateway's routes change
4. **Aggregation** — one client call can fan out to several services (though see BFF, §4)

**And the costs, which you must be able to name:**
1. **A single point of failure.** If the gateway is down, everything is down. **It must be more available than anything behind it** — multiple replicas, aggressive health checks, and no business logic.
2. **A latency hop** — usually 1–5ms, but it's real
3. **An operational component** to configure, monitor, and upgrade
4. **A place where business logic accretes.** The most common failure mode: routing rules become conditional business decisions, and the gateway becomes an undeployable monolith nobody owns. **Keep it dumb.**

---

## 2. Spring Cloud Gateway

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

**Important architectural note: Spring Cloud Gateway is built on WebFlux, not MVC.** That's the correct choice for a gateway — it's a pure I/O-bound proxy holding many concurrent connections with almost no business logic, which is exactly WebFlux's sweet spot (Lesson 18 §4). **Consequence: never put blocking code in a gateway filter.** One blocking call freezes an event-loop thread and, with only a few of them, the whole gateway.

> **Boot 4 note:** Spring Cloud Gateway now also ships an **MVC (servlet) variant** (`spring-cloud-starter-gateway-mvc`), which pairs well with virtual threads. Worth knowing both exist; the reactive one remains the default for high-connection-count edges.

```yaml
spring:
  cloud:
    gateway:
      default-filters:
        - DedupeResponseHeader=Access-Control-Allow-Origin
        - AddRequestHeader=X-Gateway, vigil-gw
      routes:
        - id: monitors
          uri: lb://vigil-api                    # ★ 'lb://' = load-balanced by discovery
          predicates:
            - Path=/api/v1/monitors/**,/api/v1/orgs/**
          filters:
            - name: CircuitBreaker
              args:
                name: apiCircuitBreaker
                fallbackUri: forward:/fallback/api
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 20
                redis-rate-limiter.burstCapacity: 40
                key-resolver: "#{@apiKeyResolver}"
            - name: Retry
              args:
                retries: 2
                methods: GET                     # ★ ONLY idempotent methods
                series: SERVER_ERROR
                backoff: { firstBackoff: 50ms, maxBackoff: 500ms, factor: 2 }

        - id: status-pages
          uri: lb://vigil-api
          predicates:
            - Path=/status/**
          filters:
            - name: RequestRateLimiter            # public → tighter limits
              args:
                redis-rate-limiter.replenishRate: 5
                redis-rate-limiter.burstCapacity: 10
                key-resolver: "#{@ipKeyResolver}"

        - id: checker-internal
          uri: lb://vigil-checker
          predicates:
            - Path=/internal/checks/**
            - Header=X-Internal-Token, .+         # ★ not reachable from the public internet
```

**Two details in that config worth highlighting:**

**`methods: GET` on the retry filter is essential.** A gateway retrying a POST after a timeout creates a duplicate — the request may well have succeeded and only the response was lost (Lesson 34 §5). **Only retry idempotent methods, or require an idempotency key.** A gateway configured to retry all methods is a duplicate-creation machine, and it's a genuinely common misconfiguration.

**Different rate limits per route.** The authenticated API gets 20/s per key; public status pages get 5/s per IP. **One global limit is either too loose for public endpoints or too tight for paying customers.**

### Predicates and filters

**Predicates** (route matching): `Path`, `Method`, `Header`, `Query`, `Host`, `Cookie`, `After`/`Before`/`Between` (time windows — useful for scheduled cutovers), `RemoteAddr`, `Weight` (**canary routing**).

**Filters** (request/response modification): `AddRequestHeader`, `RemoveRequestHeader`, `RewritePath`, `StripPrefix`, `SetStatus`, `CircuitBreaker`, `RequestRateLimiter`, `Retry`, `RequestSize`, `SecureHeaders`.

```yaml
# Canary release: 10% of traffic to v2, purely by configuration
- id: monitors-v1
  uri: lb://vigil-api-v1
  predicates: [ Path=/api/v1/monitors/**, "Weight=monitors, 90" ]
- id: monitors-v2
  uri: lb://vigil-api-v2
  predicates: [ Path=/api/v1/monitors/**, "Weight=monitors, 10" ]
```

**`Weight` gives you canary deployments with no code, no feature flags, and no client changes** — and you can shift the ratio while watching error rates in Grafana. That's a genuinely useful capability and a good thing to mention in a deployment discussion.

### A custom global filter

```java
@Component
@Order(-1)                                     // ★ before routing
public class CorrelationIdGatewayFilter implements GlobalFilter {

    private static final String HEADER = "X-Correlation-Id";

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String id = Optional.ofNullable(exchange.getRequest().getHeaders().getFirst(HEADER))
                            .filter(StringUtils::hasText)
                            .orElseGet(() -> UUID.randomUUID().toString());

        ServerHttpRequest mutated = exchange.getRequest().mutate()
                .header(HEADER, id)
                .headers(h -> h.remove("X-User-Id"))       // ★★ STRIP client-supplied internals
                .headers(h -> h.remove("X-Org-Id"))
                .build();

        exchange.getResponse().getHeaders().add(HEADER, id);

        return chain.filter(exchange.mutate().request(mutated).build())
                .contextWrite(Context.of("correlationId", id));   // ★ Reactor Context, not MDC
    }
}
```

**The header-stripping lines are a security control, not hygiene.** If downstream services trust `X-User-Id`, and the gateway forwards a client-supplied one, **any client can impersonate any user** — a complete authentication bypass in three characters of missing code. **The gateway must remove every internal header before setting its own.** (Lesson 38 §5.)

**And note `contextWrite` rather than `MDC.put`.** In WebFlux, a request hops threads between operators, so `ThreadLocal` doesn't follow it — you use Reactor's `Context` (Lesson 18 §2). Sixth appearance of the ThreadLocal lesson, in its reactive form.

---

## 3. Authentication at the edge

```
                    ┌─────────────────────────────────────┐
   Client ──JWT───▶ │  GATEWAY                            │
                    │  · validates signature (JWKS)       │
                    │  · validates exp / iss / aud        │
                    │  · STRIPS client internal headers   │
                    │  · sets X-User-Id, X-Org-Id, X-Roles│
                    │    (or mints an internal token)     │
                    └───────────────┬─────────────────────┘
                                    ▼
                    ┌─────────────────────────────────────┐
                    │  vigil-api                           │
                    │  trusts the gateway for identity     │
                    │  enforces AUTHORIZATION itself       │
                    └─────────────────────────────────────┘
```

### The two models

**Model A — the gateway validates, services trust headers.**
```yaml
spring.security.oauth2.resourceserver.jwt.issuer-uri: https://auth.vigil.io
```
✅ Validated once; services stay simple.
❌ **Services are now only as secure as the network.** Anyone who can reach a service directly can set `X-User-Id` freely.

**Model B — every service validates the JWT itself.**
✅ Defence in depth; a service is safe even if reached directly.
❌ Every service needs JWKS access and validation config; slightly more work per request (though JWKS is cached, so it's cheap).

**The answer that shows judgment:**

> *"Model B — every service validates the token — because Model A makes the network the security boundary, and 'the internal network is trusted' is exactly the assumption zero-trust architecture exists to reject. Validation is cheap since JWKS is cached. The gateway still does the first check so bad tokens are rejected early, and it strips client-supplied internal headers. If services must trust headers, then mTLS between services is mandatory, which in practice means a service mesh."*

**That answer covers the trade-off, names the security principle, and identifies the mitigation.** It's much stronger than picking one model.

**⭐ And the crucial distinction: authentication at the edge, authorization in the service.**

The gateway can check "is this a valid token?" It cannot check "may *this* user delete *this* monitor?" — that requires the data (Lesson 28 §3). **Object-level authorization must live in the service that owns the data.** A gateway that tries to do authorization either duplicates business rules or performs data lookups it shouldn't.

**This is the single most important point in the lesson**, because getting it wrong produces either a gateway full of business logic or services with no authorization at all.

---

## 4. BFF — Backend for Frontend

```
                    ┌──── web-bff ────┐        different clients need
   Web SPA ───────▶ │ aggregates for  │        different shapes, so give
                    │ a dashboard grid│        them different backends
                    └────────┬────────┘
                             │
                    ┌────────┴────────┐
                    │  vigil services │
                    └────────┬────────┘
                             │
   Mobile app ─────▶ ┌───────┴─────────┐
                     │  mobile-bff     │  smaller payloads, fewer round trips,
                     │  battery-aware  │  aggressive caching
                     └─────────────────┘
```

**The problem it solves:** a mobile client on a poor connection wants one call returning exactly the fields it renders. A web dashboard wants richer data. A generic API serves both badly — either it over-fetches for mobile or it needs five round trips.

**Why a BFF rather than options on a shared API:** each BFF is **owned by the client team** and evolves with the client, so changing the mobile app's data needs doesn't require negotiating with a shared-API team. **That ownership property is the real argument**, and it's what distinguishes a BFF from "just another gateway."

**GraphQL as an alternative:** the client specifies exactly what it wants, so one endpoint serves all clients. Spring for GraphQL is mature. **Trade-offs: harder to cache (POST with a body, no URL-based caching), the N+1 problem returns with force (needs DataLoader batching), and query complexity is a genuine DoS vector requiring depth/complexity limits.** Being able to name those three specific costs is what makes a GraphQL answer credible rather than enthusiastic.

**And the honest caution:** a BFF per client is more code to maintain. With one web client, a well-designed REST API plus sparse fieldsets is usually enough. **BFFs earn their keep at 3+ meaningfully different clients.**

---

## 5. Service mesh — know the concept and the trade-off

```
WITHOUT a mesh: every service implements resilience in its own code
   service A ──[Resilience4j, mTLS, tracing, retries]──▶ service B
   → N implementations; polyglot services duplicate it in each language

WITH a mesh: a sidecar proxy handles it, transparently
   service A ──▶ [envoy sidecar] ══mTLS══▶ [envoy sidecar] ──▶ service B
   → policy is configuration, applied uniformly, in no application's code
```

**What a mesh (Istio, Linkerd) gives you:**
- **Automatic mTLS between all services** — encryption and identity with zero application code. **This is the strongest single argument.**
- Retries, timeouts, circuit breaking, and load balancing as policy
- Traffic shifting for canary/blue-green, by configuration
- Automatic L7 metrics and trace propagation for every service
- Fault injection for chaos testing

**The costs, and they're real:**
- **Significant operational complexity.** Istio in particular has a reputation for being a system unto itself.
- A sidecar per pod: extra memory (~50–100MB) and CPU, plus a latency hop each way
- **Two layers of retries and timeouts** — application *and* mesh — which interact confusingly. Retry amplification (Lesson 34 §2) gets worse if both retry.
- Another component in every request path to debug

**The calibrated answer:** *"A mesh is worth it when you have many services, especially polyglot ones, and need uniform mTLS and traffic policy without touching application code. For a handful of Spring services, Resilience4j plus Kubernetes Services plus Micrometer tracing covers most of it at far lower operational cost. And if you adopt a mesh, you should move resilience policy **out** of the application rather than having both layers retry — the double-retry interaction is a genuine trap."*

**Linkerd is notably simpler than Istio** and worth naming if the requirement is mainly mTLS and metrics.

---

## 6. Versioning and coordinated change

**The core problem: services deploy independently, so two versions coexist. Every change must be compatible with the version currently running.**

**Expand/contract applies to APIs, exactly as it does to database columns** (Lesson 24 §3):

```
Renaming a response field across services:

Deploy 1: the provider returns BOTH `name` and `title` (additive → safe)
Deploy 2: consumers switch to reading `title`
Deploy 3: (once no consumer reads `name`) the provider removes it
```

**API versioning across services** — the same rules as Lesson 16 §4, plus one addition: **internal APIs can evolve faster than public ones, because you know and control all consumers.** For internal service-to-service contracts, additive-only evolution plus contract tests is usually sufficient and cheaper than maintaining `/v1` and `/v2` in parallel.

**Contract tests are what make independent deploys safe** (Lesson 31 §2). Without them, "independent deployability" is a slogan — you'll coordinate anyway, out of fear.

**Deployment ordering — the rule worth memorizing:**

```
Adding a field to a response:      deploy the PROVIDER first (consumers ignore extras)
Adding a required request field:    deploy the CONSUMER first? No —
                                    make it OPTIONAL, deploy the provider, then the consumer,
                                    then make it required
Removing anything:                  deploy CONSUMERS first (stop using it), then the provider
Adding a new event type:            deploy CONSUMERS first (so they're ready to handle it)
```

**Generalization: additive changes deploy provider-first; removals deploy consumer-first.** Getting this backwards causes a brief outage during every rolling deploy — brief enough to be dismissed as a fluke, frequent enough to erode trust in deploys. **That's a specific, practical piece of knowledge very few candidates have.**

---

## 7. The gateway's own failure modes

**The gateway is a single point of failure, so it needs more care than the services behind it.**

| Risk | Mitigation |
|---|---|
| Gateway down → total outage | Multiple replicas; no shared state; a DNS/L4 load balancer in front |
| Gateway becomes a bottleneck | Keep it stateless; **no blocking calls**; scale horizontally |
| A slow backend saturates gateway connections | Per-route timeouts, circuit breakers, and **bulkheads per route** |
| Business logic accretes | Review discipline: routing and policy only, never business rules |
| Config error breaks all routes | Version and test the config; canary the gateway itself |
| **Retries on non-idempotent methods** | `methods: GET` only, or require an idempotency key |

**The bulkhead-per-route point deserves emphasis:** without it, a slow `vigil-checker` consumes all the gateway's connection capacity and **requests to `vigil-api` fail too** — the gateway becomes the mechanism by which one service's problems become everyone's. **That's the same failure isolation argument as Lesson 34 §4, applied one layer up**, and it's a genuinely good observation.

**Health-checking must be right too:** the gateway should route only to *ready* backends (Kubernetes readiness probes do this for you — Lesson 35 §2.1), and its **own** health check must not depend on all backends being healthy, or one bad service takes the gateway out of rotation and breaks everything.

---

## 8. Build this — Vigil's gateway (optional, after the split)

**1. Create `vigil-gateway`** with `spring-cloud-starter-gateway`, routes for `/api/v1/**` → api, `/status/**` → api, and `/internal/**` → checker (header-gated).

**2. JWT validation at the edge** with `oauth2-resource-server`, plus the `CorrelationIdGatewayFilter` from §2 that sets identity headers **and strips client-supplied ones.**

**3. ⭐ Prove the header-injection vulnerability.** First, have `vigil-api` trust `X-User-Id` and have the gateway forward headers without stripping. Then:
```bash
curl -H "Authorization: Bearer $USER_TOKEN" -H "X-User-Id: <admin-uuid>" \
     http://localhost:8080/api/v1/admin/organizations
```
**Confirm you get admin access as a normal user.** Then add the stripping filter and confirm it's blocked. **Then implement Model B** — the service validates the JWT itself — and confirm the attack fails even with header stripping removed. **You've just demonstrated defence in depth on your own system, which is a genuinely compelling thing to describe.**

**4. Redis rate limiting** with different limits per route: 20/s per API key for authenticated routes, 5/s per IP for public status pages. Verify with `hey -n 100 -c 20` and confirm 429s with `Retry-After`. **Then run two gateway replicas and confirm the limit is enforced *globally*** — that's why the limiter is Redis-backed rather than in-memory (Lesson 17 §4).

**5. Circuit breaker with a fallback.** Stop `vigil-api` and confirm requests get the fallback response quickly rather than hanging. Watch the circuit state in `/actuator/health`.

**6. ⭐ Prove the retry trap.** Configure `Retry` with `methods: GET,POST`. Make `vigil-api` slow to respond on POST (respond after the gateway's timeout, but still process the request). **Send one POST and confirm two monitors were created.** Then restrict to `methods: GET` and confirm exactly one. **This is a five-minute experiment that permanently fixes your instincts about gateway retries.**

**7. Canary with `Weight`.** Run two instances of `vigil-api` (tag one so you can tell them apart in logs), split 90/10 by weight, send 100 requests, and confirm the distribution. **Then shift to 50/50 without restarting the gateway.**

**8. Tracing across the whole system.** Confirm one trace spans gateway → api → kafka → checker → kafka → notifier. **Screenshot the waterfall for your README** — a trace crossing a gateway, three services and a broker is an unusually good portfolio artifact.

**9. Bulkhead per route.** Make `vigil-checker` hang. **Confirm `/api/v1/monitors` still responds.** Then remove the per-route isolation and confirm the gateway degrades globally. **That contrast is the whole argument for bulkheads at the edge.**

**10. Gateway resilience.** Run two gateway replicas behind an nginx or Docker Compose load balancer. Kill one mid-load-test and **confirm zero failed requests.** Then confirm the gateway's own health check doesn't depend on backend health.

---

## 9. Interview traps

**Q1. What does an API gateway do, and what are the risks?**
Single entry point providing routing, authentication, rate limiting, CORS, TLS termination, and trace initiation — implemented once instead of N times. Risks: it's a single point of failure (so it needs more replicas and better health checks than what's behind it), it adds a latency hop, and it accretes business logic if not policed. Keep it dumb: routing and policy only.

**Q2. Why is Spring Cloud Gateway built on WebFlux?**
A gateway is a pure I/O-bound proxy holding many concurrent connections with almost no computation — exactly WebFlux's sweet spot. The consequence: **never put blocking code in a gateway filter**, because with only a few event-loop threads, one blocking call freezes the whole gateway. (A servlet/MVC variant now exists too, which pairs with virtual threads.)

**Q3. Should the gateway do authentication, authorization, or both?**
**Authentication at the edge, authorization in the service.** The gateway can validate a token; it cannot answer "may *this* user delete *this* monitor?" without the data. Object-level authorization must live in the service that owns the data, or the gateway either duplicates business rules or performs lookups it shouldn't.

**Q4. Should services re-validate the JWT, or trust gateway headers?**
Services should validate. Trusting headers makes the network the security boundary — the assumption zero-trust exists to reject — and anyone who reaches a service directly can set `X-User-Id`. Validation is cheap because JWKS is cached. If services must trust headers, mTLS between them becomes mandatory, which in practice means a service mesh.

**Q5. What's the most dangerous gateway misconfiguration?**
Two candidates. **Forwarding client-supplied internal headers**: if downstream trusts `X-User-Id` and the gateway doesn't strip it, any client impersonates any user — a total authentication bypass. And **retrying non-idempotent methods**: a retried POST after a timeout creates duplicates, because the original may have succeeded with only the response lost. Restrict retries to GET or require an idempotency key.

**Q6. How do you rate limit across multiple gateway instances?**
Redis-backed (Spring Cloud Gateway's `RequestRateLimiter` with `redis-rate-limiter`), so the counter is shared. In-memory limits multiply by replica count — three gateways with a 100/min limit each permit 300/min.

**Q7. What is a BFF and when is it worth it?**
A backend per client type, **owned by that client's team**, returning exactly the shape that client needs — so the mobile app's data requirements can change without negotiating with a shared-API team. That ownership is the real argument. Worth it at three-plus meaningfully different clients; with one web client, a good REST API plus sparse fieldsets is enough.

**Q8. GraphQL instead?**
It solves over/under-fetching with one endpoint for all clients. Costs: hard to cache (POST with a body, no URL-based caching), the N+1 problem returns and needs DataLoader batching, and query complexity is a real DoS vector requiring depth and complexity limits.

**Q9. What is a service mesh and would you use one?**
Sidecar proxies handling mTLS, retries, timeouts, circuit breaking, traffic shifting, and L7 metrics without application code. Worth it for many (especially polyglot) services needing uniform mTLS and traffic policy. For a handful of Spring services, Resilience4j + Kubernetes Services + Micrometer tracing covers most of it far more cheaply. **And if you adopt one, move resilience policy out of the application** — application *and* mesh both retrying compounds retry amplification.

**Q10. How do you deploy a change across services without downtime?**
Expand/contract on the API contract, and get the **ordering** right: additive changes deploy **provider-first** (consumers ignore extra fields); removals deploy **consumer-first** (stop using it, then remove). New event types: consumers first. Contract tests make it safe — without them, "independent deployability" is a slogan and teams coordinate out of fear.

**Q11. How do you do a canary release with a gateway?**
The `Weight` predicate splits traffic by ratio between two routes — no code, no feature flags, no client changes — and you shift the ratio while watching error rates and latency. `Header`-based routing gives targeted canaries for specific users or internal traffic.

**Q12. How do you stop one slow backend from taking down the gateway?**
Per-route timeouts, circuit breakers, and **bulkheads per route**, so a slow checker can't consume all connection capacity and fail requests bound for the API. Same failure-isolation argument as bulkheads inside a service, applied at the edge. Plus: the gateway's own health check must not depend on backend health.

---

## 10. Retention check

1. Give the one-line rule for what the gateway does versus what the service does, regarding auth.
2. Name the two most dangerous gateway misconfigurations and the harm each causes.
3. Why must the gateway strip internal headers before setting them?
4. Why is a Redis-backed rate limiter necessary at the edge?
5. Give the deployment-ordering rule for additive vs removal changes.
6. When is a service mesh worth its operational cost, and what must you change if you adopt one?

---

## Module 10 complete ✅

You can now reason about asynchronous messaging with precise delivery semantics, decide *whether* to split a system and *where*, handle distributed data honestly, and run the edge. **Crucially, you can also argue against microservices** — which is the more valuable skill.

**Next:** [Lesson 40 — Docker, CI/CD & deployment](../11-deployment/40-docker-cicd-deployment.md) — getting Vigil onto a real URL that other people can use.

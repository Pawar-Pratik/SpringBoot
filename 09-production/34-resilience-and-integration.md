# Lesson 34 — Resilience & outbound integration

> **Why this lesson exists:** Every distributed system fails, and the question is only whether it fails *gracefully*. This lesson is about the patterns that stop someone else's outage from becoming yours — timeouts, retries, circuit breakers, bulkheads, idempotency. For Vigil these aren't optional extras: its entire job is making thousands of calls to unreliable third-party endpoints and delivering notifications to APIs that go down.

**Time:** ~75 minutes · **Prereq:** Lessons 14, 18, 19, 23

---

## 1. Timeouts — the one that matters most

**If you take one thing from this lesson: every network call needs a timeout, and the defaults are effectively infinite.**

```
No read timeout + a hung upstream
    → your thread blocks forever
    → the thread pool exhausts (Lesson 14)
    → the connection pool exhausts (Lesson 19)
    → EVERY endpoint fails, including /health
    → your service is down because someone else's is slow
```

**This is the single most common cause of cascading production failure**, and it's a one-line fix that people don't make because nothing goes wrong until the day it does.

```java
@Bean
RestClient checkClient(RestClient.Builder builder) {
    return builder
            .requestFactory(ClientHttpRequestFactories.get(
                    ClientHttpRequestFactorySettings.DEFAULTS
                            .withConnectTimeout(Duration.ofSeconds(3))   // TCP handshake
                            .withReadTimeout(Duration.ofSeconds(10))))   // waiting for bytes
            .build();
}
```

### The timeouts you must set, at every layer

| Layer | Setting | Sane value |
|---|---|---|
| HTTP connect | `withConnectTimeout` | 2–5s (a healthy TCP handshake is <100ms) |
| HTTP read | `withReadTimeout` | 5–30s, based on the endpoint's real p99 |
| Connection acquisition (pooled clients) | pool `connectionRequestTimeout` | 1–3s |
| Database query | `statement_timeout` / `@QueryHints` javax.persistence.query.timeout | 5–30s |
| Transaction | `@Transactional(timeout = 10)` | Bounds the whole unit of work |
| Hikari pool wait | `connection-timeout` | 2–3s (Lesson 19) |
| Redis command | `spring.data.redis.timeout` | 1–2s (Lesson 33) |
| Server request | `server.tomcat.connection-timeout` | 20s |

**Set `statement_timeout` at the database level too** — it's the backstop that catches queries no application timeout covers:
```yaml
spring:
  datasource:
    hikari:
      connection-init-sql: "SET statement_timeout = '30s'"
```

### Choosing values: the timeout budget

**Timeouts must decrease as you go deeper.** If your API promises a 3-second response but calls a service with a 10-second timeout, your timeout is a lie and your caller times out first — leaving you doing work nobody will read.

```
Client       → 5s
  API gateway  → 4s
    Vigil API    → 3s
      Slack call   → 2s        ← each layer is strictly smaller
      DB query     → 1s
```

**Derive from measured p99, not from guesses.** A timeout below p99 fails healthy requests; one at 10× p99 is not a timeout, it's a formality. **Rule of thumb: p99 × 1.5–2, then verify your error rate doesn't move.**

**And a subtlety worth knowing:** a read timeout is *per-read*, not per-request. A server that sends one byte every 9 seconds never trips a 10-second read timeout and can hold your thread indefinitely. For genuine protection you also need an **overall request budget** — Resilience4j's `TimeLimiter`, or an HTTP client that supports a total-request timeout. **This is a real gap most people don't know about, and naming it is a strong signal.**

---

## 2. Retries

```xml
<dependency><groupId>org.springframework.retry</groupId>
  <artifactId>spring-retry</artifactId></dependency>
```

```java
@EnableRetry
@Configuration
public class RetryConfig { }
```

```java
@Service
public class SlackNotifier {

    @Retryable(
        retryFor = {ResourceAccessException.class, HttpServerErrorException.class},
        noRetryFor = {HttpClientErrorException.class},          // ★ never retry a 4xx
        maxAttempts = 3,
        backoff = @Backoff(delay = 500, multiplier = 2, maxDelay = 5000, random = true))
    public void notify(Incident incident) { }

    @Recover
    public void fallback(Exception e, Incident incident) {
        log.error("Slack delivery failed after retries for incident {}", incident.getId(), e);
        deadLetterRepository.save(DeadLetter.from(incident, e));   // ★ don't silently lose it
    }
}
```

### The rules of retrying, and each has a real failure behind it

**1. Only retry *transient* failures.**

| Retry | Don't retry |
|---|---|
| Connection refused / reset | **400 Bad Request** — it'll be bad next time too |
| Timeouts | **401/403** — credentials won't improve |
| 502/503/504 | **404** — it won't appear |
| 429 (**honour `Retry-After`**) | **422** — semantically invalid |
| Deadlock / serialization failure | Any validation error |
| Optimistic lock conflict (Lesson 23) | |

**Retrying a 400 is a bug that triples your load and fixes nothing.** It also masks the real error in logs.

**2. Always use exponential backoff *with jitter*.**

```
No backoff:            retry, retry, retry — you DDoS a struggling service
Fixed 1s:              10,000 clients retry in lockstep every second — a synchronized wave
Exponential:           0.5s, 1s, 2s, 4s — better, but all clients still align
Exponential + JITTER:  0.4s, 1.1s, 2.3s, 3.6s — ✅ retries spread out
```

**`random = true` is not a nicety.** Without jitter, all clients that failed together retry together, producing a **retry storm** that keeps the recovering service down. This is a well-documented cause of prolonged outages — the dependency comes back up, gets hit by every client simultaneously, and falls over again. **Naming the retry storm is one of the best answers in this lesson.**

**3. Retries multiply load — bound them.**

```
3 services chained, each retrying 3×:  3 × 3 × 3 = 27 requests
```
**Retry amplification.** With retries at every layer, one user request becomes dozens against your deepest dependency — precisely when it's already struggling. **Retry at ONE layer, ideally the outermost that can meaningfully recover**, and never retry a request that a lower layer already retried.

**4. Retries require idempotency.** §5. A retried POST that isn't idempotent creates duplicates. **This is the constraint that makes retries dangerous rather than free.**

**5. Never retry inside a transaction.**
```java
// ❌ Sleeping while holding a database connection and locks
@Transactional
public void process() { retryTemplate.execute(ctx -> slackClient.send(...)); }
```
You're holding a pooled connection through the backoff delays (Lesson 23 §6). Retry *outside* the transaction, or better, publish an event and retry the delivery separately.

**6. Retry advice must be OUTSIDE transaction advice.** `@Retryable` on a `@Transactional` method works because retry ordering places it outermost by default — so each attempt gets a *fresh* transaction. **If it were inside, you'd retry within a rolled-back transaction, which cannot succeed.** Verify the ordering (Lesson 10 §3); it's a subtle, high-value detail.

---

## 3. Circuit breakers

**Retries help with brief blips. They make sustained failures worse** — you keep hammering a dead service, holding threads, and delaying the error the caller needs to see.

**A circuit breaker fails fast when a dependency is clearly broken.**

```
                    failure rate > threshold
        ┌────────┐ ─────────────────────────▶ ┌────────┐
        │ CLOSED │                            │  OPEN  │
        │ (calls │                            │ (calls │
        │  pass) │ ◀───────────────────────── │  FAIL  │
        └────────┘   success rate recovered   │ FAST)  │
             ▲                                └────┬───┘
             │                                     │ after waitDuration
             │        ┌───────────┐                │
             └────────│ HALF_OPEN │◀───────────────┘
               enough │ (a few    │
              succeed │  trial    │ ── still failing ──▶ back to OPEN
                      │  calls)   │
                      └───────────┘
```

**Why "fail fast" is a feature, not a compromise:** while the circuit is open, calls return immediately instead of blocking a thread for 10 seconds. **Your threads stay available, your other endpoints keep working, and the struggling dependency gets breathing room to recover.** A circuit breaker protects *both* sides.

```xml
<dependency><groupId>io.github.resilience4j</groupId>
  <artifactId>resilience4j-spring-boot3</artifactId></dependency>
<dependency><groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-aop</artifactId></dependency>
```

```yaml
resilience4j:
  circuitbreaker:
    configs:
      default:
        sliding-window-type: COUNT_BASED
        sliding-window-size: 20
        minimum-number-of-calls: 10           # ★ don't trip on the first 2 failures
        failure-rate-threshold: 50            # % failures to open
        slow-call-duration-threshold: 3s
        slow-call-rate-threshold: 80          # ★ slow counts as failed
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 3
        automatic-transition-from-open-to-half-open-enabled: true
        register-health-indicator: true
        record-exceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
          - org.springframework.web.client.HttpServerErrorException
        ignore-exceptions:                     # ★ 4xx are not the dependency's fault
          - org.springframework.web.client.HttpClientErrorException
    instances:
      slack:
        base-config: default
      checkTarget:                             # ★ per-monitor, see §3.1
        base-config: default
        failure-rate-threshold: 80
        wait-duration-in-open-state: 60s
```

```java
@Service
public class SlackNotifier {

    @CircuitBreaker(name = "slack", fallbackMethod = "fallback")
    @Retry(name = "slack")                     // ★ ordering matters — see below
    @TimeLimiter(name = "slack")
    public CompletableFuture<Void> notify(Incident incident) { }

    private CompletableFuture<Void> fallback(Incident incident, Throwable t) {
        if (t instanceof CallNotPermittedException) {
            log.warn("Slack circuit OPEN — queuing incident {} for later", incident.getId());
        }
        deadLetterRepository.save(DeadLetter.from(incident, t));   // ★ preserve the work
        return CompletableFuture.completedFuture(null);
    }
}
```

**The decorator order in Resilience4j (outermost → innermost) is:**
```
Retry ( CircuitBreaker ( RateLimiter ( TimeLimiter ( Bulkhead ( your call )))))
```

**Retry is OUTSIDE CircuitBreaker, deliberately.** So each retry attempt is evaluated by the breaker, and once the breaker opens, further retries fail instantly instead of hammering the dependency. **If retry were inside, you'd retry three times against a known-dead service on every call.** Understanding *why* that order is correct is a genuinely advanced point.

**`minimum-number-of-calls` matters:** without it, two failures out of two calls is a 100% failure rate and the circuit opens on a fluke. Set it so the rate is statistically meaningful.

**`slow-call-rate-threshold` is underrated:** a dependency that responds in 30 seconds is functionally down even though it never errors. Counting slow calls as failures is what catches that — and it's the case pure error-rate breakers miss.

### 3.1 The Vigil-specific design point

**Vigil calls thousands of *different* customer URLs.** A single shared circuit breaker would be wrong in both directions:
- One broken customer endpoint would open the circuit and **stop checking everyone else** — the exact opposite of what a monitoring tool must do
- Conversely, no breaker at all means slow targets consume all your check capacity

**The design: bulkhead the check workers (bounded concurrency), apply per-target timeouts, and use a per-target circuit breaker keyed on the monitor** — so a persistently dead endpoint is checked less aggressively (with backoff) rather than blocking capacity, while every other monitor is unaffected.

**Being able to reason about *where* the failure boundary belongs — rather than applying a breaker uniformly — is exactly the judgment senior interviews probe for.** And it's a genuinely interesting design problem specific to this project.

---

## 4. Bulkheads and rate limiters

### Bulkhead — isolate failure domains

Named after ship compartments: a hull breach floods one compartment, not the vessel.

```yaml
resilience4j:
  bulkhead:
    instances:
      slack:      { max-concurrent-calls: 10, max-wait-duration: 100ms }
      email:      { max-concurrent-calls: 20 }
  thread-pool-bulkhead:
    instances:
      checks:     { core-thread-pool-size: 10, max-thread-pool-size: 50, queue-capacity: 200 }
```

**The problem it solves:** without a bulkhead, all 200 Tomcat threads can end up blocked on one slow dependency, so **an unrelated endpoint that touches nothing but the database also fails.** With a bulkhead, at most 10 threads can be waiting on Slack; the other 190 keep serving everything else.

**This is the pattern that prevents "one dependency broke and the whole service went down."** Say it that way — it's more memorable than the definition.

| Type | Mechanism |
|---|---|
| **Semaphore bulkhead** | Caps concurrent calls on the *caller's* thread. Cheap. ✅ Default. |
| **Thread-pool bulkhead** | A dedicated pool per dependency. Full isolation, more overhead. |

### Rate limiter — respect other people's limits

```yaml
resilience4j:
  ratelimiter:
    instances:
      slack:
        limit-for-period: 1        # Slack webhooks: ~1 request/second
        limit-refresh-period: 1s
        timeout-duration: 5s
```

Two distinct concerns worth separating:
- **Outbound** (this): don't exceed a third party's quota and get banned
- **Inbound** (Lesson 17): protect yourself from clients

**Resilience4j's rate limiter is per-instance.** With 5 replicas at 1/s each, you send 5/s and still get throttled. For a genuine cluster-wide limit you need Redis (Bucket4j) — the same distributed-state problem as caching and scheduling. **Noticing that pattern repeat across three lessons is the mark of someone who's internalized the material.**

---

## 5. Idempotency — the pattern that makes everything else safe

**Every resilience mechanism assumes idempotency.** Retries, at-least-once message delivery, load balancer retries, mobile clients retrying on timeout — all of them can deliver the same operation twice.

**The killer case:** a request succeeds, but the response is lost on the way back. The client sees a timeout and retries. **The operation happened; the client doesn't know.** No amount of server-side correctness prevents that — it's a property of the network.

### Idempotency keys

```java
@PostMapping("/monitors")
public ResponseEntity<MonitorResponse> create(
        @RequestHeader(value = "Idempotency-Key", required = false) String idempotencyKey,
        @Valid @RequestBody CreateMonitorRequest request) {
    return ResponseEntity.status(CREATED).body(service.create(request, idempotencyKey));
}
```

```java
@Service
@RequiredArgsConstructor
public class IdempotencyService {

    private final StringRedisTemplate redis;
    private static final Duration TTL = Duration.ofHours(24);

    public <T> T execute(String key, String requestFingerprint, Class<T> type, Supplier<T> op) {
        if (key == null) return op.get();                     // opt-in

        String redisKey = "idem:" + key;
        String existing = redis.opsForValue().get(redisKey);

        if (existing != null) {
            var record = parse(existing);
            // ★ Same key + DIFFERENT body = a client bug. Fail loudly rather than
            //   returning a response for an operation they didn't request.
            if (!record.fingerprint().equals(requestFingerprint)) {
                throw new IdempotencyKeyReusedException(key);   // → 422
            }
            if (record.isInProgress()) throw new RequestInProgressException(key);  // → 409
            return deserialize(record.response(), type);        // ★ replay the ORIGINAL response
        }

        // Claim the key atomically before doing the work
        boolean claimed = Boolean.TRUE.equals(redis.opsForValue()
                .setIfAbsent(redisKey, inProgress(requestFingerprint), TTL));
        if (!claimed) throw new RequestInProgressException(key);

        try {
            T result = op.get();
            redis.opsForValue().set(redisKey, completed(requestFingerprint, result), TTL);
            return result;
        } catch (Exception e) {
            redis.delete(redisKey);        // ★ allow a genuine retry after a failure
            throw e;
        }
    }
}
```

**Three details that make this correct rather than approximate:**
1. **Store and replay the original response**, don't just skip the work. The client needs the created resource's ID, and a retry must return the same body as the original.
2. **Fingerprint the request body.** The same key with a different payload is a client bug — returning the first response would silently discard their second request. Fail with 422.
3. **Claim atomically with `setIfAbsent`.** Otherwise two concurrent retries both see "no record" and both execute.

**Stripe's API is the canonical reference here**, and mentioning that is a good signal — it shows you've looked at how a company that cannot afford double-charging actually does it.

### Natural idempotency — better than a key when you can get it

```java
// ✅ Idempotent by construction — a unique constraint makes the second attempt a no-op
@Transactional
public Monitor create(UUID orgId, CreateMonitorRequest req) {
    return repo.findByOrgIdAndName(orgId, req.name())
            .orElseGet(() -> repo.save(new Monitor(orgId, req)));
}

// ✅ Upsert — the database enforces it
INSERT INTO check_result (id, monitor_id, checked_at, status)
VALUES (?, ?, ?, ?)
ON CONFLICT (monitor_id, checked_at) DO NOTHING;

// ✅ State-transition guards — the second call is a no-op
public void acknowledge(UUID incidentId, String user) {
    Incident i = load(incidentId);
    if (i.getStatus() != OPEN) return;          // already acknowledged → nothing to do
    i.acknowledge(user);
}
```

**Designing for natural idempotency is better than bolting on keys**, because it can't be forgotten and needs no extra infrastructure. **A unique constraint plus `ON CONFLICT DO NOTHING` is idempotency the database enforces for you** — and Vigil uses exactly that for check results, where at-least-once delivery from the outbox means duplicates are expected.

### Idempotent consumers (for Kafka — Lesson 37)

```java
@KafkaListener(topics = "monitor-events")
@Transactional
public void consume(MonitorEvent event) {
    if (processedEventRepository.existsById(event.eventId())) return;   // ★ dedupe
    handle(event);
    processedEventRepository.save(new ProcessedEvent(event.eventId(), Instant.now()));
    // ★ Same transaction: the dedupe record and the side effect commit atomically
}
```
**The dedupe record and the business change must be in one transaction**, or you can process the event and crash before recording it — and then process it again. That atomicity requirement is the same insight as the outbox pattern (Lesson 32), viewed from the consuming side.

---

## 6. Graceful degradation and fallbacks

```java
@CircuitBreaker(name = "uptimeService", fallbackMethod = "cachedUptime")
public Uptime uptime(UUID monitorId) { return expensiveCalculation(monitorId); }

private Uptime cachedUptime(UUID monitorId, Throwable t) {
    return cache.getStale(monitorId).orElse(Uptime.unknown());   // ★ stale beats nothing
}
```

**A good fallback is honest about being degraded.** Options, best to worst:

1. **Serve stale data, labelled as stale.** A status page showing "as of 2 minutes ago" is far better than an error page — especially during an incident, when it's most needed.
2. **Serve a partial response.** Return the monitors and omit the uptime percentages rather than failing the whole request.
3. **Queue the work and return 202 Accepted.** For writes: "we've accepted this, it will be processed."
4. **Return a clear, actionable error.** Not a stack trace, and not a lie.

**❌ The bad fallback: returning a plausible default that hides the failure.** `return Uptime.of(100)` when the calculation failed means your dashboard shows everything green during an outage. **A fallback that lies is worse than an error** — it removes the operator's ability to know something is wrong. That's the principle to articulate.

---

## 7. Build this — Vigil's resilience layer

**1. Timeouts everywhere.** Audit every outbound call: `RestClient` connect + read, Hikari `connection-timeout`, `statement_timeout` via `connection-init-sql`, Redis `timeout`. **Write them in a table with a justification for each value** — that document is a great thing to have in the repo.

**2. ⭐ Prove your timeout fires** (revisiting Lesson 30 §6). WireMock with `withFixedDelay(60_000)`, then assert the call fails in ~10s, not 60s. **Then set the read timeout to 0 (infinite) and watch the test hang** — that hang is a production outage in miniature.

**3. Retries with jitter** on the Slack notifier: `retryFor` transient exceptions only, `noRetryFor` 4xx, exponential backoff with `random = true`, and a `@Recover` writing to a dead-letter table. **Log every attempt** and confirm the delays are non-uniform.

**4. Prove that 4xx isn't retried.** WireMock returning 400. Count the requests: **exactly 1.** Then remove `noRetryFor` and count again: 3. **You just tripled load on a request that could never succeed.**

**5. Circuit breaker on Slack.**
```yaml
resilience4j.circuitbreaker.instances.slack:
  sliding-window-size: 10
  minimum-number-of-calls: 5
  failure-rate-threshold: 50
  wait-duration-in-open-state: 20s
```
Then: WireMock returns 500 → send 10 notifications → **confirm the circuit opens** (`/actuator/health` shows it, and further calls fail instantly with `CallNotPermittedException`). Fix WireMock to return 200, wait 20s, and **watch HALF_OPEN → CLOSED** in the logs. **Watching that state machine transition on your own service is what makes the concept stick.**

**6. Measure the fail-fast benefit.** With the circuit CLOSED and a 10s-delayed WireMock, time 20 concurrent notification attempts. Then with the circuit OPEN, time them again. **The difference — 10s vs ~0ms per call — is the thread-hours you just saved**, and it's the whole point of the pattern.

**7. Bulkhead, demonstrated.** Add `max-concurrent-calls: 5` to the Slack bulkhead. Make WireMock slow. Then hit an **unrelated** endpoint (`/api/v1/monitors`) while 50 notifications are in flight. **Confirm the unrelated endpoint stays fast.** Then remove the bulkhead and confirm it degrades. **This experiment demonstrates failure isolation better than any diagram.**

**8. Slow-call detection.** Configure `slow-call-duration-threshold: 2s` and `slow-call-rate-threshold: 80`, then make WireMock return **200 OK after 5 seconds**. **Confirm the circuit opens even though nothing errored.** That's the case a naive error-rate breaker misses entirely.

**9. Idempotency for POST.** Implement `IdempotencyService` with Redis. Then:
- Same key, same body, twice → **one monitor created, both responses identical**
- Same key, different body → **422**
- Concurrent requests with the same key → **one succeeds, one gets 409**
- Key expires after 24h → the operation can run again

**10. Natural idempotency for check results.** `UNIQUE (monitor_id, checked_at)` plus `ON CONFLICT DO NOTHING`. Then intentionally deliver the same outbox message twice and confirm exactly one row. **This is the pattern that makes at-least-once delivery safe** (Lesson 32).

**11. Graceful degradation, end to end.** Kill Redis, kill WireMock (as a stand-in for Slack), and confirm:
- The status page still renders (from the database, slower)
- Monitors are still checked
- Notifications go to the dead-letter table instead of vanishing
- `/actuator/health` shows the degraded components
- **No endpoint returns 500**

**A demo where you kill two dependencies live and the application keeps working — with honest degradation — is one of the most persuasive things you can show an interviewer.** It's also a great README section.

**12. Watch the metrics.** With Actuator + Micrometer:
```
resilience4j_circuitbreaker_state
resilience4j_circuitbreaker_calls_seconds_count{kind="failed"}
resilience4j_bulkhead_available_concurrent_calls
resilience4j_retry_calls_total{kind="successful_with_retry"}
```
That last one is genuinely useful in production: **a rising "successful with retry" rate means a dependency is degrading before it fails outright** — an early warning signal (Lesson 35).

---

## 8. Interview traps

**Q1. What's the most important resilience practice?**
Timeouts on every network call. Defaults are effectively infinite, so a hung upstream blocks threads indefinitely, exhausting the thread and connection pools until every endpoint — including health checks — fails. Your service goes down because someone else's is slow.

**Q2. What timeouts must you configure?**
HTTP connect and read, connection-acquisition, database statement and transaction timeouts, connection-pool wait, Redis command timeout, and server request timeout. Values must **decrease with depth** (a timeout budget), derived from measured p99 rather than guessed. Note that a read timeout is per-read, so you also want an overall request budget (`TimeLimiter`) against slow-drip responses.

**Q3. When should you retry, and when must you not?**
Retry transient failures: connection errors, timeouts, 502/503/504, 429 (honouring `Retry-After`), deadlocks, optimistic-lock conflicts. Never retry 4xx — the request is wrong and will be wrong again, so you triple load and mask the real error. And never retry a non-idempotent operation without an idempotency key.

**Q4. Why is jitter essential in backoff?**
Without it, all clients that failed together retry together, producing a synchronized **retry storm** that knocks the recovering dependency back over as soon as it comes up. Jitter spreads retries so recovery can actually happen.

**Q5. What is retry amplification?**
Retries at multiple layers multiply: three chained services each retrying three times means 27 requests to the deepest one — the most load exactly when it's least able to serve it. Retry at **one** layer, ideally the outermost that can meaningfully recover.

**Q6. Explain a circuit breaker's states.**
CLOSED (calls pass, failures counted) → **OPEN** when the failure rate over a sliding window exceeds the threshold (calls fail immediately, no thread blocked) → after `waitDuration`, **HALF_OPEN** admits a few trial calls → success closes it, failure reopens it. It protects both sides: your threads stay free, and the struggling dependency gets breathing room.

**Q7. Why must Retry wrap CircuitBreaker rather than the reverse?**
So each retry attempt passes through the breaker, and once the breaker opens, retries fail instantly instead of hammering a known-dead dependency. Reversed, you'd retry three times per call against a service you already know is down.

**Q8. What's a bulkhead and what problem does it solve?**
A cap on concurrent calls per dependency. Without it, all your request threads can end up blocked on one slow dependency, so unrelated endpoints fail too. With it, only N threads can wait on that dependency and the rest of the service keeps working — it prevents "one dependency broke and the whole service went down."

**Q9. Why does idempotency matter for resilience?**
Every retry mechanism can deliver an operation twice — and the unavoidable case is a request that succeeds while its response is lost, so the client legitimately retries something that already happened. Without idempotency, resilience creates duplicates.

**Q10. How do you implement an idempotency key?**
Client sends `Idempotency-Key`; the server claims it atomically (`SETNX` with a TTL), executes, and stores the **response** so a retry replays the original. Fingerprint the request body so the same key with a different payload fails with 422 rather than silently returning the wrong response. Delete the key on failure so a genuine retry can proceed. Stripe's API is the reference design.

**Q11. What's better than an idempotency key?**
Natural idempotency: a unique constraint plus `ON CONFLICT DO NOTHING`, a find-or-create, or a state-transition guard that makes a second call a no-op. The database enforces it, so it can't be forgotten and needs no extra infrastructure.

**Q12. What makes a good fallback?**
Honesty about degradation: stale data clearly labelled as stale, a partial response, or 202 Accepted with queued work. **A fallback that returns a plausible fake value — 100% uptime when the calculation failed — is worse than an error**, because it removes the operator's ability to know something is wrong.

**Q13. What does `slow-call-rate-threshold` catch that error rate doesn't?**
A dependency that responds successfully but takes 30 seconds. It never errors, so a pure error-rate breaker stays closed while your threads pile up. Counting slow calls as failures is what detects functional-but-unusable dependencies.

---

## 9. Retention check

1. Trace the full chain from "no read timeout" to "total outage."
2. Why is jitter not optional, and what's the name of the failure it prevents?
3. Explain the retry/circuit-breaker ordering and why the reverse is wrong.
4. Give the one failure case that *no* server-side correctness can prevent, and the pattern that handles it.
5. Why is a unique constraint better than an idempotency key when you can use it?
6. Why is a fallback returning 100% uptime worse than an error?

---

**Next:** [Lesson 35 — Observability](35-observability.md) — metrics, structured logs, traces, and knowing your system is broken before your customers tell you.

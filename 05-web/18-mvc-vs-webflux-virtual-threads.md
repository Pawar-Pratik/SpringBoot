# Lesson 18 — MVC vs WebFlux vs virtual threads

> **Why this lesson exists:** "Should we use reactive?" is a question you will be asked, and the industry's answer changed materially when virtual threads shipped. Most candidates either say "reactive is faster" (wrong and shallow) or can't discuss it at all. You're going to be able to reason about it from the thread model up — and give the calibrated answer that senior engineers give.

**Time:** ~60 minutes · **Prereq:** Lesson 14

---

## 1. The two problems reactive was invented for

### Problem 1 — Thread-per-request doesn't scale to high concurrency

From Lesson 14: 200 threads, one per in-flight request, blocked while waiting on I/O.

```
Request → thread assigned → DB query (50ms, thread BLOCKED, doing nothing)
                          → HTTP call to Slack (200ms, thread BLOCKED)
                          → serialize response (2ms of actual CPU work)
       → thread released

Thread was held for 252ms and did 2ms of work. It was IDLE 99.2% of the time.
```

Each platform thread costs ~1MB of stack (reserved virtual memory) plus kernel scheduling overhead. 10,000 concurrent connections would need 10,000 threads — ~10GB of stack and enough context switching to bury the CPU. **It doesn't work.**

### Problem 2 — There's no backpressure

A producer faster than a consumer (a firehose of check results into a slow database) overflows queues and eventually the heap. Reactive Streams standardizes **demand signalling**: the consumer says "send me 10 more," so the producer can't outrun it. That's a genuine, distinct contribution — and it's the part people forget when they focus only on thread counts.

---

## 2. The reactive model

```
   MVC (thread-per-request)                  WebFlux (event loop)

   200 threads                               ~N threads (N = CPU cores)
   ┌──────┐                                  ┌──────┐
   │thread│ ── blocked on DB ──── waiting     │ loop │ ─ start DB call, register callback,
   │thread│ ── blocked on HTTP ── waiting     │ loop │   MOVE ON to the next request
   │thread│ ── blocked on DB ──── waiting     └──────┘
   │ ...  │                                   when the DB responds, the OS notifies the
   │thread│                                   loop, which resumes that request's pipeline
   └──────┘
   250MB of stacks, mostly idle               a few MB, always busy
```

The thread is **never blocked**. Instead of waiting, it registers a callback and processes something else. Same hardware, far more concurrent connections.

```java
// MVC — imperative, blocking
@GetMapping("/{id}")
public MonitorResponse get(@PathVariable UUID id) {
    Monitor m = repository.findById(id).orElseThrow();     // blocks here
    Uptime u = uptimeClient.fetch(id);                     // blocks here
    return mapper.toResponse(m, u);
}

// WebFlux — declarative, non-blocking
@GetMapping("/{id}")
public Mono<MonitorResponse> get(@PathVariable UUID id) {
    return repository.findById(id)                          // Mono<Monitor>
            .switchIfEmpty(Mono.error(new NotFoundException(id)))
            .zipWith(uptimeClient.fetch(id))                // both calls CONCURRENTLY
            .map(t -> mapper.toResponse(t.getT1(), t.getT2()));
}
```

Note what the reactive version gets for free: `zipWith` runs both calls **concurrently**. In MVC you'd need `CompletableFuture` and an executor. That composability is real and underrated.

### `Mono` and `Flux`

```java
Mono<T>      // 0 or 1 element, then complete or error
Flux<T>      // 0..N elements, then complete or error
```

```java
Mono.just(monitor);
Mono.empty();
Mono.error(new NotFoundException());
Flux.fromIterable(list);
Flux.interval(Duration.ofSeconds(1));

flux.map(m -> toDto(m))                     // transform each
    .filter(m -> m.isFailing())
    .flatMap(m -> enrichAsync(m))           // async transform, concurrent, order not preserved
    .concatMap(m -> enrichAsync(m))         // async transform, order preserved, sequential
    .take(10)
    .timeout(Duration.ofSeconds(5))
    .retryWhen(Retry.backoff(3, Duration.ofMillis(200)))
    .onErrorResume(e -> Flux.empty())
    .doOnNext(m -> log.debug("{}", m))      // side effects
    .subscribeOn(Schedulers.boundedElastic())
    .subscribe();                            // ★ NOTHING HAPPENS UNTIL SUBSCRIBE
```

**Nothing executes until subscription.** Building a `Mono` and not returning it is a no-op — the #1 beginner bug, and it fails *silently*. Spring subscribes for you when you return the publisher from a controller.

**`flatMap` vs `concatMap`** is the interview detail: `flatMap` interleaves and doesn't preserve order (fast); `concatMap` is sequential and ordered. Using `flatMap` where order matters is a subtle, hard-to-reproduce bug.

### The absolute rule: one blocking call poisons everything

```java
@GetMapping("/{id}")
public Mono<MonitorResponse> get(@PathVariable UUID id) {
    return Mono.fromCallable(() -> jpaRepository.findById(id))   // ❌❌ BLOCKING JPA CALL
            .map(mapper::toResponse);
}
```

You just blocked an event-loop thread. With ~8 loop threads, **8 concurrent slow requests freeze your entire application** — every endpoint, for every user. This is dramatically worse than MVC's degradation, which is why reactive is *unforgiving*: it demands that your whole stack be non-blocking.

**Which means the whole stack must change:**

| Blocking (MVC) | Reactive (WebFlux) |
|---|---|
| JDBC | **R2DBC** (and it lacks JPA — no ORM, no lazy loading, fewer features) |
| Spring Data JPA / Hibernate | Spring Data R2DBC (no ORM) |
| `RestTemplate`/`RestClient` | `WebClient` |
| Jedis | Lettuce (already reactive) |
| Servlet API, `HttpServletRequest` | `ServerWebExchange` |
| `ThreadLocal` (MDC, SecurityContext, request scope) | **Reactor Context** — a different mechanism entirely |
| Any blocking library, ever | must be wrapped on `boundedElastic()`, or replaced |

**That last row deserves emphasis.** `ThreadLocal` doesn't work when a request hops threads mid-pipeline, so MDC logging, `SecurityContextHolder`, and request-scoped beans all need reactive equivalents. **This is where the real cost of reactive lives** — not in learning `Mono`, but in every library, debugging technique, and piece of muscle memory that assumes a thread identity.

**If you use R2DBC, you give up JPA.** No `@OneToMany`, no lazy loading, no dirty checking, no `EntityManager`. For a schema-light append-heavy service that's fine; for a rich domain model it's a large loss. Being able to state this specific trade-off is what a real answer sounds like.

### The costs, honestly

**1. Stack traces become useless.**
```
at reactor.core.publisher.MonoFlatMap$FlatMapMain.onNext(MonoFlatMap.java:158)
at reactor.core.publisher.FluxMapFuseable$MapFuseableSubscriber.onNext(FluxMapFuseable.java:129)
... 60 more frames of Reactor internals ...
```
Nothing tells you *which request* or *which of your lines* failed. `Hooks.onOperatorDebug()` helps but costs performance; `ReactorDebugAgent` is better. Still: debugging is materially harder, and this is the complaint you'll hear most from teams who adopted it.

**2. Debuggers barely work.** Breakpoints land in Reactor internals. Print-debugging via `.log()` and `doOnNext` becomes your main tool.

**3. The learning curve is real and it's a *team* cost.** One person who knows Reactor and four who don't produces code with subtle bugs — a forgotten `subscribe()`, a `block()` sneaked in "just here," a `flatMap` where order mattered. **The bus factor argument is legitimate and worth raising.**

**4. Throughput is not automatically better.** For a typical CRUD service at moderate concurrency, WebFlux often shows *no throughput improvement* — sometimes slightly worse, because of per-operator overhead. Its win is **resource efficiency at high concurrency**: many more connections per unit of memory. If you don't have that problem, you're paying complexity for nothing.

---

## 3. Virtual threads — the plot twist

Java 21 (Sept 2023) delivered virtual threads (Project Loom), and they change the calculus substantially.

A **virtual thread** is a JVM-managed thread multiplexed onto a small pool of OS carrier threads. When it blocks, the JVM **unmounts** it from its carrier and mounts another. Cost: a few hundred bytes, no kernel involvement.

```properties
# Java 21+, Spring Boot 3.2+. That's the entire migration.
spring.threads.virtual.enabled=true
```

**What this buys:**

```java
// Ordinary, readable, blocking code:
@GetMapping("/{id}")
public MonitorResponse get(@PathVariable UUID id) {
    Monitor m = repository.findById(id).orElseThrow();     // blocks the VIRTUAL thread —
    Uptime u = uptimeClient.fetch(id);                     // the carrier thread is FREE
    return mapper.toResponse(m, u);
}
```

The virtual thread blocks; the OS thread doesn't. You get **reactive-like scalability with imperative code**: normal stack traces, working debuggers, `ThreadLocal` intact, JPA intact, your entire existing ecosystem intact.

> **The framing that lands in interviews:** *"Reactive programming solved thread scarcity by changing the programming model. Virtual threads solved it by making threads cheap. For most applications the second is a far better trade, because it costs no complexity."*

**The caveats — state these, or you sound like you've only read a headline:**

1. **`synchronized` pinning.** In Java 21, a virtual thread blocking inside a `synchronized` block **pins** its carrier thread, negating the benefit. Fixed in later versions (Java 24/25), but on Java 21 you must audit hot paths and prefer `ReentrantLock`. Some older libraries are affected.
2. **Native/JNI calls pin** the carrier and always will.
3. **`ThreadLocal` at scale.** A million virtual threads each with thread-locals is real memory. `ScopedValue` (JEP 446+) is the intended successor.
4. **You still need pooling for downstreams.** ⭐ Virtual threads don't create database connections. A million virtual threads sharing 10 Hikari connections means 999,990 waiting on the pool. **Virtual threads relocate the bottleneck from your thread pool to your dependencies** — and *that's the point*, because now your bottleneck is a real resource limit you should be managing explicitly rather than an accidental artifact of thread count.
5. **No backpressure.** Reactive Streams' demand signalling has no virtual-thread equivalent. For streaming pipelines where a fast producer can overwhelm a slow consumer, reactive still has a genuine advantage.
6. **Don't pool virtual threads.** They're cheap and disposable — pooling them defeats the design. `Executors.newVirtualThreadPerTaskExecutor()`, not a fixed pool.

---

## 4. The decision framework

### Choose **MVC + virtual threads** (the default in 2026) when:
- You're building a typical REST API / CRUD service — **this is most software**
- You use JPA, or any blocking library
- Your team doesn't already know Reactor
- You want debuggable stack traces and working profilers
- Concurrency is in the hundreds-to-tens-of-thousands range

### Choose **WebFlux** when:
- **You genuinely need streaming with backpressure** — SSE/WebSocket fan-out to many clients, or a pipeline where a fast producer must be throttled by a slow consumer
- **You're an I/O-bound aggregator/gateway** — mostly forwarding and composing calls with little business logic. This is WebFlux's sweet spot, and it's exactly why Spring Cloud Gateway is built on it.
- **Extreme connection counts with tiny per-connection work** — 100k+ idle-ish long-lived connections
- Your whole stack is already non-blocking (R2DBC or NoSQL, Lettuce, `WebClient`)
- **The team already knows Reactor** — this is a bigger factor than any technical one

### Choose **neither / rethink** when:
- You're CPU-bound. Neither model helps; you need more CPU, better algorithms, or caching.
- Your real problem is a missing index or an N+1 query. **Far more "we need reactive for performance" conversations are actually undiagnosed database problems.** Saying this out loud is one of the most senior-sounding things in this lesson.

### The answer to give in an interview

> *"I'd default to MVC with virtual threads. It gives me the scalability benefit that motivated reactive, without giving up JPA, readable stack traces, ThreadLocal-based context, or my team's existing skills. I'd reach for WebFlux when I specifically need backpressure for streaming, or when the service is an I/O-bound gateway with very high connection counts and little business logic — Spring Cloud Gateway is the archetype. And before either, I'd check whether the actual bottleneck is a database query, because it usually is."*

That answer demonstrates: you know both models, you know what changed, you know the costs, and you diagnose before prescribing.

### Comparison table

| | MVC (platform threads) | MVC + virtual threads | WebFlux |
|---|---|---|---|
| Programming model | Imperative | Imperative | Declarative/functional |
| Concurrency ceiling | ~200–800 | Very high | Very high |
| Memory per request | ~1MB stack | ~KB | Minimal |
| JPA/Hibernate | ✅ | ✅ | ❌ (R2DBC, no ORM) |
| Stack traces | ✅ Clear | ✅ Clear | ❌ Reactor noise |
| Debugger | ✅ | ✅ | ⚠️ Painful |
| `ThreadLocal` (MDC, Security) | ✅ | ✅ | ❌ Reactor Context |
| Backpressure | ❌ | ❌ | ✅ |
| Learning curve | Low | Low | **High** |
| Java version | any | **21+** | 8+ |
| Ecosystem | Everything | Everything | Reactive libs only |

---

## 5. `WebClient` and `RestClient` — use them regardless

Even in an MVC app, you'll use these for outbound calls. Vigil's entire purpose is outbound HTTP.

### `RestClient` (Boot 3.2+) — the modern blocking client

```java
@Bean
RestClient checkClient(RestClient.Builder builder) {
    return builder
            .requestFactory(ClientHttpRequestFactories.get(
                    ClientHttpRequestFactorySettings.DEFAULTS
                            .withConnectTimeout(Duration.ofSeconds(5))
                            .withReadTimeout(Duration.ofSeconds(10))))
            .defaultHeader("User-Agent", "Vigil/1.0")
            .build();
}
```
```java
CheckOutcome outcome = checkClient.get()
        .uri(monitor.getUrl())
        .retrieve()
        .onStatus(HttpStatusCode::is5xxServerError,
                  (req, res) -> { throw new UpstreamFailure(res.getStatusCode()); })
        .toEntity(String.class)
        .getStatusCode()
        .value() == 200 ? CheckOutcome.up() : CheckOutcome.down();
```

**`RestClient` has `WebClient`'s fluent API with blocking semantics.** It's the recommended replacement for `RestTemplate` in new code — `RestTemplate` is in maintenance mode (not deprecated, but no new features).

### `WebClient` — non-blocking, usable from MVC too

```java
Mono<String> body = webClient.get().uri(url).retrieve().bodyToMono(String.class);
String blocking = body.block();       // ⚠️ fine in MVC, NEVER on a WebFlux event loop
```

### HTTP Interfaces — the nicest option (Spring 6+)

```java
public interface SlackApi {
    @PostExchange("/services/{token}")
    void postMessage(@PathVariable String token, @RequestBody SlackMessage message);
}
```
```java
@Bean
SlackApi slackApi(RestClient.Builder builder) {
    RestClient client = builder.baseUrl("https://hooks.slack.com").build();
    return HttpServiceProxyFactory
            .builderFor(RestClientAdapter.create(client))
            .build()
            .createClient(SlackApi.class);
}
```

**Declarative HTTP clients, built into Spring** — Feign's ergonomics without the extra dependency. Spring generates a proxy implementing your interface (Lesson 09's mechanism again — proxies everywhere). **Vigil uses this for Slack and webhook delivery.** Mentioning HTTP interfaces instead of Feign shows you're current.

### The timeout rule — non-negotiable

**Every outbound HTTP call must have a connect timeout and a read timeout.** The defaults are effectively infinite:

```
No read timeout + a hung upstream = a thread blocked forever
                                  = pool exhaustion
                                  = total outage caused by someone else's bad day
```

This is Lesson 14's thread starvation with an external trigger. It's the single most common cause of cascading production failures in service-oriented systems. Lesson 34 adds retries, circuit breakers and bulkheads on top.

---

## 6. Build this

**1. Measure the difference yourself.** Add:
```java
@GetMapping("/blocking-io")
public String blockingIo() throws InterruptedException {
    Thread.sleep(200);                       // stands in for a DB or HTTP call
    return Thread.currentThread().toString();
}
```
Load test with 500 concurrent requests. Use `hey`, `k6`, or `ab`:
```bash
hey -n 5000 -c 500 http://localhost:8080/blocking-io
```
Run three configurations and record p50/p95/p99 and errors:
- `server.tomcat.threads.max=200` (default)
- `server.tomcat.threads.max=20` — watch queueing destroy p99
- `spring.threads.virtual.enabled=true` — watch it flatten

**Write the numbers down.** "I load tested this and here's what happened" is enormously more credible in an interview than "virtual threads are better." And this is a genuinely great thing to put in Vigil's README.

**2. Verify the thread type.** Return `Thread.currentThread().toString()` and compare: `Thread[#42,http-nio-8080-exec-1,5,main]` vs `VirtualThread[#123]/runnable@ForkJoinPool-1-worker-3`. Note the **carrier thread** named in the virtual case — that's the multiplexing, visible.

**3. Feel the reactive model in isolation.** Add `spring-boot-starter-webflux` to a *separate* scratch project (mixing both in one app is confusing and generally a bad idea in production):
```java
@GetMapping("/flux")
public Flux<String> stream() {
    return Flux.interval(Duration.ofSeconds(1))
               .map(i -> "tick " + i)
               .take(5);
}
```
```bash
curl -N http://localhost:8081/flux
```
Watch values arrive one per second — **streaming, not a batched response.** That's the thing MVC can't do as naturally.

**4. Feel the reactive pain.** In that project, put `Thread.sleep(5000)` inside a `map()`. Fire 10 concurrent requests. **Watch the entire application freeze** — every endpoint, not just that one. Then compare: do the same in MVC and only that endpoint's requests queue. **This one experiment teaches you more about "should we use reactive?" than any blog post.**

**5. Forget to subscribe.**
```java
@GetMapping("/oops")
public String oops() {
    Mono.fromCallable(() -> { log.info("this never runs"); return 1; });   // not returned
    return "done";
}
```
Nothing logs. No error. **That silent failure mode is the reactive tax.**

**6. Build Vigil's HTTP checker for real** — this is production code, not an exercise:
```java
@Component
@RequiredArgsConstructor
public class HttpCheckExecutor implements CheckExecutor {

    private final RestClient client;

    @Override public boolean supports(MonitorType type) { return type == MonitorType.HTTP; }

    @Override
    public CheckOutcome execute(Monitor m) {
        long start = System.nanoTime();
        try {
            ResponseEntity<Void> res = client.get().uri(m.getUrl()).retrieve().toBodilessEntity();
            long ms = (System.nanoTime() - start) / 1_000_000;
            return res.getStatusCode().value() == m.getExpectedStatus()
                    ? CheckOutcome.up(ms, res.getStatusCode().value())
                    : CheckOutcome.down(ms, "unexpected status " + res.getStatusCode().value());
        } catch (ResourceAccessException e) {                 // timeout / connection refused
            return CheckOutcome.down(elapsed(start), "unreachable: " + e.getMostSpecificCause());
        } catch (Exception e) {
            return CheckOutcome.down(elapsed(start), e.getClass().getSimpleName());
        }
    }
}
```
Configure a 5s connect and 10s read timeout. **Then test it against a deliberately hanging endpoint** (`Thread.sleep(60000)` in another app, or `https://httpstat.us/200?sleep=60000`) and confirm it times out rather than hanging forever. **Verifying your timeout actually fires is a step almost everyone skips** — and untested timeout config is how outages happen.

**7. HTTP interface.** Convert your Slack notification channel to the `@PostExchange` declarative style.

---

## 7. Interview traps

**Q1. When would you use WebFlux over MVC?**
When I need streaming with backpressure (SSE/WebSocket fan-out, or a fast producer that must be throttled by a slow consumer); when the service is an I/O-bound gateway/aggregator with very high connection counts and little business logic; and when the whole stack is already non-blocking. Otherwise I default to MVC with virtual threads, because it gives similar scalability without giving up JPA, stack traces, ThreadLocal context, or team familiarity.

**Q2. Is WebFlux faster than MVC?**
Not in throughput for typical workloads — sometimes slightly slower due to operator overhead. Its advantage is **resource efficiency at high concurrency**: far more concurrent connections per unit of memory. If you don't have a concurrency problem, you're buying complexity with no return.

**Q3. What happens if you block in a WebFlux handler?**
You block an event-loop thread. With ~one loop per core, a handful of concurrent blocking calls freezes the entire application for all users. Mitigation is `subscribeOn(Schedulers.boundedElastic())`, but then you've reintroduced a thread pool and lost the point. It's why reactive is all-or-nothing.

**Q4. What do virtual threads change about this debate?**
They make blocking cheap: a virtual thread unmounts from its OS carrier while blocked, so you get high concurrency with imperative code. That removes thread scarcity — reactive's main motivation — for most applications. Reactive retains a genuine edge only for backpressure/streaming.

**Q5. What are the limitations of virtual threads?**
`synchronized` blocks pinned the carrier in Java 21 (fixed in later versions; prefer `ReentrantLock` on hot blocking paths); native/JNI calls pin; `ThreadLocal` at massive scale is a memory concern (`ScopedValue` is the successor); no backpressure; and **downstream limits become the bottleneck** — a million virtual threads still share your connection pool. Also, don't pool them.

**Q6. Why can't you use JPA with WebFlux?**
JDBC is a blocking API and JPA/Hibernate is built on it, with lazy loading and dirty checking that assume thread-bound state. The reactive alternative is R2DBC via Spring Data R2DBC — which is not an ORM: no `@OneToMany`, no lazy loading, no `EntityManager`. That loss is a major, often-underestimated cost of going reactive.

**Q7. Why doesn't `ThreadLocal` work in reactive code?**
A request's pipeline can execute across multiple threads as operators switch schedulers, so thread-bound state doesn't follow it. MDC logging, `SecurityContextHolder`, and request-scoped beans all need reactive equivalents — Reactor's `Context`, propagated explicitly.

**Q8. What is backpressure and why is it reactive's unique contribution?**
Consumer-driven demand signalling: the subscriber requests N items, so a fast producer cannot overwhelm a slow consumer. Neither thread-per-request nor virtual threads provide it — they just block, which works for request/response but not for continuous streams. It's the one capability virtual threads don't replicate.

**Q9. `RestTemplate`, `RestClient`, or `WebClient`?**
`RestClient` for new blocking code — fluent API, modern, actively developed. `WebClient` for reactive (usable from MVC with `.block()`). `RestTemplate` is in maintenance mode: still supported, no new features, avoid for new code. Best of all: **HTTP Interfaces** (`@GetExchange`/`@PostExchange`) for declarative clients over either.

**Q10. What must every outbound HTTP call have?**
Connect **and** read timeouts. Defaults are effectively infinite, so one hung upstream blocks threads indefinitely and cascades into total outage. Then: retries with backoff and jitter, a circuit breaker, and ideally a bulkhead so one dependency can't consume all your capacity.

**Q11. Someone says "we need reactive, our API is slow." What do you do?**
Measure first. Slowness is usually a database problem — a missing index, an N+1 query, or a chatty call pattern — or a lack of caching, none of which reactive fixes. Reactive addresses *concurrency* limits, not *latency*. I'd look at p99 latency breakdown, query plans, and thread-pool saturation before changing programming models.

---

## 8. Retention check

1. What resource does thread-per-request waste, and roughly how much per thread?
2. Why does one blocking call in WebFlux hurt far more than one slow endpoint in MVC?
3. What exactly happens to a virtual thread when it blocks on I/O?
4. Name the new bottleneck virtual threads expose, and why that's arguably an improvement.
5. Name the one capability reactive has that virtual threads don't.
6. Give the two timeouts every HTTP client needs and the failure mode if they're missing.

---

## Module 5 complete ✅

You can now trace a request end to end, design a clean API, handle errors professionally, secure browser access, and reason about the concurrency model. **That's the full web layer.**

**Next:** [Lesson 19 — JDBC, DataSources & connection pooling](../06-data/19-jdbc-datasource-pooling.md) — the start of the hardest and most valuable module in this course. Slow down here.

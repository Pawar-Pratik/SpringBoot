# Lesson 14 — Servlets, DispatcherServlet & the request lifecycle

> **Why this lesson exists:** "Walk me through what happens when a request hits your Spring Boot app" is one of the most common senior interview questions, and it's a brutal filter — most candidates say "it goes to the controller." You're going to be able to narrate it from TCP socket to JSON response, name the components, and explain where the threads come from. That last part is what makes you able to diagnose a hung production service.

**Time:** ~70 minutes · **Prereq:** Module 2, Lesson 11

---

## 1. The idea in one sentence

> **One servlet receives every request and one thread is held for its entire duration — so Spring MVC is a front controller with pluggable strategies, and most production latency incidents are that thread being held by something slow.**

Narrating the lifecycle gets you through the interview; the thread arithmetic — `throughput = threads / latency` — is what lets you diagnose the service that is "down" while the CPU sits at 3%.

---

## 2. Foundations: HTTP and the servlet container

### HTTP, precisely enough

```http
POST /api/v1/monitors HTTP/1.1
Host: vigil.io
Content-Type: application/json
Authorization: Bearer eyJhbGci...
Accept: application/json
Content-Length: 68

{"name":"API health","url":"https://api.example.com","intervalSeconds":60}
```

```http
HTTP/1.1 201 Created
Content-Type: application/json
Location: /api/v1/monitors/7f3a2b1c-...
Content-Length: 142

{"id":"7f3a2b1c-...","name":"API health","status":"PENDING"}
```

**Status codes you must use correctly** (getting these wrong is a visible API-design failure):

| Code | Meaning | When |
|---|---|---|
| 200 OK | Success with a body | GET, PUT, PATCH |
| 201 Created | Resource created | POST — **include a `Location` header** |
| 202 Accepted | Accepted, not yet processed | Async work queued |
| 204 No Content | Success, no body | DELETE, or PUT with nothing to return |
| 400 Bad Request | Malformed/invalid input | Validation failure, bad JSON |
| 401 Unauthorized | **Not authenticated** | Missing/invalid credentials |
| 403 Forbidden | **Authenticated but not permitted** | Wrong role/tenant |
| 404 Not Found | No such resource | |
| 409 Conflict | State conflict | Duplicate name, optimistic-lock failure |
| 422 Unprocessable | Syntactically fine, semantically invalid | Contested; 400 is safer |
| 429 Too Many Requests | Rate limited | **Include `Retry-After`** |
| 500 Internal Server Error | Your bug | Never intentional |
| 502/503/504 | Upstream broken / unavailable / timeout | Dependency failures |

**401 vs 403 is the classic trap.** 401 = "I don't know who you are" (retry with credentials). 403 = "I know exactly who you are, and no" (retrying is pointless). The names are historically backwards — 401 is literally named "Unauthorized" but means unauthenticated. Interviewers ask this constantly.

**HTTP method semantics — the two properties that matter:**

| Method | Safe (no side effects) | Idempotent (N calls = 1 call) |
|---|---|---|
| GET | ✅ | ✅ |
| HEAD | ✅ | ✅ |
| PUT | ❌ | ✅ |
| DELETE | ❌ | ✅ |
| POST | ❌ | ❌ |
| PATCH | ❌ | ❌ (not guaranteed) |

**Idempotency is not academic.** Mobile clients retry on timeout; load balancers retry; message consumers redeliver. `PUT /monitors/123 {...}` twice is harmless. `POST /monitors {...}` twice creates two monitors. That's why POST needs an idempotency key (Lesson 34) — and why this table is worth memorizing.

### The Servlet API

Java standardized server-side HTTP in 1997 with the Servlet spec. A servlet is startlingly simple:

```java
public interface Servlet {
    void service(ServletRequest req, ServletResponse res);
}
```

```java
// What raw servlet programming looked like — one servlet per URL, mapped in web.xml
public class MonitorServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws IOException {
        String id = req.getParameter("id");
        resp.setContentType("application/json");
        resp.getWriter().write("{\"id\":\"" + id + "\"}");   // manual JSON. by hand. yes.
    }
}
```

You'd hand-parse parameters, hand-build JSON, hand-set status codes, and register every servlet in a `web.xml` file. **Spring MVC's entire job is to replace all of that with one servlet and some annotations.**

> **The `jakarta` rename:** Servlet types moved from `javax.servlet.*` to `jakarta.servlet.*` when Java EE became Jakarta EE (Oracle kept the `javax` trademark). **This is the single biggest breaking change in Spring Boot 3** — every library touching servlets had to re-release. If you see `javax.servlet` in code, it's Boot 2 or earlier.

### The servlet container

A **servlet container** (Tomcat, Jetty, Undertow) handles what you don't want to: TCP sockets, HTTP parsing, connection keep-alive, TLS, thread pooling, and the servlet lifecycle. Spring Boot **embeds** one in your JAR (Lesson 01).

| Container | Notes | Boot support |
|---|---|---|
| **Tomcat** | Default. Most battle-tested, best documented, most operational knowledge in the industry | `spring-boot-starter-web` |
| **Jetty** | Lighter, good for embedded/WebSocket-heavy | `spring-boot-starter-jetty` |
| **Undertow** | Non-blocking core, low memory, JBoss lineage | `spring-boot-starter-undertow` |

```xml
<!-- Switching is genuinely this easy -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
</dependency>
```

That works because of `@ConditionalOnClass` (Lesson 11): each container's auto-configuration activates when its classes are present. **Stay on Tomcat unless you have a measured reason** — the ecosystem knowledge is worth more than marginal benchmark differences.

---

## 3. Tomcat's threading model — the part that explains production incidents

This is the most operationally important section in this lesson.

```
                          ┌─────────────────────────────────────┐
   Client connections ───▶│  Acceptor thread(s)                 │
                          │  accept() on the server socket      │
                          └──────────────┬──────────────────────┘
                                         ▼
                          ┌─────────────────────────────────────┐
                          │  Poller thread(s) — NIO selector    │
                          │  waits for readable sockets         │
                          └──────────────┬──────────────────────┘
                                         ▼
                          ┌─────────────────────────────────────┐
                          │  WORKER THREAD POOL                 │
                          │  "http-nio-8080-exec-1..200"        │
                          │                                     │
                          │  ★ ONE THREAD PER IN-FLIGHT REQUEST │
                          │    for its ENTIRE duration          │
                          └─────────────────────────────────────┘
```

```properties
server.tomcat.threads.max=200            # max worker threads (default 200)
server.tomcat.threads.min-spare=10       # kept warm
server.tomcat.accept-count=100           # OS-level backlog once threads are exhausted
server.tomcat.max-connections=8192       # concurrent connections (NIO holds these cheaply)
server.tomcat.connection-timeout=20s
```

### Why this model is the source of most latency incidents

A worker thread is **blocked and unavailable** for the whole request — including while waiting on the database, an HTTP call to Slack, or a slow disk.

**Do the arithmetic.** 200 threads, and every request makes one 2-second call to a slow dependency:

```
throughput = 200 threads / 2 s = 100 requests/second
```

Request 101 in a given second waits in the accept queue. When that fills (100), the OS refuses connections and clients see connection resets. **Your CPU is at 3%.** You are not compute-bound; you are *thread-starved*. This is what a "the service is down but the servers look idle" incident actually is.

**And here's the failure mode people miss:** one slow dependency can take down endpoints that don't even use it. Threads pile up on the slow call, exhaust the pool, and now your `/health` endpoint can't get a thread either. **A single slow downstream service becomes a total outage.** This is precisely why timeouts, circuit breakers and bulkheads exist (Lesson 34) — and being able to explain this chain is a strong senior signal.

**The three responses, and their trade-offs:**

| Approach | Mechanism | Cost |
|---|---|---|
| **More threads** | `threads.max=400` | Each platform thread ≈ 1MB stack + context-switch overhead. Doesn't scale past a few thousand. |
| **Reactive (WebFlux)** | Never block; ~CPU-count event-loop threads | Whole stack must be non-blocking; harder to write and debug (Lesson 18) |
| **Virtual threads** ⭐ | `spring.threads.virtual.enabled=true` | Same blocking code, threads become ~free. **Java 21+. The modern answer.** |

```properties
# Java 21+, Spring Boot 3.2+ — one line
spring.threads.virtual.enabled=true
```

Now each request gets a **virtual thread**: a JVM-managed continuation that costs a few hundred bytes and *unmounts from its carrier thread* when it blocks. You can have a million. **Blocking code becomes cheap again**, which removes most of reactive programming's motivation for typical CRUD services.

**Caveats to state (this is what makes the answer credible rather than hype):**
- **`synchronized` blocks pinned the carrier thread** in Java 21 — a synchronized block around a blocking call negated the benefit. This was largely fixed in Java 24/25, but Java 21 codebases should use `ReentrantLock` on paths that block.
- **`ThreadLocal` on millions of threads** is a memory concern — hence `ScopedValue`.
- **Your database connection pool is now the bottleneck.** A million virtual threads and 10 DB connections means 999,990 threads queueing on Hikari. **Virtual threads move the bottleneck; they don't delete it.** Saying this out loud is the difference between having read a headline and having understood the change.

---

## 4. `DispatcherServlet` — the front controller

Spring MVC registers **one** servlet mapped to `/`. Every request goes through it.

> **The Front Controller pattern:** instead of one servlet per URL, a single entry point handles all requests and *dispatches* to the right handler. That centralization is what lets Spring apply cross-cutting behaviour — exception handling, content negotiation, locale resolution — uniformly.

### The full request lifecycle

```
   ①  TCP connection → Tomcat acceptor → poller → worker thread assigned
                                    │
   ②  FILTER CHAIN (jakarta.servlet.Filter) — before Spring MVC sees anything
       ├─ CharacterEncodingFilter
       ├─ RequestContextFilter        ← binds request to a ThreadLocal (RequestContextHolder)
       ├─ FormContentFilter
       ├─ SPRING SECURITY FILTER CHAIN  ← auth happens HERE, not in the controller  [Lesson 25]
       └─ your custom filters (correlation ID, request logging)
                                    │
   ③  DispatcherServlet.doDispatch()
       │
       ├─ 3a. MultipartResolver — is it multipart? wrap the request
       │
       ├─ 3b. HandlerMapping — which handler serves this URL+method?
       │       RequestMappingHandlerMapping consults a map built at startup by
       │       reflecting over every @Controller's @RequestMapping methods
       │       → returns a HandlerExecutionChain (handler + interceptors)
       │       → no match? → 404 (NoHandlerFoundException)
       │
       ├─ 3c. HandlerInterceptor.preHandle()  (each, in order)
       │       → returns false? short-circuit, response already written
       │
       ├─ 3d. HandlerAdapter — invokes the handler
       │       RequestMappingHandlerAdapter:
       │       ├─ HandlerMethodArgumentResolvers build each parameter:
       │       │    @PathVariable   → PathVariableMethodArgumentResolver
       │       │    @RequestParam   → RequestParamMethodArgumentResolver
       │       │    @RequestBody    → RequestResponseBodyMethodProcessor
       │       │                       → HttpMessageConverter (Jackson) → your object
       │       │    @Valid          → triggers Bean Validation here
       │       ├─ ★ reflectively invokes YOUR controller method
       │       └─ HandlerMethodReturnValueHandler processes the return value
       │            @ResponseBody → HttpMessageConverter → JSON bytes
       │            String/ModelAndView → a view name to resolve
       │
       ├─ 3e. HandlerInterceptor.postHandle()  (reverse order)
       │
       ├─ 3f. ViewResolver + View.render()   [only for view-based responses]
       │
       ├─ 3g. HandlerExceptionResolver  ← if anything above threw
       │       ExceptionHandlerExceptionResolver → your @ExceptionHandler / @ControllerAdvice
       │       ResponseStatusExceptionResolver   → @ResponseStatus annotations
       │       DefaultHandlerExceptionResolver   → Spring's built-in mappings
       │
       └─ 3h. HandlerInterceptor.afterCompletion()  (always, even on exception)
                                    │
   ④  Response flushed → filter chain unwinds (in reverse) → worker thread RETURNED to pool
```

**Narrate that in an interview and you're done.** Very few candidates can name `HandlerMapping`, `HandlerAdapter`, `HandlerMethodArgumentResolver`, `HttpMessageConverter`, and `HandlerExceptionResolver` in the right order. Practise it out loud until it's fluent — it's genuinely the highest-return answer in this module.

### The components, in one table

| Component | Job | Key implementation |
|---|---|---|
| `HandlerMapping` | URL+method → handler | `RequestMappingHandlerMapping` |
| `HandlerAdapter` | Invoke the handler | `RequestMappingHandlerAdapter` |
| `HandlerMethodArgumentResolver` | Build each method parameter | ~30 built-in; **extensible** |
| `HttpMessageConverter` | Body ↔ object | `MappingJackson2HttpMessageConverter` |
| `HandlerMethodReturnValueHandler` | Process the return value | `RequestResponseBodyMethodProcessor` |
| `HandlerExceptionResolver` | Exception → response | `ExceptionHandlerExceptionResolver` |
| `ViewResolver` | View name → `View` | `ThymeleafViewResolver` |
| `LocaleResolver` | Determine locale | `AcceptHeaderLocaleResolver` |
| `MultipartResolver` | Parse uploads | `StandardServletMultipartResolver` |

**`HandlerMethodArgumentResolver` is the extension point worth remembering**, because it answers a great interview question: *"how would you inject the current tenant into every controller method?"*

```java
@Component
public class CurrentTenantResolver implements HandlerMethodArgumentResolver {

    @Override
    public boolean supportsParameter(MethodParameter p) {
        return p.hasParameterAnnotation(CurrentTenant.class);
    }

    @Override
    public Object resolveArgument(MethodParameter p, ModelAndViewContainer mav,
                                  NativeWebRequest req, WebDataBinderFactory binder) {
        return TenantContext.require();
    }
}
```
```java
@GetMapping("/monitors")
List<MonitorResponse> list(@CurrentTenant TenantId tenant) { ... }   // clean, typed, testable
```

Register it via `WebMvcConfigurer.addArgumentResolvers(...)`. **Vigil uses this pattern.**

---

## 5. Filters vs Interceptors — a decision you'll be asked to justify

```
Tomcat
  └─ FILTER CHAIN                    ← servlet spec; sees EVERYTHING
      └─ DispatcherServlet
          └─ INTERCEPTORS            ← Spring MVC; knows the handler
              └─ AOP aspects         ← Spring beans; knows the method
                  └─ your controller
```

| | Filter | HandlerInterceptor |
|---|---|---|
| Defined by | Servlet spec (`jakarta.servlet.Filter`) | Spring MVC |
| Wraps | Every request, including static resources, 404s, and errors | Only requests that reached a handler |
| Can modify request/response bodies | ✅ (via wrappers) | ❌ practically |
| Knows the target handler method | ❌ | ✅ (`HandlerMethod` — including its annotations) |
| Access to Spring beans | ✅ if registered as a bean | ✅ |
| Can short-circuit | ✅ (don't call `chain.doFilter`) | ✅ (`preHandle` returns false) |
| Runs on async completion | needs `AsyncListener` care | `afterConcurrentHandlingStarted` |

**The rule:**
- **Filter** for anything that must apply to *every* request regardless of routing: correlation IDs, request/response logging, security, CORS, compression, rate limiting, body caching.
- **Interceptor** when you need to know *which handler* is being called: reading a custom annotation on the method, per-endpoint auditing, injecting model attributes.
- **AOP aspect** when you care about a *method* and its arguments, not the HTTP request at all.

**Why correlation IDs must be a filter, not an interceptor:** a 404 never reaches a handler, so an interceptor never runs, so that request's logs have no correlation ID — which is exactly the request you'll be trying to trace. Real reasoning, and a strong interview answer.

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class CorrelationIdFilter extends OncePerRequestFilter {

    private static final String HEADER = "X-Correlation-Id";

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res,
                                    FilterChain chain) throws ServletException, IOException {
        String id = Optional.ofNullable(req.getHeader(HEADER))
                            .filter(StringUtils::hasText)
                            .orElseGet(() -> UUID.randomUUID().toString());
        MDC.put("correlationId", id);            // → appears in every log line  [Lesson 35]
        res.setHeader(HEADER, id);               // → client can quote it in a support ticket
        try {
            chain.doFilter(req, res);
        } finally {
            MDC.clear();                         // ★ MANDATORY — thread is reused!
        }
    }
}
```

**Two production-critical details:**
1. **`extends OncePerRequestFilter`, not `implements Filter`.** On a `forward` or `include` dispatch, a raw filter runs *again*. `OncePerRequestFilter` guards against that. Nearly all real filters should extend it.
2. **`MDC.clear()` in a `finally` is not optional.** Worker threads are pooled and reused. Leave MDC populated and the *next* request logs the *previous* request's correlation ID — a debugging nightmare where your traces silently lie to you. Same class of bug as request-scope leaking (Lesson 08).

---

## 6. Async request processing

The blocking model holds a worker thread for the whole request. Async releases it:

```java
@GetMapping("/reports/{id}")
public CompletableFuture<ReportResponse> report(@PathVariable UUID id) {
    return CompletableFuture.supplyAsync(() -> reportService.generate(id), reportExecutor);
}
```

Or with more control:
```java
@GetMapping("/stream")
public SseEmitter stream() {                              // Server-Sent Events
    SseEmitter emitter = new SseEmitter(Duration.ofMinutes(5).toMillis());
    checkEventPublisher.register(emitter);
    return emitter;
}
```

Return types Spring MVC understands asynchronously: `Callable<T>`, `DeferredResult<T>`, `CompletableFuture<T>`, `SseEmitter`, `StreamingResponseBody`, and `Mono`/`Flux`.

**What actually happens:** the Tomcat worker thread returns to the pool as soon as the controller returns; when the future completes, Spring re-dispatches on another thread to write the response.

**The honest caveat:** you haven't removed the blocking work, you've *moved* it to another pool. That's a real win when the work is genuinely long (report generation, SSE, long-polling) or when you want to isolate slow work from your request pool (a bulkhead). It's pointless overhead if you just shove a fast DB call onto another executor.

**And with virtual threads, most of this becomes unnecessary** for ordinary request handling — which is exactly why `spring.threads.virtual.enabled=true` is such a big deal. `SseEmitter` and `StreamingResponseBody` remain useful because they're about *response shape*, not thread economics.

---

## 7. Build & break

**1. ⭐ Watch a real request go through, end to end.**
```properties
logging.level.org.springframework.web=DEBUG
logging.level.org.springframework.web.servlet.DispatcherServlet=TRACE
logging.level.org.springframework.web.servlet.mvc.method.annotation=TRACE
```
Hit an endpoint and read the log carefully. You'll see the handler mapped, arguments resolved, the return value written. **Match each line to a step in §4.** This is the exercise that converts the diagram into knowledge.

**2. See the handler map.** Hit `/actuator/mappings` — every URL → handler method, exactly as `RequestMappingHandlerMapping` built it at startup.

**3. Prove the thread model, and starve the pool.**
```java
@GetMapping("/slow")
public String slow() throws InterruptedException {
    log.info("start on {}", Thread.currentThread());
    Thread.sleep(5000);
    log.info("done  on {}", Thread.currentThread());
    return "ok";
}
```
```properties
server.tomcat.threads.max=5
server.tomcat.accept-count=2
```
Fire 10 concurrent requests:
```bash
for i in $(seq 1 10); do curl -s -o /dev/null -w "%{http_code} %{time_total}\n" http://localhost:8080/slow & done; wait
```
(PowerShell: `1..10 | ForEach-Object -Parallel { curl.exe -s -o NUL -w "%{http_code} %{time_total}`n" http://localhost:8080/slow }`)

Observe: 5 run immediately, 2 queue, the rest are **refused**. You just reproduced a production thread-starvation incident on your laptop with 5 lines of config. **This is the single most valuable experiment in the lesson.**

**4. Fix it with virtual threads.** Add `spring.threads.virtual.enabled=true` (Java 21 required — install it if you haven't) and repeat. Note the thread names change to `VirtualThread[#...]` and all 10 requests complete in ~5s total. **Then reason about what would happen if `/slow` hit the database instead of sleeping** — the answer is "you'd starve the Hikari pool instead," which is the whole lesson about virtual threads moving bottlenecks.

**5. Write the correlation-ID filter** from §5. Verify the header round-trips, that it appears in log output (add `%X{correlationId}` to your logging pattern), and that a **404** also gets one. Then **remove `MDC.clear()`**, hammer the endpoint concurrently, and watch correlation IDs bleed between requests. Feel that bug.

**6. Filter vs interceptor, empirically.** Register both, printing on entry. Then request:
- a valid endpoint → both run
- a nonexistent URL → **only the filter runs**
- a static resource (`/favicon.ico`) → only the filter

**That's the decision matrix, proven rather than memorized.**

**7. Custom argument resolver.** Implement `@CurrentTenant` from §4 (read the tenant from a header for now). Use it in a controller. This is a genuinely impressive thing to be able to write from scratch in an interview.

### Explain out loud (60s)

1. The request lifecycle from TCP socket to JSON, naming each component in order.
2. 401 vs 403, and which methods are safe and which idempotent.
3. The thread arithmetic, and why one slow dependency takes down unrelated endpoints.
4. What virtual threads change, what they don't, and the two caveats.
5. Filter vs interceptor vs aspect — what each can see, and the deciding case.

---

## 8. Production rules

| Do / Never | Why |
|---|---|
| **401 means unauthenticated, 403 means authenticated and refused** | The names are historically backwards, and the distinction tells the client whether retrying with credentials can possibly help |
| **Send `Location` on every 201, and `Retry-After` on every 429** | Without them the client has to guess where the resource went or when to come back, and guessing clients retry aggressively |
| **Treat POST as non-idempotent and give it an idempotency key** | Mobile clients, load balancers and consumers all retry on timeout; two POSTs create two monitors, and no amount of care at the client fixes that |
| **Stay on Tomcat unless you've measured a reason not to** | Switching is one dependency swap, but the industry's operational knowledge of Tomcat is worth more than a marginal benchmark win |
| **Never size a thread pool without doing `threads / latency` arithmetic** | 200 threads against a 2-second dependency is 100 req/s, whatever your CPU is doing — capacity is set by the slowest call, not the hardware |
| **Put a timeout on every outbound call** | Without one, threads pile up on a slow dependency until the pool is empty and endpoints that never touch it start failing too |
| **Say out loud that virtual threads move the bottleneck rather than removing it** | A million virtual threads and ten Hikari connections means a million threads queueing on the pool — the constraint relocates, it doesn't vanish |
| **On Java 21, use `ReentrantLock` instead of `synchronized` around blocking calls** | A `synchronized` block pins the carrier thread, cancelling the benefit of virtual threads; this was only fixed in much later JDKs |
| **Put cross-cutting HTTP concerns in a `Filter`, not an interceptor or an aspect** | Only the filter runs for 404s, static resources and requests that never reach a controller — which is exactly when you need a correlation ID |
| **Always `MDC.clear()` in a `finally`** | Worker threads are pooled and reused, so a stale entry attaches one request's correlation ID to the next user's logs |
| **Never authenticate in a controller** | Authentication happens in the security filter chain, before Spring MVC exists — a check in the controller is both too late and easy to forget |

---

## 9. Interview traps

**Q1. Walk me through what happens when an HTTP request hits your Spring Boot app.** *(the flagship question)*
Give §4: TCP → Tomcat acceptor/poller → a worker thread from the pool → the servlet filter chain (including Spring Security) → `DispatcherServlet.doDispatch` → `HandlerMapping` finds the handler → interceptor `preHandle` → `HandlerAdapter` uses argument resolvers (and `HttpMessageConverter` for `@RequestBody`) to build parameters → your controller method is invoked reflectively → the return value goes through a return-value handler and message converter to JSON → `postHandle` → exception resolvers if anything threw → `afterCompletion` → response flushed → **the worker thread returns to the pool**.

**Q2. What is `DispatcherServlet`?**
The single front-controller servlet, mapped to `/`, that all requests pass through. It owns the dispatch pipeline (mapping, adapting, converting, exception resolution) so cross-cutting behaviour is applied uniformly instead of per-servlet.

**Q3. How many threads handle requests, and what happens when they run out?**
Tomcat's worker pool, 200 by default, **one thread per in-flight request for its whole duration**. When exhausted, requests queue in `accept-count` (default 100); beyond that the OS refuses connections. Symptom: high latency and connection resets with low CPU — thread starvation, not compute saturation.

**Q4. One downstream dependency gets slow. Why might your whole service go down?**
Threads block on that call and accumulate until the pool is exhausted — then *every* endpoint, including health checks, is starved. Mitigations: aggressive timeouts, circuit breakers, bulkheads (separate pools per dependency), and virtual threads to make thread exhaustion far less likely.

**Q5. What do virtual threads change, and what don't they?**
They make each request's thread nearly free (hundreds of bytes, unmounted while blocked), so blocking code scales without a large platform-thread pool. They **don't** remove downstream limits — your connection pool, rate limits, and the downstream service itself become the bottleneck. Java 21 caveats: `synchronized` pinned the carrier thread (fixed in later versions; prefer `ReentrantLock`), and `ThreadLocal` at that scale is a memory concern (`ScopedValue` is the successor).

**Q6. Filter vs interceptor vs AOP — how do you choose?**
Filter (servlet spec) for concerns applying to every request including 404s and static resources, and where you may need to wrap the request/response: correlation IDs, logging, security, CORS, rate limiting. Interceptor when you need the resolved `HandlerMethod` (e.g. reading a custom annotation on the target method). AOP when you care about the method and its arguments, independent of HTTP.

**Q7. Why must a correlation-ID filter be a filter?**
Because requests that 404 or hit static resources never reach a handler, so an interceptor wouldn't run — leaving exactly the requests you most want to trace without an ID.

**Q8. Why `OncePerRequestFilter`?**
A plain `Filter` runs again on internal `forward`/`include` dispatches, duplicating side effects. `OncePerRequestFilter` sets a request attribute to guarantee a single execution.

**Q9. Why must you clear MDC in a `finally`?**
Worker threads are pooled and reused. Leftover MDC values leak into the next request's logs, so your traces attribute the wrong correlation ID — silently wrong observability, which is worse than none.

**Q10. 401 vs 403?**
401 Unauthorized = not authenticated (missing/invalid credentials; retrying with credentials may work; should include `WWW-Authenticate`). 403 Forbidden = authenticated but not permitted (retrying is pointless). The naming is historically misleading.

**Q11. Which HTTP methods are idempotent, and why does it matter?**
GET, HEAD, PUT, DELETE (and OPTIONS/TRACE). POST and PATCH are not. It matters because clients, load balancers and message consumers retry: a retried PUT is harmless, a retried POST duplicates. Hence idempotency keys for POST.

**Q12. How would you make a value available to every controller method without a parameter on each?**
A custom `HandlerMethodArgumentResolver` bound to a custom annotation (e.g. `@CurrentTenant`), registered via `WebMvcConfigurer`. Type-safe and testable — better than a static holder or reading a header in every method.

---

## 10. Retention check

1. Narrate the request lifecycle naming at least six Spring components. Time yourself; aim for 90 seconds.
2. With 200 threads and a uniform 500ms response time, what's your theoretical max throughput? What if one endpoint slows to 5s?
3. Why does a single slow dependency threaten your whole service, and name three mitigations.
4. What exactly does a virtual thread do when it hits a blocking call, and what becomes the new bottleneck?
5. Give the one-line rule for filter vs interceptor, plus the 404 argument.
6. What breaks if you forget `MDC.clear()`, and why is that especially insidious?

---

**Next:** [Lesson 15 — Controllers, mapping & serialization](15-controllers-and-mapping.md) — every way to get data in and out of a controller, and precise control over JSON.

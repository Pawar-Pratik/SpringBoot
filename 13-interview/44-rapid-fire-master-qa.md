# Lesson 44 — Rapid-fire master Q&A

> **The whole course, revisable in 90 minutes.** Cover the answers. Say yours out loud *before* reading. Failing to recall is what builds retrieval strength — being reminded of an answer feels identical and is nearly useless under pressure.
>
> **How to use this:** first pass, answer everything and mark what you missed. Second pass (next day), only the misses. Third pass (a week later), only the misses again. Anything still missing after three passes, go back to the lesson.

**Legend:** ⭐ = asked in almost every interview · 🔥 = separates senior from mid-level

---

## Module 1 — Java & the JVM

**1. `ClassNotFoundException` vs `NoClassDefFoundError` vs `NoSuchMethodError`?**
CNFE: a checked exception from an explicit dynamic lookup (`Class.forName`). NCDFE: an `Error` — the compiler resolved the class but it's absent at runtime, *or* its static initializer threw. NSME: the class is present but the signature isn't → **a version conflict**.

**2. ⭐ How does Spring Boot's executable JAR work?**
Fat JAR with dependencies nested under `BOOT-INF/lib`. The JVM can't load nested JARs, so the manifest's `Main-Class` is Boot's `JarLauncher`, which installs a classloader that reads nested entries and then reflectively invokes your `Start-Class`.

**3. How does Maven resolve a version conflict?**
Nearest-definition-wins (shallowest depth), ties broken by declaration order. **Not** highest-version-wins — that's Gradle. Fix by declaring directly, excluding, or `dependencyManagement`. Diagnose with `dependency:tree`.

**4. Why is the Postgres driver `runtime` scope?**
So it's absent from the compile classpath, making it *impossible* to import driver classes in business code. The build tool enforces the abstraction.

**5. ⭐ Why should Spring beans be stateless?**
They're singletons shared by every request thread (Tomcat: 200). Mutable instance fields become shared mutable state → race conditions and cross-request data leakage. Bugs appear only under production load and can't be reproduced.

**6. `==` vs `.equals()`, and the `Integer` trap?**
`==` compares references for objects. The JVM caches `Integer` −128..127, so `==` is *intermittently* correct on boxed integers — worse than always wrong. Use `==` only for primitives and enums.

**7. The `equals`/`hashCode` contract?**
Equal objects must have equal hash codes. Break it and hash collections lose your objects: `contains()` returns false for a contained element. Unequal objects *may* collide — that's legal, just slower.

**8. ⭐ Checked vs unchecked, and why does Spring prefer unchecked?**
Most infrastructure failures aren't locally recoverable, so forcing `throws`/`catch` adds noise without safety. Spring translates `SQLException` into a portable unchecked `DataAccessException` hierarchy. **And `@Transactional` only rolls back on unchecked by default.**

**9. Why `BigDecimal` for money, and the constructor trap?**
Binary floating point can't represent 0.1. `new BigDecimal(0.1)` inherits the double's error — use the `String` constructor. Compare with `compareTo`, since `equals` distinguishes `2.0` from `2.00`.

**10. `Instant` vs `LocalDateTime` for a created-at column?**
`Instant` — an unambiguous point on the UTC timeline. `LocalDateTime` has no zone, so it's ambiguous across servers and broken by DST.

**11. Why records for DTOs but not JPA entities?**
Records are final, immutable, and have no no-arg constructor. Hibernate needs a non-final class (proxy subclassing), a no-arg constructor (reflective instantiation), and mutable fields (dirty checking).

**12. `orElse` vs `orElseGet`?**
`orElse`'s argument is **always evaluated**, even when a value is present. `orElseGet` is lazy. Use `orElseGet` whenever the default is expensive or has side effects.

**13. Where does `Optional` belong?**
Return types expressing "no result." Not fields, parameters, or collections — that's the designers' stated intent.

**14. When would you use `parallelStream()`?**
Rarely. Only large, CPU-bound, in-memory work with no I/O or shared state, after measuring. It uses the shared common ForkJoinPool, and in a web app requests are already parallel — so you compete with your own throughput.

**15. ⭐🔥 What does an annotation actually do?**
Nothing. It's inert metadata. A framework must read it via reflection — which requires `RetentionPolicy.RUNTIME` (`CLASS` is the default and isn't visible to reflection).

**16. ⭐🔥 How does dependency injection actually work?**
Spring reflects on the constructor, reads its parameter types, resolves a bean for each (recursively), and calls `Constructor.newInstance(args)`. Field injection uses `Field.setAccessible(true)` + `set()`.

**17. ⭐🔥 What is a proxy, and which annotations depend on one?**
An object implementing the same type as your bean, wrapping it and adding behaviour before/after delegating. `@Transactional`, `@Async`, `@Cacheable`, `@Retryable`, `@PreAuthorize`, `@Validated`, `@Observed` — plus Hibernate's lazy loading.

**18. Why is Spring startup slow but runtime fast?**
All reflective work — scanning, condition evaluation, dependency resolution, proxy generation — happens once at startup. After that, calls are ordinary invocations through pre-built proxies.

---

## Module 2 — Spring Core

**19. ⭐ `BeanFactory` vs `ApplicationContext`?**
`BeanFactory` is the minimal lazy container. `ApplicationContext` adds **eager** singleton instantiation, events, i18n, resource loading, automatic `BeanPostProcessor` detection, and `Environment`. Eager is deliberate: misconfiguration fails at startup, not at 3am.

**20. `@Component` vs `@Bean`?**
`@Component` marks a class you own for scanning. `@Bean` is a method returning any object — required for third-party classes, conditional creation, or multiple instances of one type.

**21. 🔥 Are `@Service`, `@Repository` and `@Component` the same?**
Not quite. `@Service` adds nothing functional. **`@Repository` enables exception translation** to `DataAccessException` via a post-processor proxy. `@Controller` marks the class for request-handler scanning. `@Configuration` gets a CGLIB proxy.

**22. 🔥 What does `proxyBeanMethods = false` change?**
No CGLIB proxy on the `@Configuration` class, so calling one `@Bean` method from another **executes it again** — potentially creating duplicate objects (e.g. multiple connection pools). Spring's own auto-configs use it because they always pass dependencies as parameters.

**23. 🔥 `BeanFactoryPostProcessor` vs `BeanPostProcessor`?**
BFPP runs **before** instantiation, on bean *definitions* (e.g. `${...}` resolution). BPP runs **during** instantiation, on *instances*, and can return a substitute — **which is how AOP proxies are created**.

**24. ⭐ Constructor vs field injection — give four reasons.**
(1) `final` fields → immutability and safe cross-thread publication. (2) The object can never be partially initialized. (3) Testable with plain `new`; adding a dependency breaks test *compilation* rather than NPE-ing. (4) A long parameter list is a visible design signal that field injection hides. Bonus: cycles fail at startup.

**25. ⭐ How does Spring choose between multiple candidates of one type?**
`@Qualifier` at the injection point → `@Primary` on a candidate → parameter/field **name** matching a bean name → `@Priority`. Otherwise `NoUniqueBeanDefinitionException`.

**26. What happens if you inject `List<SomeInterface>`?**
You get every implementation, ordered by `@Order`/`Ordered` (arbitrary otherwise). It's the idiomatic strategy/plugin pattern — new implementations need no change to the consumer.

**27. ⭐🔥 Why did Boot 2.6 ban circular references by default?**
The old resolution injected partially constructed "early references," which could hand a bean a **pre-proxy** instance — silently disabling `@Transactional`/`@Async` on it — and made initialization order fragile.

**28. Best fix for a circular dependency?**
Extract the shared responsibility into a third bean, or invert one direction with an application event. `@Lazy` and `allow-circular-references=true` are band-aids; the latter is technical debt with a config key.

**29. ⭐🔥 What happens when you inject a prototype into a singleton?**
It's resolved **once** — you get one instance forever. Fixes: `ObjectProvider.getObject()`, `@Lookup`, or a scoped proxy. Often the right answer is just `new`.

**30. Does Spring destroy prototype beans?**
No. `@PreDestroy` is never called; the caller owns cleanup.

**31. ⭐ How can a request-scoped bean be injected into a singleton?**
Via a **scoped proxy** that resolves the current thread's instance per method call through `RequestContextHolder`'s ThreadLocal. Without it: "Scope 'request' is not active for the current thread."

**32. ⭐🔥 Why do request scope, `SecurityContextHolder` and MDC all break in `@Async`?**
All three are `ThreadLocal`-based and `@Async` runs on a different thread. Fix with a `TaskDecorator` (or `DelegatingSecurityContext*Executor`), clearing in a `finally`.

**33. ⭐🔥 Why doesn't `@Transactional` work in `@PostConstruct`?**
Proxies are created in `BeanPostProcessor.postProcessAfterInitialization` — **after** `@PostConstruct`. You hold the raw target, so no advice applies. Use `ApplicationReadyEvent`.

**34. When does `@PreDestroy` not run?**
`SIGKILL`, OOM kill, JVM crash, hard container stop. **Never rely on it for data integrity** — durability must be achieved before shutdown.

**35. ⭐ What is graceful shutdown and why does it matter?**
`server.shutdown=graceful` stops accepting new connections on SIGTERM and lets in-flight requests finish within `timeout-per-shutdown-phase`. With readiness probes and a preStop delay, it enables zero-downtime deploys.

**36. 🔥 How does `@ConditionalOnClass` avoid `NoClassDefFoundError`?**
Spring reads the annotation from the class file with **ASM**, without loading the referenced class, and asks the classloader whether the name resolves.

---

## Module 3 — AOP

**37. ⭐ Define aspect, join point, pointcut, advice, weaving.**
Aspect = the module. Join point = where advice could apply (**in Spring AOP, always a method execution**). Pointcut = the selecting expression. Advice = the code. Weaving = linking (Spring: runtime, via proxies).

**38. Which advice type can prevent the target running, and what uses that?**
`@Around`, by not calling `proceed()`. Caching returns a hit without invoking; retry calls `proceed()` repeatedly; `@Transactional` wraps it in try/commit/rollback.

**39. ⭐ JDK proxy vs CGLIB — which does Boot use?**
CGLIB (`proxyTargetClass=true`). JDK proxies need an interface and the result **isn't** an instance of your class (→ `BeanNotOfRequiredTypeException`). CGLIB subclasses, so it can't handle `final` classes/methods or `private` methods.

**40. ⭐⭐🔥 Explain self-invocation.** *(the flagship AOP question)*
Proxy AOP intercepts only calls arriving *through* the proxy. `this.method()` executes inside the target, bypassing it, so the annotation has **no effect, silently**. Fixes: extract to another bean (best), self-inject `@Lazy`, `TransactionTemplate`, `AopContext.currentProxy()`, or AspectJ weaving (the only true fix).

**41. ⭐ Why is `@Transactional` on a `private` method a bug?**
CGLIB advises by overriding; `private`/`final`/`static` methods can't be overridden. No error — the annotation is simply inert.

**42. Which annotations does self-invocation break, and which is a *security* bug?**
`@Transactional`, `@Async`, `@Cacheable`, `@CacheEvict`, `@Retryable`, `@Validated`, `@Observed` — and **`@PreAuthorize`, where a skipped check is an authorization bypass.**

**43. How do you check at runtime whether AOP applied?**
`bean.getClass().getName()` — a `$$SpringCGLIB$$` or `$Proxy` suffix means proxied. For transactions: `TransactionSynchronizationManager.isActualTransactionActive()` and `logging.level.org.springframework.transaction.interceptor=TRACE`.

**44. What must every `@Around` advice do?**
Call `proceed()`, **return its result**, and **rethrow** exceptions. Swallowing either corrupts caller semantics far from the cause.

**45. 🔥 When would you NOT write an aspect?**
For business logic (invisible rules), for anything Spring provides, and for HTTP concerns (use filters — they see 404s and static resources and can short-circuit). Biggest risk: a typo'd pointcut or missing `@Component` fails **silently** — hence a mandatory test asserting the aspect applies.

---

## Module 4 — Spring Boot

**46. ⭐⭐ How does auto-configuration work?** *(the flagship Boot question)*
`@SpringBootApplication` → `@EnableAutoConfiguration` → `@Import(AutoConfigurationImportSelector)`, a **`DeferredImportSelector`** so it runs *after* all user config. It reads `META-INF/spring/…AutoConfiguration.imports` from every JAR (~150 candidates), pre-filters via a metadata index without loading classes, applies exclusions, sorts by `@AutoConfigureBefore/After`, and imports survivors. Each one's `@Conditional`s then decide: typically `@ConditionalOnClass` ("is the library present?") + `@ConditionalOnMissingBean` ("did the user define their own?").

**47. ⭐🔥 Why is the import selector deferred?**
So auto-config is processed **last**, making `@ConditionalOnMissingBean` meaningful — all user beans are registered by then. Reverse it and user overrides stop working.

**48. What is a starter? Does it contain code?**
A curated dependency bundle — **usually no code at all**. Its effect is indirect: it puts classes on the classpath, flipping `@ConditionalOnClass` conditions true and activating auto-configs that already exist.

**49. ⭐ How do you find out why an auto-configuration didn't apply?**
`--debug` / `debug=true` for the condition evaluation report, or `/actuator/conditions`. Read the **negative matches** — each states the exact unmet condition.

**50. 🔥 Override vs customize an auto-configured bean?**
Defining your own bean makes Boot back off entirely — so you lose *all* its defaults. Prefer the customizer hook (`Jackson2ObjectMapperBuilderCustomizer`, `WebMvcConfigurer`), which augments rather than replaces. **"Customize, don't replace."**

**51. ⭐ Property precedence order?**
CLI args > `SPRING_APPLICATION_JSON` > system properties > env vars > profile-specific outside the JAR > profile-specific inside > `application.yml` outside > inside > `@PropertySource` > defaults. Two rules: more specific wins, closer to the deployment wins.

**52. ⭐ `@Value` vs `@ConfigurationProperties`?**
`@Value`: one value, SpEL, **exact key only**, no validation. `@ConfigurationProperties`: a typed object with relaxed binding, Bean Validation (**fails at startup**), nesting, IDE metadata, and `/actuator/configprops` visibility.

**53. What is relaxed binding, and where doesn't it apply?**
`vigil.checks.max-concurrent` also binds from `maxConcurrent`, `max_concurrent`, `VIGIL_CHECKS_MAX_CONCURRENT`. **Applies to `@ConfigurationProperties`, not `@Value`.**

**54. 🔥 When should you use profiles, and when not?**
Use them when the **set of beans** differs structurally. Never for values, and never for business logic — that makes production the only untested code path. Property-driven config keeps one code path everywhere.

**55. ⭐ How do you handle secrets?**
Never in the repo. Minimum: env vars + gitignored local files + a secret-scanning pre-commit hook. Better: mounted secret files via `spring.config.import=configtree:/run/secrets/`. Best: a secret manager with **rotation**, or workload identity so no long-lived secret exists.

**56. ⭐🔥 How do you structure a project?**
Package-by-feature, not by layer — each feature owns its controller/service/repository/entity/DTOs, so repositories and entities can be **package-private** and the compiler enforces that other features go through the public API. Package-by-layer forces everything public: layers on paper, no boundaries in code.

**57. ⭐ Where does `@Transactional` belong?**
The service layer. Not the controller (the transaction would span serialization). Not the repository (each method becomes its own, so you can't compose writes atomically).

**58. 🔥 How do you stop architecture eroding?**
Three tiers: package-private visibility (free, compiler-enforced), **ArchUnit tests** (cheap, precise, in CI), and multi-module Maven or Spring Modulith. Start with the first two.

---

## Module 5 — Web & REST

**59. ⭐⭐ Walk me through what happens when a request arrives.** *(the flagship web question)*
TCP → Tomcat acceptor/poller → a **worker thread from the pool** → servlet filter chain (including Spring Security) → `DispatcherServlet.doDispatch` → `HandlerMapping` finds the handler → interceptor `preHandle` → `HandlerAdapter` builds parameters via `HandlerMethodArgumentResolver`s (and `HttpMessageConverter` for `@RequestBody`) → your method is invoked reflectively → the return value goes through a return-value handler and message converter → `postHandle` → exception resolvers if it threw → `afterCompletion` → response flushed → **thread returns to the pool**.

**60. ⭐ How many threads handle requests, and what happens when they run out?**
Tomcat's worker pool, 200 by default, **one thread per in-flight request for its whole duration**. Exhausted → requests queue in `accept-count` (100) → then the OS refuses connections. Symptom: high latency and connection resets with **low CPU**.

**61. ⭐🔥 One downstream dependency gets slow. Why might your whole service go down?**
Threads block on it and accumulate until the pool is exhausted — then *every* endpoint, including health checks, is starved. Mitigations: timeouts, circuit breakers, bulkheads, virtual threads.

**62. ⭐🔥 What do virtual threads change, and what don't they?**
They make each request's thread nearly free (unmounted while blocked), so blocking code scales — removing reactive's main motivation. They **don't** remove downstream limits: your connection pool and rate limits become the bottleneck. Java 21 caveats: `synchronized` pinning (fixed later; prefer `ReentrantLock`), ThreadLocal at scale, no backpressure. Don't pool them.

**63. ⭐ Filter vs interceptor vs aspect?**
Filter for concerns applying to *every* request including 404s and static resources, and where you may wrap request/response: correlation IDs, logging, security, CORS, rate limiting. Interceptor when you need the resolved `HandlerMethod` (e.g. a custom annotation on the target method). Aspect when you care about the method and its arguments, not HTTP.

**64. Why must a correlation-ID filter be a filter, and why `OncePerRequestFilter`?**
404s and static resources never reach a handler, so an interceptor wouldn't run — leaving exactly the requests you most want to trace without an ID. `OncePerRequestFilter` prevents re-execution on internal forwards.

**65. ⭐ Why must you clear MDC in a `finally`?**
Worker threads are pooled. Leftover values leak into the next request's logs, so your traces attribute the wrong correlation ID — silently wrong observability, worse than none.

**66. ⭐ 401 vs 403?**
401 = not authenticated (retrying with credentials may work). 403 = authenticated but not permitted (retrying is pointless). The naming is historically misleading.

**67. ⭐ Which HTTP methods are idempotent, and why does it matter?**
GET, HEAD, PUT, DELETE (and OPTIONS/TRACE). POST and PATCH are not. Clients, load balancers and consumers retry — a retried PUT is harmless, a retried POST duplicates. Hence idempotency keys.

**68. ⭐⭐ Why not return JPA entities from controllers?**
Five reasons: your schema becomes the API contract (renames break clients); sensitive fields are exposed automatically; Jackson triggers lazy loading → N+1, `LazyInitializationException`, or infinite recursion; one entity can't shape both list and detail; and as *input* it's a **mass-assignment vulnerability** letting clients set `id`, `status`, `organizationId`.

**69. ⭐ `@NotNull` vs `@NotEmpty` vs `@NotBlank`?**
`@NotNull` rejects null only. `@NotEmpty` rejects null and zero-length. `@NotBlank` (String only) rejects null, empty, and whitespace-only. For user text: `@NotBlank`.

**70. ⭐🔥 `@Valid` vs `@Validated`, and why must a handler cover both exceptions?**
`@Valid` (Jakarta) on parameters and nested fields for cascading. `@Validated` (Spring) on a **class** to enable method-parameter validation, and it supports groups. They throw **different** exceptions — `MethodArgumentNotValidException` vs `ConstraintViolationException` — so missing the second makes query-param validation return an ugly 500.

**71. Why did validation break after upgrading to Boot 2.3+?**
`spring-boot-starter-validation` was removed from `starter-web`, so Hibernate Validator left the classpath and `@Valid` became a silent no-op.

**72. ⭐ What must never appear in an error response?**
Stack traces, SQL, framework/class names, internal messages, and which permission failed. Log it all server-side with a generated `errorId` and return only that ID.

**73. ⭐🔥 What's wrong with offset pagination?**
`OFFSET n` makes the database scan and discard n rows, so deep pages degrade linearly; and concurrent inserts shift rows, duplicating or skipping items. Cursor/keyset (`WHERE (ts,id) < (:lastTs,:lastId)`) is constant-time and stable, at the cost of no random access and no total count.

**74. Why MapStruct over ModelMapper?**
Compile-time generated plain Java: zero reflection, readable output, and unmapped fields become **compile errors** with `unmappedTargetPolicy=ERROR`. Reflective mappers fail silently or at runtime.

**75. ⭐ Is CORS a security feature for your API?**
No. It protects *users* from other websites and is enforced by the **browser** — curl/Postman/mobile ignore it. Authentication and authorization protect your API.

**76. ⭐ Why does CORS break when you add Spring Security, and what's the fix?**
The security filter chain runs before `DispatcherServlet`; the unauthenticated `OPTIONS` preflight gets a 401, which the browser reports as a CORS error. Fix: `.cors(Customizer.withDefaults())` so Security's `CorsFilter` handles preflights early.

**77. Why is `allowedOrigins("*")` + `allowCredentials(true)` illegal?**
It would let any website make credentialed requests as your logged-in users and read the responses. The spec forbids it; Spring throws at startup.

**78. ⭐ Why would an `Instant` serialize as a number or array?**
`WRITE_DATES_AS_TIMESTAMPS` enabled, or `JavaTimeModule` unregistered — almost always because someone defined a bare `ObjectMapper` bean, making Boot back off and dropping every default.

**79. Why is `@JsonTypeInfo(use = CLASS)` dangerous?**
It lets the payload name a Java class to instantiate — a deserialization gadget attack behind numerous CVEs. Use `use = NAME` with an explicit `@JsonSubTypes` allowlist.

**80. ⭐🔥 When would you choose WebFlux over MVC?**
When I need **backpressure** for streaming, or the service is an I/O-bound gateway with very high connection counts and little business logic, and the whole stack is non-blocking. Otherwise MVC + virtual threads: similar scalability without losing JPA, stack traces, ThreadLocal context, or team familiarity.

**81. ⭐ What must every outbound HTTP call have?**
Connect **and** read timeouts. Defaults are effectively infinite, so one hung upstream cascades to total outage. Then retries with jittered backoff, a circuit breaker, and a bulkhead.

---

## Module 6 — Data & persistence

**82. ⭐ Why pool connections, and what does `close()` do?**
Opening one costs 5–50ms (TCP + TLS + auth, plus a process fork on Postgres). `close()` on a pooled connection **returns it to the pool** — you're holding a proxy that intercepts the call.

**83. ⭐⭐🔥 What pool size, and why isn't bigger better?**
Roughly `(cores × 2) + spindles` — often 10. Bigger is usually slower: the database has finite CPU/disk, so more concurrency just time-slices the same work while adding context switching, per-connection memory, process overhead and lock contention. Queueing at the pool is healthy admission control.

**84. ⭐🔥 Ten pods × 20 connections. What's the problem?**
200 connections against a `max_connections` of 100 → "too many clients already", and autoscaling makes it worse. Pool size is per-instance; the limit is global. Fix: size for replica count, raise `max_connections`, or use PgBouncer in transaction mode.

**85. ⭐ `Connection is not available, request timed out`. How do you diagnose it?**
Read `total/active/idle/waiting` in the message, then investigate **duration** not size: slow queries, `@Transactional` spanning an HTTP call, `open-in-view=true`, or a leak (enable `leak-detection-threshold`). Raising the pool masks it and moves the bottleneck into the database.

**86. 🔥 What is `max-lifetime` for?**
Retiring connections before something else silently kills them — the DB's idle timeout, or a load balancer's (AWS NLB: 350s). Without it you get intermittent "connection reset" errors after quiet periods.

**87. ⭐ What does `@Repository` actually do?**
Beyond being a `@Component`, it enables **exception translation**: a post-processor proxies the bean so vendor `SQLException`s become Spring's portable unchecked `DataAccessException` hierarchy.

**88. 🔥 When would you use `JdbcClient` instead of JPA?**
Reporting/analytics (window functions, CTEs, percentiles JPQL can't express), bulk inserts/updates, read-only projections, and hot paths needing exact SQL. JPA for CRUD on a rich domain model with relationships and optimistic locking.

**89. ⭐⭐ What is the persistence context?**
A transaction-scoped first-level cache and change tracker: a map from entity key to instance plus a snapshot of loaded state. It gives identity guarantees (one row = one object), automatic dirty checking, and write-behind (deferred, batchable SQL).

**90. ⭐ Why does an entity update without calling `save()`?**
It's *managed*. At flush, Hibernate compares each managed entity against its loaded snapshot and issues UPDATEs for differences.

**91. ⭐ My update didn't persist and there's no error. Why?**
The entity was detached — usually because the method had no `@Transactional`, so the context closed after the read and the mutation was never tracked.

**92. 🔥 `persist()` vs `merge()`, and what does `save()` do?**
`persist()` makes the *given* transient instance managed. `merge()` copies state into a managed **copy** and returns it — the argument stays detached. `save()` checks `isNew()` and calls one or the other, which is why saving entities with assigned IDs issues an extra SELECT each.

**93. ⭐ When does Hibernate flush?**
At commit; before a JPQL/Criteria query whose results could be affected (default `AUTO`); on explicit `flush()`; and immediately on insert with `IDENTITY` IDs. Flush ≠ commit — flushed changes still roll back.

**94. ⭐⭐ What is `LazyInitializationException` and how do you fix it properly?**
Touching an uninitialized proxy after the context closed. Proper fixes: fetch it in the query (`JOIN FETCH`/`@EntityGraph`) or use a DTO projection. Bad fixes: `EAGER` (degrades every other query, can't be overridden per-query) or relying on `open-in-view`.

**95. ⭐⭐🔥 What is `open-in-view`, what's the default, and why disable it?**
It keeps the persistence context open for the whole HTTP request; Boot defaults it to **true**. Disable it: it holds a database connection through serialization (a leading cause of pool exhaustion), hides N+1 problems in the presentation layer, and masks incomplete service contracts.

**96. ⭐ Which fetch types default to EAGER?**
`@ManyToOne` and `@OneToOne`. Set them `LAZY` explicitly — EAGER can't be overridden per query, whereas LAZY can be upgraded with a fetch join, so LAZY is strictly more flexible.

**97. 🔥 Why does `GenerationType.IDENTITY` prevent batch inserts?**
Hibernate must execute each INSERT immediately to obtain the generated key, so it can't defer or group. Use `SEQUENCE` with an `allocationSize` matching the DB sequence increment.

**98. 🔥 UUID or auto-increment primary keys?**
Sequential: best index locality, but **enumerable** — a data-scraping risk in a multi-tenant API. UUID v4: safe but fragments the B-tree on insert. **UUID v7**: time-ordered, so unguessable *and* insert-friendly. Best default for high-volume multi-tenant tables.

**99. ⭐ Why must you use `EnumType.STRING`?**
`ORDINAL` (the default) stores the index, so adding or reordering constants silently reinterprets every existing row — no error, complete corruption.

**100. ⭐⭐ Explain the bidirectional relationship trap.**
`mappedBy` marks the collection as the inverse side; Hibernate persists only changes to the **owning** side (the FK holder). Adding to `parent.getChildren()` without `child.setParent(parent)` gives a null FK — the object graph looks right, the database doesn't. Always write `addX`/`removeX` helpers updating both sides.

**101. ⭐ Why is `CascadeType.ALL` on `@ManyToOne` dangerous?**
Deleting the child cascades REMOVE to the parent, which cascades to *its* children — deleting a monitor can delete the whole organization. Cascades follow composition, parent→child only.

**102. ⭐🔥 How do you implement `equals`/`hashCode` on an entity?**
Use `Hibernate.getClass()` (not `getClass()`) so proxies compare correctly; compare only the ID and return false when it's null; and return a **constant** `hashCode` so it's stable across the transient→persistent transition. Better: an application-assigned immutable business key.

**103. ⭐ Why is Lombok `@Data` wrong on an entity?**
`equals`/`hashCode` over all fields touches lazy associations (queries or `LazyInitializationException` inside `Set.add`) with an unstable hash, and `toString` recurses infinitely across bidirectional relations. Safe subset: `@Getter`, `@Setter`, `@NoArgsConstructor(PROTECTED)`, `@ToString(onlyExplicitlyIncluded = true)`.

**104. 🔥 Why avoid `@ManyToMany`?**
The join table holds only two FKs, but real relationships acquire attributes (role, joined-at, invited-by) and adding one is a migration. Model it as an entity with two `@ManyToOne`s.

**105. 🔥 How does Spring Data create an implementation for your interface?**
`@EnableJpaRepositories` scans for `Repository` sub-interfaces and registers a `JpaRepositoryFactoryBean`. The factory instantiates `SimpleJpaRepository` for standard methods, parses custom method names into queries, and builds a **JDK dynamic proxy**. It's `FactoryBean` + reflection + proxies.

**106. ⭐🔥 What does `@Modifying` do, and what are its two flags for?**
Marks a query as an update/delete. Bulk JPQL bypasses the persistence context, so loaded entities go stale and pending changes may be unwritten: `flushAutomatically` writes first, `clearAutomatically` evicts after so reads see new values. Also: no `@Version` bump, no lifecycle callbacks. Requires a transaction.

**107. ⭐ What are projections, and which kind silently doesn't work?**
Interfaces/records selecting a subset of columns — narrower SELECT, nothing managed or dirty-checked, no lazy-loading risk. **Open (SpEL) projections load the whole entity**, defeating the purpose.

**108. 🔥 `Page` vs `Slice`?**
`Page` runs a second `count(*)`. `Slice` fetches size+1 rows and only reports whether there's a next page. For infinite scroll, `Slice` halves the queries and the count is unused.

**109. 🔥 Why does `JOIN FETCH` with `Pageable` warn (`HHH90003004`)?**
Joining a collection multiplies rows, so `LIMIT` would return the wrong number of roots. Hibernate fetches **everything** and paginates in memory — a potential OOM. Fix with `@EntityGraph` on to-one associations, a two-query ID-then-fetch, or `@BatchSize`.

**110. ⭐⭐ How does `@Transactional` work?**
`@Around` advice on a proxy: the `PlatformTransactionManager` begins or joins a transaction bound to the thread via ThreadLocal, invokes the target, commits — or rolls back on `RuntimeException`/`Error`. All proxy limitations apply.

**111. ⭐⭐ Explain propagation, and a real use for `REQUIRES_NEW`.**
`REQUIRED` (default) joins or creates. `REQUIRES_NEW` suspends the current one and creates an independent transaction — for audit logs that must persist despite a business rollback, and per-item batch processing. Costs: a second pooled connection while suspended, no visibility of the outer's uncommitted work, and **silently a no-op when self-invoked**.

**112. ⭐⭐🔥 What is `UnexpectedRollbackException`?**
An inner `REQUIRED` method throws; it can't roll back (it doesn't own the transaction) so it marks the shared transaction **rollback-only**. The caller catches and continues, then commit is refused. Fix: make the inner `REQUIRES_NEW`, or let it propagate, or `noRollbackFor`.

**113. ⭐⭐ Does `@Transactional` roll back on a checked exception?**
**No.** Default is `RuntimeException` and `Error` only; checked exceptions **commit**. Fix with `rollbackFor = Exception.class`, or use unchecked domain exceptions.

**114. ⭐ Optimistic vs pessimistic locking?**
Optimistic (`@Version`): the UPDATE includes the version; zero rows affected = conflict → `OptimisticLockingFailureException` → retry or 409. No locks, no blocking, no deadlocks. Pessimistic (`SELECT FOR UPDATE`): a real lock, for genuine high contention with short critical sections — and it **must** have a timeout.

**115. ⭐🔥 What's `SELECT FOR UPDATE SKIP LOCKED` for?**
Concurrent work queues: each worker locks a batch and others skip locked rows instead of blocking, so N workers pull disjoint batches with no coordination — a job queue on plain Postgres, no broker.

**116. ⭐⭐🔥 Why is an HTTP call inside a `@Transactional` method dangerous?**
It holds a pooled DB connection for the whole round trip, so a slow third party exhausts the pool and takes you down. And it's not atomic: the call can succeed while the transaction rolls back. Fix: publish a domain event and act on it in `@TransactionalEventListener(AFTER_COMMIT)`, or use an outbox.

**117. ⭐⭐ What is the N+1 problem, and how do you *detect* it?**
One query loads N parents; accessing a lazy association fires one query each. Detect with SQL logging, and — the real answer — **assert query counts in tests** via Hibernate `Statistics` or a datasource proxy, because N+1 is a *regression*.

**118. 🔥 Does `Set` fix `MultipleBagFetchException`?**
It removes the exception but **not the Cartesian product** — you've hidden a performance disaster. Real fixes: one collection per query, or `@BatchSize`/subselect.

**119. ⭐ Why is `ddl-auto: update` unacceptable in production?**
It never drops or renames, can't migrate data, produces state-dependent schema (so environments diverge), has no version history or rollback, and multiple instances race on startup. Use Flyway with `ddl-auto: validate`.

**120. ⭐ What's the cardinal rule of Flyway?**
Never edit an applied migration — checksums fail and environments would have run different SQL. Always add a new one.

**121. ⭐⭐🔥 How do you rename a column with zero downtime?**
Expand/contract over four deploys: add the new column and write both → backfill in batches and switch reads → stop writing the old → drop it. Required because rolling deploys run old and new code against one database.

**122. 🔥 Why `CREATE INDEX CONCURRENTLY`, and why `lock_timeout`?**
A plain `CREATE INDEX` blocks writes for its duration. And **any** `ALTER TABLE` needs `ACCESS EXCLUSIVE` — if a long query holds a conflicting lock, the ALTER waits and **every subsequent query queues behind it**, freezing the table. A short `lock_timeout` makes it fail fast instead.

**123. ⭐ Composite index column order?**
Equality predicates first, then range/sort: `(monitor_id, checked_at DESC)` serves `WHERE monitor_id = ? ORDER BY checked_at DESC` with **no sort step**. Leftmost-prefix rule: `(a,b,c)` doesn't help a query filtering only on `b`.

**124. Does Postgres index foreign keys automatically?**
**No** — only primary keys and unique constraints. Missing FK indexes make joins and cascading deletes extremely slow.

**125. 🔥 Would you use Hibernate's second-level cache?**
Usually not in a multi-instance deployment: it's per-JVM, so N pods hold N diverging caches, and cross-instance invalidation needs a distributed provider costing more complexity than it saves. Use an explicit shared Redis cache with a visible TTL.

**126. ⭐🔥 How would you implement multi-tenancy?**
Discriminator column for many small tenants (best scale, weakest isolation); schema-per-tenant for hundreds; database-per-tenant for few regulated tenants. With a discriminator, defend in depth: a Hibernate `@Filter` or unconditional `Specification` predicate, **Postgres RLS** so the database enforces it even with an app bug, and tests asserting cross-tenant queries return nothing.

---

## Module 7 — Security

**127. ⭐⭐ How does Spring Security work?**
A single servlet filter (`DelegatingFilterProxy` → `FilterChainProxy`) selects one `SecurityFilterChain` per request and runs ordered filters: context loading, headers, CORS, CSRF, then authentication filters that build an `Authentication` into `SecurityContextHolder`, then `ExceptionTranslationFilter`, then `AuthorizationFilter`. **All before `DispatcherServlet`**, so a controller only runs for authorized requests.

**128. Explain the authentication flow.**
An auth filter builds an unauthenticated token → `AuthenticationManager` (`ProviderManager`) picks a supporting `AuthenticationProvider` → `DaoAuthenticationProvider` calls `UserDetailsService.loadUserByUsername`, compares via `PasswordEncoder.matches`, runs account checks → the authenticated `Authentication` with authorities is stored in the context.

**129. ⭐⭐ 401 vs 403 — where is the decision made, and what causes an unexpected 403?**
`ExceptionTranslationFilter`: `AuthenticationException` → 401, `AccessDeniedException` → 403 (or 401 if anonymous). Unexpected 403s: **the `ROLE_` prefix**, or **CSRF** blocking a state-changing request.

**130. ⭐ `hasRole` vs `hasAuthority`?**
`hasRole("ADMIN")` checks for `ROLE_ADMIN` — it adds the prefix. `hasAuthority("ADMIN")` matches literally. Mixing them gives a 403 for a user who obviously has the role.

**131. ⭐⭐🔥 What is CSRF, and when can you disable it?**
An attacker's site triggers a state-changing request; the browser attaches cookies automatically so it's authenticated. Disable it for stateless APIs using `Authorization: Bearer`, because **browsers don't attach that header automatically**. **But if you store the token in a cookie you're vulnerable again** — the protection comes from the transport, not the token format.

**132. ⭐ How should you store passwords?**
A delegating `PasswordEncoder` with BCrypt (10–12) or Argon2id — the algorithm prefix in the hash allows migration without password resets. Never a fast hash: password hashes must be deliberately slow, memory-hard, and per-password salted.

**133. 🔥 Should your `User` entity implement `UserDetails`?**
No. It couples the domain to the framework, and a lazily-loaded entity in the `SecurityContext` becomes detached — so touching an association throws from inside a filter. Wrap it in a small immutable `UserDetails`.

**134. ⭐ Why must `.anyRequest().authenticated()` come last?**
Rules evaluate in order, first match wins, so a catch-all first makes later rules dead code. Ending with it gives **deny-by-default**, so a new endpoint is protected until deliberately opened.

**135. What replaced `WebSecurityConfigurerAdapter`?**
`SecurityFilterChain` beans configured via `HttpSecurity`. The adapter was deprecated in Security 5.7 and **removed in 6.0**. Also `antMatchers`→`requestMatchers`, `authorizeRequests`→`authorizeHttpRequests`.

**136. 🔥 How do you prevent user enumeration on login?**
Identical responses for "no such user" and "wrong password." Spring's `DaoAuthenticationProvider` does this by converting `UsernameNotFoundException` to `BadCredentialsException` **and** running a dummy hash when the user is absent, so timing doesn't leak either.

**137. ⭐ Sessions or JWT?**
Depends. For one application with browser clients, a session in Redis is often simpler: revocation is free and state costs little. JWT earns its complexity for multiple independent services validating identity without a central call, or non-browser clients.

**138. ⭐⭐ How do you revoke a JWT?**
Strictly you can't — it's valid until `exp`. Options: short-lived access tokens with revocable stored refresh tokens; a `jti` blacklist; or a `token_version` claim checked against a cached value. **The honest point: the latter two reintroduce state — which is what a session does — so if you need instant revocation, question whether stateless was the right premise.**

**139. ⭐ What is refresh token rotation and reuse detection?**
Each refresh issues a new token and invalidates the old, so a stolen token is useful only until the next refresh. If an already-used token is presented, someone has a copy — you can't tell which party, so you revoke the whole **family** and force re-authentication.

**140. ⭐🔥 `localStorage` or cookies for tokens?**
`httpOnly` + `Secure` + `SameSite=Strict` cookies for browsers: XSS can't read the token so it can't be exfiltrated and reused. Cost is CSRF, handled by `SameSite` + tokens. Good hybrid: refresh token in an httpOnly cookie, access token in JS memory only. And regardless: storage limits blast radius, it doesn't replace output encoding and CSP.

**141. 🔥 HS256 or RS256?**
HS256 when one service issues and validates. RS256/ES256 when multiple services validate — they need only the public key and therefore **cannot mint tokens**, whereas a shared secret would let any service forge for any other.

**142. 🔥 Describe the algorithm confusion attack.**
A server verifies RS256 with a public key. The attacker signs a forged token with **HS256 using that public key as the HMAC secret**. A verifier that lets the token's `alg` choose will validate it — and the key is public. Defence: pin the expected algorithm. Same principle kills `alg: none`.

**143. ⭐ Which claims must you validate beyond the signature?**
`exp`, `iss`, `aud` (and `nbf`). Skipping `aud` lets a token minted for another service be replayed against yours.

**144. ⭐⭐ OAuth2 vs OIDC?**
OAuth2 is **authorization** delegation, yielding an **access token** for calling APIs. OIDC is an **authentication** layer on top, adding an **ID token** (a JWT with identity claims), `/userinfo`, and discovery. "Sign in with Google" is OIDC.

**145. ⭐ What is PKCE and what does it prevent?**
The client sends `SHA256(verifier)` up front and the raw verifier at exchange. It prevents authorization-code interception — on mobile, a malicious app registering the same URI scheme could steal the code, and a public client has no secret. Now recommended for **all** clients; mandatory in OAuth 2.1.

**146. Why are the implicit and password grants deprecated?**
Implicit returned a token in the URL fragment (leaking via history, Referer, logs) with no refresh token. ROPC has the app handle the user's password — defeating OAuth's purpose and breaking MFA and federated login.

**147. What do `state` and `nonce` protect against?**
`state`: CSRF on the callback (login CSRF — linking a victim's session to an attacker's account). `nonce`: ID token replay from another session.

**148. ⭐🔥 Why should you never identify users by email from an ID token?**
Emails change and get reassigned, and some providers don't verify them — so matching on email allows **account takeover**. Key on `(iss, sub)`, store a `FederatedIdentity`, and trust email only when `email_verified` is true.

**149. 🔥 Describe the confused deputy attack.**
An app that authenticates by dereferencing a client-supplied *access* token can be fed one issued to a **different** client, calls `/userinfo`, and logs the attacker in as that user. Access tokens have no binding to your client; ID tokens do, via `aud` — which is exactly why OIDC exists.

**150. ⭐ Should code check roles or permissions?**
Permissions. Roles are business vocabulary that changes; permissions are technical capabilities that don't. Model roles as bundles of permissions so a new role is one enum change, not an audit of every `@PreAuthorize`.

**151. ⭐⭐🔥 What is IDOR and how do you prevent it?**
An authenticated, permitted user accesses a resource they don't own by supplying its ID. Prevent by **scoping the query** (`findByIdAndOrgId`) so unauthorized rows are never loaded — stronger than a post-load check, which can be forgotten. Return **404, not 403**, to avoid confirming existence. And test it.

**152. ⭐🔥 Why is `@PreAuthorize` on a self-invoked method a security hole?**
Method security is proxy-based, so `this.method()` bypasses the proxy and **the check never runs** — silently. It's an authorization bypass, not a performance quirk.

**153. ⭐ Which Actuator endpoints must never be public, and what's the best control?**
`/env`, `/configprops` (secrets), `/heapdump` (**every credential and PII value in memory, in plaintext**), `/threaddump`, `/mappings`, `/loggers` (writable). Best control: `management.server.port` on a separate port the load balancer doesn't expose.

**154. 🔥 Why is Vigil especially exposed to SSRF?**
Its core function is fetching user-supplied URLs. Defences: scheme allowlist; block private/loopback/link-local IPs (notably `169.254.169.254`, which serves cloud IAM credentials); **check the resolved IP at connection time** (DNS rebinding); validate redirects; never return the response body. **And network egress isolation, so internal addresses are unreachable even with an app bug.**

---

## Module 8 — Testing

**155. ⭐ Why write tests?**
Primarily to make change safe — without them, every modification carries unbounded risk, refactoring stops, and the codebase calcifies. It follows that a test breaking on every refactor is actively harmful.

**156. ⭐ When should you NOT mock?**
Value objects and data (build them). **Types you don't own** — your mock encodes your possibly-wrong belief about the library; use the real thing or a fake server. The class under test. And avoid mocking so much that the test only verifies Mockito.

**157. Mock vs stub vs fake?**
Stub returns canned values. Mock is a stub you also assert interactions on. **Fake is a working lightweight implementation** — often the best option: consistent behaviour, no per-test setup, tests read as behaviour rather than mock choreography.

**158. 🔥 Why avoid `@InjectMocks`?**
It sets fields reflectively, so a new dependency is silently null and the test NPEs. Explicit construction breaks **compilation** of every test — immediate, precise feedback.

**159. 🔥 What's wrong with over-verifying interactions?**
It tests implementation, not behaviour, so pure refactors break tests — taxing the very activity tests exist to enable. Assert outcomes; verify only interactions that *are* the outcome.

**160. Is 100% coverage a good goal?**
No. Coverage measures execution, not correctness — an assertion-free test gives 100%. Use 70–80% as a smell detector. Better questions: does every fixed bug have a regression test, and do tests fail for the right reasons (which mutation testing measures)?

**161. How do you test time-dependent logic?**
Inject a `Clock` and use `Clock.fixed(...)`, so time is an input. Never `Thread.sleep`.

**162. ⭐🔥 How does Spring cache test contexts, and why does it matter?**
Contexts are cached per *configuration* — classes, profiles, property sources, `@MockitoBean` definitions, customizers. **Every distinct configuration builds another context.** Scattered `properties = {...}` and varied mocks are the usual cause of slow suites. `@DirtiesContext` evicts the cache.

**163. ⭐⭐ Why not use H2 for tests?**
It's a different database: no real `jsonb`, no `FOR UPDATE SKIP LOCKED`, no `percentile_cont`/`FILTER`, no partial indexes, no RLS, different plans and error codes. Tests pass, production breaks. Testcontainers runs the actual version you deploy.

**164. What does `@ServiceConnection` do?**
Boot 3.1+ auto-configures connection properties from a container bean, replacing hand-written `@DynamicPropertySource` blocks.

**165. ⭐🔥 Why must you call `em.flush()` and `em.clear()` in repository tests?**
`flush()` sends the SQL so constraints actually fire. `clear()` empties the context so the query hits the **database** rather than the first-level cache. **Without `clear()`, a repository test can pass with a completely broken column mapping.**

**166. 🔥 What breaks about `@Transactional` rollback under `RANDOM_PORT`?**
The request runs on a Tomcat thread with its own transaction, so the test's rollback doesn't undo what it committed. Clean up explicitly.

**167. ⭐ How do you test outbound HTTP calls, and which test matters most?**
WireMock — a real server you control. The most valuable test: `withFixedDelay` proving your **read timeout actually fires**, because an unenforced timeout is what causes pool exhaustion and cascading outages, and almost nobody tests it.

**168. ⭐🔥 How do you stop architecture from eroding?**
ArchUnit tests in the normal suite: layering, feature boundaries, cycles, and correctness rules like "`@Transactional` methods must be public" — which catches the proxy no-op that produces no error.

**169. 🔥 Coverage vs mutation testing?**
Coverage says which lines ran. Mutation testing (PIT) modifies your code — flipping conditionals, removing void calls — and checks whether a test fails. A surviving mutant is a real undetected bug. **Coverage tells you what ran; mutation testing tells you whether your assertions matter.**

**170. ⭐🔥 What do you do about flaky tests?**
Quarantine with a ticket, owner and date — **never retry.** Retrying hides the cause, and a flaky test is often reporting a real race condition in the *code*. Then fix the root: inject a `Clock`, scope assertions to data the test created, replace sleeps with Awaitility, use random ports, stub external HTTP.

**171. Most common cause of flakiness?**
Shared state — usually an absolute count over a shared table. Fix by scoping every assertion to the test's own data, which also unlocks parallel execution.

**172. 🔥 Should coverage be a build gate?**
An absolute threshold is poor — it punishes touching legacy code and incentivizes assertion-free tests. Better: **coverage must not decrease on changed files.**

---

## Module 9 — Production

**173. ⭐⭐ What's wrong with the default `@Async` executor?**
`SimpleAsyncTaskExecutor` — a new thread per call, unbounded and unpooled. With virtual threads it's less catastrophic but still unbounded, moving the failure to the connection pool. Always define a named, bounded executor.

**174. ⭐🔥 Explain `ThreadPoolTaskExecutor`'s growth behaviour.**
Create up to `corePoolSize` → then **queue** → when the queue is full, create up to `maxPoolSize` → then reject. **The trap: a large queue means `maxPoolSize` is never reached** — `core=4, max=100, queue=10000` behaves as a 4-thread pool.

**175. Which rejection policy, and the caveat?**
`CallerRunsPolicy` usually — the submitter executes it, forcing the producer to slow down (natural backpressure). Caveat: if the caller is a Tomcat request thread, you've made a request thread do background work.

**176. ⭐ `fixedDelay` vs `fixedRate`?**
`fixedDelay` measures the gap **after completion**, so runs never overlap. `fixedRate` measures from the previous start, so a long run makes executions queue up. Default to `fixedDelay`.

**177. ⭐🔥 What happens if a `@Scheduled` task throws?**
For `fixedDelay`/`fixedRate` it's **cancelled permanently** — never runs again until restart, with no alert. Always catch inside, or set an `ErrorHandler` on the scheduler.

**178. How many threads run `@Scheduled` tasks by default?**
**One**, shared by all — so one slow task starves every schedule. Configure `spring.task.scheduling.pool.size`.

**179. ⭐⭐ You deploy three replicas. What happens to `@Scheduled`?**
Every instance runs every schedule — triplicated reports, concurrent purges, monitors checked three times. Fixes: **ShedLock** for singleton jobs, or **`FOR UPDATE SKIP LOCKED`** work claiming so every instance takes a disjoint batch (better for high volume — it scales with replicas), or clustered Quartz.

**180. 🔥 ShedLock's `lockAtMostFor` and `lockAtLeastFor`?**
`lockAtMostFor` releases the lock if the holder dies — it must exceed the job's worst-case runtime or a second instance starts while the first runs. `lockAtLeastFor` prevents a double-run from clock skew on fast jobs. Use `.usingDbTime()` so the DB clock is authoritative.

**181. Are Spring application events synchronous?**
Yes by default — the publisher's thread, inside its transaction.

**182. ⭐⭐ What does `@TransactionalEventListener(AFTER_COMMIT)` solve?**
The listener runs only if the transaction committed, so you never notify about rolled-back work and no I/O happens inside the transaction. Gotchas: DB writes there need `REQUIRES_NEW`, and outside a transaction it doesn't run unless `fallbackExecution = true`.

**183. ⭐⭐🔥 What is the transactional outbox pattern?**
Write the outgoing message to a table **in the same transaction** as the business change, then a poller publishes it. It makes "the change happened" and "the message will be sent" one atomic fact — which `AFTER_COMMIT` can't guarantee, since the process can die between commit and send. Consequence: at-least-once, so consumers must be idempotent.

**184. ⭐🔥 What's the most dangerous caching bug?**
A cache key omitting the tenant. A no-argument method reading the tenant from a ThreadLocal caches under one key for everyone — **one tenant's data served to another.** A breach, not a performance issue.

**185. ⭐ Local vs distributed cache?**
Local (Caffeine) is per-JVM, so with N instances an eviction affects only 1/N of traffic and users see stale data on some requests. Redis is one shared cache with global evictions — the right default for a multi-instance service.

**186. ⭐⭐ What is a cache stampede and how do you prevent it?**
A popular key expires and all concurrent requests miss simultaneously, hammering the source — hardest exactly when you're busiest. Fixes: `sync = true` (**per-JVM only**), a distributed lock, **stale-while-revalidate** (serve stale, refresh async — no user ever waits), and **TTL jitter** so bulk-loaded keys don't expire together.

**187. 🔥 Explain the `@CacheEvict`/commit race.**
`@CacheEvict` fires when the method returns, but the transaction commits later. In that window a concurrent request misses, reads the **old** committed row, and re-caches it — stale for the whole TTL. Fix with `transactionAware()` or evicting from `@TransactionalEventListener(AFTER_COMMIT)`.

**188. ⭐🔥 What happens when Redis goes down?**
It must degrade, not fail: a short command timeout, a `CacheErrorHandler` swallowing GET/PUT failures, a circuit breaker, and a health indicator reporting **DEGRADED** not DOWN. **Note the asymmetry — a failed EVICT means knowingly serving stale data, so swallowing that must be deliberate.**

**189. ⭐⭐ What's the most important resilience practice?**
Timeouts on every network call. Defaults are effectively infinite, so a hung upstream blocks threads until every endpoint fails — your service goes down because someone else's is slow.

**190. ⭐ When should you retry, and when must you not?**
Retry transient failures: connection errors, timeouts, 502/503/504, 429 (honouring `Retry-After`), deadlocks, optimistic-lock conflicts. **Never 4xx** — the request is wrong and will be wrong again, so you triple load and mask the real error.

**191. ⭐🔥 Why is jitter essential in backoff?**
Without it, all clients that failed together retry together — a **retry storm** that knocks the recovering dependency back over as soon as it returns.

**192. 🔥 What is retry amplification?**
Retries at multiple layers multiply: three chained services each retrying three times = 27 requests to the deepest, at its worst moment. Retry at **one** layer.

**193. ⭐ Explain a circuit breaker's states.**
CLOSED (calls pass, failures counted) → **OPEN** when the failure rate over a sliding window exceeds the threshold (calls fail instantly, no thread blocked) → after `waitDuration`, **HALF_OPEN** admits trial calls → success closes, failure reopens. It protects both sides.

**194. 🔥 Why must Retry wrap CircuitBreaker?**
So each attempt passes through the breaker, and once open, retries fail instantly. Reversed, you'd retry three times per call against a known-dead service.

**195. ⭐ What is a bulkhead?**
A cap on concurrent calls per dependency. Without it, all request threads can block on one slow dependency so unrelated endpoints fail. It prevents "one dependency broke and the whole service went down."

**196. 🔥 What does `slow-call-rate-threshold` catch?**
A dependency that responds *successfully* but takes 30 seconds — functionally down, never errors, so a pure error-rate breaker stays closed while threads pile up.

**197. ⭐⭐ Why does idempotency matter, and what's the unavoidable case?**
Every retry mechanism can deliver an operation twice. **The unavoidable case: a request succeeds but its response is lost, so the client legitimately retries something that already happened.** No server-side correctness prevents that.

**198. ⭐ How do you implement an idempotency key?**
Client sends `Idempotency-Key`; the server claims it atomically (`SETNX` + TTL), executes, and stores the **response** so a retry replays the original. Fingerprint the body so the same key with a different payload → 422. Delete on failure. Stripe's API is the reference.

**199. 🔥 What's better than an idempotency key?**
Natural idempotency: a unique constraint + `ON CONFLICT DO NOTHING`, a find-or-create, or a state-transition guard. The database enforces it, so it can't be forgotten.

**200. 🔥 What makes a bad fallback?**
Returning a plausible fake value. `return Uptime.of(100)` when the calculation failed means your dashboard shows green during an outage. **A fallback that lies is worse than an error** — it removes the operator's ability to know something is wrong.

**201. ⭐⭐🔥 Liveness vs readiness, and the classic mistake?**
Liveness = "irrecoverably broken?" → the pod is **killed**. Readiness = "can it serve now?" → removed from the load balancer. **Checking the database in liveness means a brief DB blip kills every pod, they restart, overwhelm the recovering database, and loop** — a 30-second blip becomes a total outage. Liveness = ping; readiness = essential dependencies only.

**202. ⭐🔥 What is high cardinality and why does it matter for metrics?**
Each unique tag combination is a time series. A user or monitor ID as a tag creates thousands from one metric, destabilizing Prometheus and generating large bills. **If you can't enumerate the values, it isn't a tag** — put it in logs or traces.

**203. ⭐🔥 Why are averages useless for latency, and what's the percentile trap?**
An average hides the tail: 99 × 10ms + 1 × 10s averages 110ms. Use p95/p99. **The trap: percentiles can't be averaged across instances** — export histogram buckets so Prometheus can compute fleet quantiles with `histogram_quantile()`.

**204. ⭐ What should `ERROR` mean?**
"A human must act." Using it for expected conditions trains operators to ignore errors. For Vigil, a failed *check* is INFO; a failure to *record* one is ERROR.

**205. 🔥 How do you handle trace sampling?**
Head-based (Spring's default) decides up front — cheap, but may discard the request you care about. **Tail-based** at the collector buffers spans and retains slow/errored traces. Common hybrid: 1% of successes, 100% of errors.

**206. ⭐ What should you alert on?**
User-visible symptoms — error rate, latency, and business signals like `checks.overdue` — not causes like CPU. Every alert must be actionable; **alert fatigue is the real failure mode.** Modern practice: multi-window error-budget burn-rate alerts.

**207. 🔥 How do you monitor a monitoring system?**
Externally. A shared failure takes out both the service and its alerting — use a third-party check or a dead-man's-switch heartbeat that alerts when it *stops* reporting. Same principle as not storing backups on the machine you're backing up.

**208. ⭐⭐ High latency with low CPU — what does that tell you?**
You're **waiting, not computing.** Blocking I/O: slow queries, N+1, a slow downstream, or pool exhaustion. No JVM tuning will help. Confirm with a thread dump — many threads in the same frame identify the bottleneck.

**209. ⭐⭐🔥 Why do Spring Boot pods get OOMKilled?**
Heap isn't total memory — metaspace, code cache, thread stacks (~1MB each) and direct buffers are outside it. `-Xmx512m` in a 512MB container is killed by the kernel: **exit code 137, no Java exception, no heap dump, nothing in the logs.** Use `-XX:MaxRAMPercentage=75`.

**210. 🔥 How do you find a memory leak?**
GC log shows old gen growing monotonically after each full GC — that's the signature. Then `jcmd GC.class_histogram`, and a heap dump in Eclipse MAT using the **dominator tree** to find what *retains* memory. Common: unbounded caches, uncleared ThreadLocals on pooled threads, static collections, oversized persistence contexts.

**211. 🔥 How would you improve startup time?**
Measure first (`BufferingApplicationStartup` + `/actuator/startup`). Then trim dependencies, **CDS/AOT caching** (20–40% for no code change — best value), Spring AOT, and native image only if cold start dominates. Lazy init is fastest but moves errors to the first request.

**212. 🔥 GraalVM native image trade-offs?**
Gains: ~50ms startup, ~50MB. Costs: no dynamic class loading (hence Spring AOT's reflection hints), long builds, **lower peak throughput because there's no JIT**, library incompatibility. It's a startup-and-footprint optimization, not a throughput one.

**213. 🔥 What tools would you use to profile a production JVM?**
**JFR** first — built in, ~1% overhead, safe live, captures GC/allocation/locks/JDBC in one file. **Async-profiler** for flame graphs, using **`-e wall`** for latency (CPU mode shows nothing for I/O-bound work). Three thread dumps 10s apart for hangs. `pg_stat_statements` ordered by **total** time.

**214. 🔥 Why order `pg_stat_statements` by total time?**
A 5ms query called 100,000 times costs more than a 2-second query called once, and it's invisible sorted by mean. High-count-low-mean is the N+1 signature.

---

## Module 10 — Messaging & microservices

**215. ⭐ How does Kafka differ from a queue?**
It's a distributed, replicated, append-only **log**: messages are retained rather than removed, each consumer group tracks its own offset, so many groups read the same data and any can **replay**. Ordering is per-partition only.

**216. ⭐⭐ What determines the partition, and why does it matter?**
`hash(key) % partitions`. Same key → same partition → ordered relative to each other. **Choosing the key is the most important Kafka design decision:** keying by entity gives per-entity ordering with cross-entity parallelism.

**217. 🔥 How does consumer parallelism work, and what's the partition-count trap?**
Consumers in a group split partitions; **partition count caps useful parallelism.** You can add partitions but not remove them, and adding **changes the key→partition mapping**, breaking ordering across the change. Over-provision modestly.

**218. ⭐⭐ Explain the delivery guarantees precisely.**
At-most-once (commit before processing — loses messages), at-least-once (commit after — duplicates, the practical default), exactly-once. **Exactly-once *delivery* is impossible** — you can't distinguish a lost ack from a lost message. Exactly-once *processing* is achievable: Kafka transactions for Kafka→Kafka, or at-least-once + an idempotent consumer once external state is involved.

**219. ⭐⭐ Why must you disable auto-commit?**
It commits on a timer regardless of processing success, so a crash after a commit but before processing **silently skips records** with no error.

**220. ⭐⭐🔥 What's a poison pill and how do you handle it?**
A message that always fails. Kafka preserves ordering, so you can't skip it — the partition stalls indefinitely. Fix: `DefaultErrorHandler` + `DeadLetterPublishingRecoverer` with bounded retries, then publish to `<topic>.DLT` and advance. Classify permanent exceptions as non-retryable, and **build a replay tool** — an unmonitored DLT is a data-loss log.

**221. ⭐⭐ How do you publish a message and update a database atomically?**
You can't — the dual-write problem. Use `@TransactionalEventListener(AFTER_COMMIT)`, the **transactional outbox**, or **Debezium CDC** reading the write-ahead log.

**222. What is `acks=all` and `enable.idempotence`?**
`acks=all`: wait for all in-sync replicas, so with `min.insync.replicas ≥ 2` you don't lose data on broker failure. `enable.idempotence`: a producer id + sequence number so the broker deduplicates retried writes — exactly-once *producing*.

**223. 🔥 What is a rebalance, and the `max.poll.interval.ms` trap?**
Partition reassignment when consumers join or leave; consumption pauses (use `CooperativeStickyAssignor` for incremental rebalancing). **If processing exceeds `max.poll.interval.ms`, the consumer is presumed dead and evicted mid-work**, so its records get reprocessed — the classic "duplicate processing under load."

**224. ⭐ How do you evolve a message schema safely?**
Only additive, optional changes with defaults. Never rename or retype. Consumers must tolerate unknown fields **and unknown enum values**. Enforce with a schema registry in `BACKWARD` mode so an incompatible schema is rejected at publish time. Set `spring.json.trusted.packages`.

**225. 🔥 Kafka or RabbitMQ?**
Kafka for high-volume event streams needing replay and multiple independent consumers. RabbitMQ for task queues needing rich routing, priorities or delays — and simpler to operate. SQS on AWS for zero ops. **"We need Kafka" is often "we need a queue," and Kafka's operational cost is real.**

**226. What's the single most important Kafka consumer metric?**
**Consumer lag.** A stalled consumer looks perfectly healthy — JVM fine, no errors, health checks pass — while messages pile up.

**227. ⭐⭐⭐🔥 Would you use microservices for a new project?**
No, by default. Start with a modular monolith: feature packages with enforced boundaries, no cross-feature shared tables, communication via service interfaces or domain events. Extract for a concrete reason — a genuinely different scaling profile, a team needing independent deploys, or isolation — along a boundary that already exists. **Microservices solve an organizational scaling problem; a small team pays the full distributed-systems cost for a benefit it can't collect.**

**228. ⭐⭐ What do you lose when you split a monolith?**
ACID transactions across features (→ sagas), fast typed in-process calls, compiler-verified refactoring across boundaries, single-stack-trace debugging, a trivial local environment, cheap cross-domain queries. Plus service discovery, a gateway, per-service pipelines, databases and migrations.

**229. ⭐ Why is a shared database an anti-pattern?**
The schema becomes a contract shared by every service, so one change breaks several and deploys must be coordinated. You've added latency and partial failure while keeping all your coupling — a distributed monolith, strictly worse than either alternative.

**230. ⭐🔥 How do you decide where boundaries go?**
Primarily **transactional boundaries** — things that must change atomically belong in one service. Then rate of change, team ownership, data access patterns, failure isolation. **Never by technical layer** (a distributed n-tier app needs a change in every service per feature) and never one-service-per-entity.

**231. ⭐ What is a saga, orchestration vs choreography?**
A sequence of local transactions with compensating actions. Orchestration: a coordinator drives — explicit and debuggable, but coupled and **must be durable**. Choreography: services react to events — decoupled, but the flow exists nowhere. Default to orchestration; beyond ~3 steps use a durable workflow engine (Temporal), because crash-resumption is the actual hard part.

**232. 🔥 What's wrong with compensating transactions?**
They aren't rollbacks. Some actions can't be undone (a sent email). Compensations can themselves fail. And **intermediate states are visible to other users** — mitigate by creating resources `PENDING` and only activating on saga completion.

**233. ⭐⭐🔥 How do you avoid sagas?**
**Redraw the boundary.** Most sagas exist because things needing one transaction were placed in different services. That's the aggregate rule applied to boundaries, and it's usually better engineering than writing the saga.

**234. 🔥 How do you query data spanning services?**
API composition (N calls; availability is the product), a **CQRS read model** built from events (one local query, seconds stale — the standard for dashboards), replication, or a BFF. State where the staleness trade-off breaks: fine for uptime, not for balances.

**235. Explain CAP precisely.**
Under a partition you choose consistency or availability. **Not "pick two of three"** — partitions are unavoidable. **PACELC** extends it: during a Partition choose A or C, **Else** choose Latency or Consistency — and that "else" is the trade you make every day.

**236. ⭐ How do you migrate a monolith to services?**
Strangler fig: a facade in front with no behaviour change, then extract one capability at a time and route just that path, with rollback available at every step. Start with something well-bounded that has an independent scaling need. Use dual-run/shadow traffic to compare responses before cutting over.

**237. 🔥 Do you need Eureka on Kubernetes?**
No. Kubernetes Services provide DNS discovery, load balancing and health checking. Eureka solved a pre-Kubernetes problem.

**238. ⭐⭐ Authentication or authorization at the gateway?**
**Authentication at the edge, authorization in the service.** The gateway can validate a token; it can't answer "may *this* user delete *this* monitor?" without the data.

**239. ⭐🔥 Should services re-validate the JWT?**
Yes. Trusting headers makes the network the security boundary — the assumption zero-trust exists to reject. Validation is cheap (JWKS is cached). If services must trust headers, mTLS becomes mandatory.

**240. ⭐⭐🔥 What are the two most dangerous gateway misconfigurations?**
**Forwarding client-supplied internal headers** — if downstream trusts `X-User-Id`, any client impersonates any user: a total authentication bypass. And **retrying non-idempotent methods** — a retried POST after a timeout creates duplicates.

**241. 🔥 Would you use a service mesh?**
Worth it for many (especially polyglot) services needing uniform mTLS and traffic policy without app code. For a handful of Spring services, Resilience4j + Kubernetes Services + Micrometer covers most of it far more cheaply. **And if you adopt one, move resilience policy out of the application** — both layers retrying compounds amplification.

---

## Module 11 — Deployment

**242. ⭐ How do you containerize Spring Boot properly?**
Multi-stage build, dependencies as a cached layer, **extract the fat JAR into layers** (`-Djarmode=tools extract --layers`) so a code change pushes ~400KB not 60MB, non-root user, `-XX:MaxRAMPercentage`, healthcheck. Or Buildpacks (`spring-boot:build-image`), which does all of it.

**243. ⭐⭐🔥 How do you achieve genuinely zero-downtime deploys?**
Four things: `maxUnavailable: 0`; **a `preStop` sleep**, because endpoint removal propagates asynchronously and traffic still arrives after SIGTERM; `server.shutdown=graceful` so in-flight requests finish; and `terminationGracePeriodSeconds` > preStop + shutdown timeout. **Missing the preStop sleep is why people see 502s despite configuring graceful shutdown.**

**244. ⭐⭐ When do you run migrations relative to deployment?**
Additive first, then deploy — old code ignores new columns. Destructive changes never in one step: **expand/contract**, because rolling deploys run old and new code against one database. Prefer a Job or init container over startup migrations. Never put a multi-million-row backfill in a migration.

**245. 🔥 Does blue/green free you from migration compatibility?**
No. Both environments share one database, so a destructive migration breaks the old one immediately. **Expand/contract is required regardless of deployment strategy.**

**246. Why tag with a commit SHA and deploy by digest?**
`latest` is ambiguous, racy and unrollbackable. A digest is immutable, so you know exactly what's running. And promote **one** artifact — rebuilding per environment means you didn't test what you shipped.

**247. 🔥 Resource requests vs limits, and the subtle one?**
Requests are the scheduler's reservation; limits are the hard cap. Exceeding memory = instant OOMKill; exceeding CPU = **throttling**, which produces periodic latency spikes that look like GC pauses. Check `container_cpu_cfs_throttled_seconds_total` before blaming the JVM.

**248. ⭐ Which 12-factor principles matter most?**
Config in the environment (one artifact promoted, or you don't test what you ship), stateless processes (or instances aren't interchangeable), disposability, dev/prod parity (the same database, not H2), logs to stdout.

---

## The 15 questions to be perfect on

If you only have an hour, drill these. They come up most, and they're where depth shows.

1. **#59** — Walk me through a request (the flagship web question)
2. **#46** — How auto-configuration works (the flagship Boot question)
3. **#40** — Self-invocation (the flagship AOP question)
4. **#89, #94, #95** — Persistence context, `LazyInitializationException`, `open-in-view`
5. **#110–113** — `@Transactional`: mechanism, propagation, `UnexpectedRollbackException`, checked exceptions
6. **#117** — N+1: what it is, and how you *detect* it
7. **#127, #129** — Spring Security's filter chain, 401 vs 403
8. **#151** — IDOR and query scoping
9. **#131** — CSRF, and why "we use JWT" isn't sufficient justification
10. **#179** — Three replicas and `@Scheduled`
11. **#183** — The transactional outbox
12. **#189, #191, #197** — Timeouts, jitter, idempotency
13. **#201** — Liveness vs readiness, and the cascade
14. **#227** — Would you use microservices? (the flagship architecture question)
15. **#243** — Zero-downtime deploys and the preStop sleep

---

**Next:** [Lesson 45 — System design & senior signals](45-system-design-and-senior-signals.md) — design rounds, behavioural questions, and the things that mark you as senior.

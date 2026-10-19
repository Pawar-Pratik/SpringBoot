# Lesson 08 — Scopes, lifecycle & conditional beans

> **Why this lesson exists:** Scope determines *how many* instances exist, which is a correctness question (get it wrong and you leak one user's data to another). Lifecycle determines *when* your code runs relative to Spring's, which is where startup and shutdown bugs live. Conditional beans are the mechanism auto-configuration is built on — understand them here and Lesson 11 becomes trivial.

**Time:** ~65 minutes · **Prereq:** Lessons 06–07

---

## Part 1 — The idea in one sentence

> **Three questions decide a bean's existence — *how many* (scope), *when* (lifecycle), and *whether at all* (conditions) — and nearly every bug in this area is a bean answering one of them on a different thread, or in a different phase, than you assumed.**

The prototype-in-a-singleton trap, the `@PostConstruct` proxy trap, and request scope dying inside `@Async` are the same mistake three times: code that assumes a bean's scope or phase matches the caller's.

---

## Part 2 — Scopes

### The six scopes

| Scope | Instances | Lifetime | Use |
|---|---|---|---|
| **`singleton`** (default) | One per container | App lifetime | ~99% of beans: services, repos, controllers, config |
| **`prototype`** | New one **per injection/lookup** | Caller's — Spring forgets it | Stateful helpers, builders |
| `request` | One per HTTP request | Request | Per-request context |
| `session` | One per HTTP session | Session | Shopping carts (legacy style) |
| `application` | One per `ServletContext` | App | Rare; ≈ singleton |
| `websocket` | One per WebSocket session | Connection | Rare |

```java
@Service                                    // singleton — implicit
@Scope("prototype")
@Scope(value = WebApplicationContext.SCOPE_REQUEST,
       proxyMode = ScopedProxyMode.TARGET_CLASS)
@RequestScope                               // shorthand, includes the proxy mode
@SessionScope
```

### `singleton` — and the misconception about it

**A Spring singleton is one instance per `ApplicationContext`, not one per JVM.** That's a different thing from the Singleton *design pattern* (a private constructor and a static `getInstance()`). Two contexts in one JVM (common in tests!) mean two instances. Interviewers ask this specifically.

**The consequence you must never forget:** *singleton = shared across every thread.* Everything in Lesson 02 §7 applies. `final` dependencies, no mutable instance state.

**Eager by default,** which is deliberate fail-fast (Lesson 06). You can override:

```java
@Service @Lazy                                  // per-bean
public class HeavyPdfRenderer { }
```
```properties
spring.main.lazy-initialization=true            # global — see the warning below
```

Global lazy init cuts startup time noticeably, and it's a **known trap**: configuration errors now surface on first request instead of at boot, and your first request pays the initialization cost. It's reasonable for local development, dangerous as a production default. Say exactly that if asked.

### `prototype` — and the trap that makes it nearly useless as people expect

```java
@Component @Scope("prototype")
public class CheckSession {
    private final List<String> log = new ArrayList<>();     // per-instance state
}
```

```java
@Service
public class CheckRunner {
    private final CheckSession session;                      // ⚠️ injected ONCE

    public CheckRunner(CheckSession session) { this.session = session; }

    public void run() {
        session.log(...);       // ← the SAME instance every call, forever
    }
}
```

**A prototype injected into a singleton is created exactly once.** The singleton is created once, so injection happens once. You have a de-facto singleton with a misleading annotation. This is a top-tier interview question because it looks like it should work.

**Three correct ways to get a fresh instance per call:**

```java
// 1. ObjectProvider — the modern, cleanest option
@Service
public class CheckRunner {
    private final ObjectProvider<CheckSession> sessions;
    public CheckRunner(ObjectProvider<CheckSession> sessions) { this.sessions = sessions; }

    public void run() {
        CheckSession session = sessions.getObject();     // ✅ fresh every call
    }
}
```

```java
// 2. @Lookup — Spring overrides the method via CGLIB to return a new bean
@Service
public abstract class CheckRunner {
    @Lookup
    protected abstract CheckSession newSession();          // ✅ fresh every call

    public void run() { CheckSession s = newSession(); }
}
```

```java
// 3. Scoped proxy — inject a proxy that resolves a new target per call
@Component
@Scope(value = "prototype", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class CheckSession { }
```

**And the honest senior answer:** if you need a fresh object per operation, **just use `new`**. A prototype bean is only worth it when the new object itself needs dependencies injected. Reaching for `prototype` where `new` would do is a common over-engineering tell.

**Also crucial:** *Spring does not manage a prototype's destruction.* No `@PreDestroy`, no cleanup. If it holds a resource, **you** must close it — which is another reason `new` + try-with-resources is usually better.

### `request` and `session` scope — and why they need a proxy

```java
@Component
@RequestScope
public class RequestContext {
    private String correlationId;
    private String tenantId;
}
```

```java
@Service                                    // SINGLETON
public class MonitorService {
    private final RequestContext ctx;        // request-scoped?! into a singleton?!

    public MonitorService(RequestContext ctx) { this.ctx = ctx; }
}
```

The singleton is created at startup, when no HTTP request exists. So what gets injected?

**A scoped proxy.** `@RequestScope` implies `proxyMode = TARGET_CLASS`, so Spring injects a proxy that, on *every method call*, looks up the real request-scoped instance for the **current thread** and delegates to it.

```
Singleton MonitorService
   └── holds → RequestContext PROXY (created once at startup)
                    │  on each method call:
                    └──▶ look up the instance bound to the current thread's request
                         → delegate
```

Without `proxyMode`, you get:
```
BeanCreationException: Scope 'request' is not active for the current thread;
consider defining a scoped proxy for this bean
```
Spring's error even tells you the fix. This is a great example of why reading error messages properly beats guessing.

**Where the request-scoped instance lives:** `RequestContextHolder`, backed by a `ThreadLocal`. Which sets up the trap:

```java
@Service
public class BadAsyncService {
    private final RequestContext ctx;

    @Async
    public void doLater() {
        ctx.getTenantId();     // 💥 different thread → no request bound → exception
    }
}
```

**Request scope is thread-bound, so it breaks across `@Async`, `@Scheduled`, executors, and reactive threads.** Same reason `SecurityContextHolder` and MDC logging context need explicit propagation (Lessons 32, 35). This single mechanism explains three separate categories of production bug — worth holding onto.

> **Virtual threads note:** with `spring.threads.virtual.enabled=true` each request gets its own virtual thread, so ThreadLocal-based scoping still works. But `ThreadLocal` on millions of short-lived virtual threads is a memory concern, which is why Java added **scoped values** (`ScopedValue`) as a better-fitting mechanism. Frameworks are migrating gradually. Mentioning this in a virtual-threads discussion is a strong signal.

**Modern alternative worth knowing:** rather than request-scoped beans, most teams pass context explicitly (a parameter or a small context object) or use a `ThreadLocal` holder directly. Explicit is easier to test and doesn't break on async. Request scope is the older idiom.

### Custom scopes (know they exist)

Implement `org.springframework.beans.factory.config.Scope` and register it. Realistic use: **tenant scope** in a multi-tenant SaaS — one bean instance per tenant, cached. Rare, but a nice answer to "have you ever extended Spring?"

---

## Part 3 — Lifecycle

### The full ordering, one bean at a time

```
1.  Instantiate                  ← ctor.newInstance(resolvedDependencies)
2.  Populate fields              ← @Autowired fields, @Value
3.  BeanNameAware.setBeanName()
    BeanFactoryAware / ApplicationContextAware  (avoid these — they couple you to Spring)
4.  BeanPostProcessor.postProcessBeforeInitialization()
5.  @PostConstruct                                          ← YOUR init hook
6.  InitializingBean.afterPropertiesSet()
7.  @Bean(initMethod = "...")
8.  BeanPostProcessor.postProcessAfterInitialization()      ← ★ PROXY CREATED HERE
    ──── bean is now live and in the singleton cache ────
    ... application runs ...
9.  @PreDestroy                                             ← YOUR cleanup hook
10. DisposableBean.destroy()
11. @Bean(destroyMethod = "...")
```

**Init hooks, ranked:**

```java
// ✅ BEST: constructor. Dependencies are final and validated; nothing else needed.
public MonitorService(MonitorRepository repo) {
    this.repo = Objects.requireNonNull(repo);
}

// ✅ GOOD: @PostConstruct — for work that needs the fully injected object
@PostConstruct
void warmCache() { statusCache.putAll(repo.findAllStatuses()); }

// ⚠️ OK: InitializingBean — same timing, but couples your class to Spring's interface
public void afterPropertiesSet() { }

// ✅ GOOD for third-party classes you can't annotate
@Bean(initMethod = "start", destroyMethod = "stop")
KafkaConsumerRunner consumer() { return new KafkaConsumerRunner(); }
```

**The rule: prefer the constructor.** If it must happen after injection, use `@PostConstruct`. Everything else is for special cases.

> `@PostConstruct` / `@PreDestroy` come from `jakarta.annotation` (formerly `javax.annotation`). In Boot 3 they need `jakarta.annotation-api`, which the standard starters include. This is a real Boot 2→3 migration papercut.

### THE `@PostConstruct` TRAP — remember step 8

Proxies are created **after** `@PostConstruct`. So inside it, `this` is the raw target:

```java
@Service
public class BadWarmup {

    @PostConstruct
    @Transactional                            // ❌ NO TRANSACTION. Proxy doesn't exist yet.
    void loadInitialData() { ... }

    @PostConstruct
    void kickOff() {
        doAsyncWork();                        // ❌ runs SYNCHRONOUSLY — self-invocation + no proxy
    }

    @Async
    public void doAsyncWork() { }
}
```

**Correct approach — use an application event:**

```java
@Component
@RequiredArgsConstructor
public class Warmup {
    private final StatusCacheLoader loader;                     // ← injected PROXY

    @EventListener(ApplicationReadyEvent.class)                 // fires after context is fully up
    void onReady() {
        loader.load();          // ✅ goes through the proxy → @Transactional works
    }
}
```

**Rule of thumb, worth memorizing:** `@PostConstruct` for *self-contained* setup that touches no proxied behaviour. `ApplicationReadyEvent` (or `ApplicationRunner`) for anything involving transactions, async, caching, or other beans' proxied methods.

### Destruction and graceful shutdown

```java
@PreDestroy
void drain() {
    log.info("draining in-flight checks");
    executor.shutdown();
    executor.awaitTermination(20, TimeUnit.SECONDS);
}
```

**`@PreDestroy` only runs on a clean shutdown** — `SIGTERM`, `ctx.close()`, JVM shutdown hook. `SIGKILL` / `kill -9` / OOM-killer / a crashed container skips it entirely. **Therefore: never rely on `@PreDestroy` for data integrity.** Anything that must survive a hard kill needs to be durable *before* shutdown (a database row, a message broker ack). This is a genuinely important production principle and a great interview point.

**Graceful shutdown in Spring Boot — configure this in every real service:**

```properties
server.shutdown=graceful
spring.lifecycle.timeout-per-shutdown-phase=30s
```

Now on SIGTERM: stop accepting new connections → let in-flight requests finish (up to the timeout) → then destroy beans. Combined with Kubernetes' `preStop` hook and readiness probes, this gives you **zero-downtime deploys**. Being able to explain the interaction between SIGTERM, readiness probes, and graceful shutdown is a senior-level deployment answer (Lesson 40).

### Application events — the lifecycle hooks that matter

| Event | When | Use for |
|---|---|---|
| `ApplicationStartingEvent` | Before anything | Very early logging |
| `ApplicationEnvironmentPreparedEvent` | Environment ready, context not created | Inspect/modify property sources |
| `ApplicationContextInitializedEvent` | Context created, definitions not loaded | Rare |
| `ApplicationPreparedEvent` | Definitions loaded, beans not created | Register extra definitions |
| `ContextRefreshedEvent` | All beans created and initialized | Cross-bean setup |
| **`ApplicationReadyEvent`** | **Everything up, server accepting traffic** | **✅ Your default startup hook** |
| `ApplicationFailedEvent` | Startup failed | Alerting on boot failure |
| `ContextClosedEvent` | Shutdown beginning | Coordinated shutdown |

```java
@Component
class ReadyLogger {
    @EventListener
    void onReady(ApplicationReadyEvent e) {
        log.info("Vigil started in {}ms", e.getTimeTaken().toMillis());
    }
}
```

`ApplicationRunner` / `CommandLineRunner` beans run just before `ApplicationReadyEvent` — use them for one-off startup tasks like seeding demo data (Vigil will).

---

## Part 4 — Conditional beans

**This is the mechanism auto-configuration is built from.** Learn it here and Lesson 11 will feel like a formality.

A `@Conditional` asks a yes/no question during the **definition phase**. If the answer is no, the bean definition is never registered — as if the code didn't exist.

### The conditions you'll actually use

```java
@Configuration
public class NotificationConfig {

    // 1. Is a class on the classpath?  ← the single most important condition
    @Bean
    @ConditionalOnClass(name = "com.slack.api.Slack")
    SlackChannel slackChannel() { ... }

    // 2. Has the user NOT already defined one?  ← "sensible default, easily overridden"
    @Bean
    @ConditionalOnMissingBean
    NotificationChannel defaultChannel() { return new LoggingChannel(); }

    // 3. Is a property set?
    @Bean
    @ConditionalOnProperty(name = "vigil.notifications.slack.enabled", havingValue = "true")
    SlackChannel slack() { ... }

    // 4. Is a property set, defaulting to ON if absent?
    @Bean
    @ConditionalOnProperty(name = "vigil.metrics.enabled", matchIfMissing = true)
    MetricsCollector metrics() { ... }

    // 5. Only in a web app?
    @Bean
    @ConditionalOnWebApplication(type = Type.SERVLET)
    RequestLoggingFilter filter() { ... }
}
```

Full set: `@ConditionalOnClass`, `@ConditionalOnMissingClass`, `@ConditionalOnBean`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty`, `@ConditionalOnResource`, `@ConditionalOnWebApplication`, `@ConditionalOnNotWebApplication`, `@ConditionalOnExpression` (SpEL), `@ConditionalOnJava`, `@ConditionalOnCloudPlatform`, `@ConditionalOnThreading` (virtual vs platform threads), `@ConditionalOnAvailableEndpoint`.

### The `@ConditionalOnClass` trick — how Spring references classes it might not have

```java
@Configuration
@ConditionalOnClass(DataSource.class)              // ← referencing a type that may be absent?
public class DataSourceAutoConfiguration { }
```

How does that not throw `NoClassDefFoundError` when the class is missing? Because **Spring reads the annotation from the bytecode using ASM, without loading the class.** It sees the string `"javax.sql.DataSource"` in the constant pool, checks whether the classloader can find it, and only *then* decides to load the configuration class.

This is the actual answer to "how does auto-configuration work without blowing up?", and very few candidates know it. Understanding it required Lesson 04's reflection/bytecode material — this is why that lesson came first.

### `@ConditionalOnMissingBean` — and why ordering is everything

```java
// In a Spring Boot auto-configuration:
@Bean
@ConditionalOnMissingBean
public ObjectMapper objectMapper() { return new ObjectMapper(); }
```

**"Give the user a default unless they defined their own."** This is *the* pattern behind "Spring Boot configures everything but I can always override it."

It only works because auto-configuration is evaluated **last** (Lesson 06 §5). By then, all your `@Configuration` and `@Component` beans are registered, so "does one already exist?" is answerable.

**Which produces the #1 conditional-bean gotcha:**

```java
// ❌ In YOUR OWN configuration — order between your own @Configuration classes is not guaranteed
@Configuration
class MyConfigA {
    @Bean @ConditionalOnMissingBean NotificationChannel a() { return new SlackChannel(); }
}
@Configuration
class MyConfigB {
    @Bean NotificationChannel b() { return new EmailChannel(); }
}
// Whether `a` is created depends on which config is processed first. Non-deterministic. 💀
```

**Rule: use `@ConditionalOnMissingBean` only in auto-configuration classes**, where the ordering guarantee exists. In your own app config, be explicit. Spring's docs say exactly this, and stating it marks you as someone who's read them.

### Writing a custom condition

```java
public class OnKubernetesCondition implements Condition {
    @Override
    public boolean matches(ConditionContext ctx, AnnotatedTypeMetadata md) {
        return ctx.getEnvironment().containsProperty("KUBERNETES_SERVICE_HOST");
    }
}
```
```java
@Bean
@Conditional(OnKubernetesCondition.class)
PodMetadataCollector podMetadata() { ... }
```

Extend `SpringBootCondition` instead of implementing `Condition` when you want your condition's outcome to appear in the `/actuator/conditions` report with a readable message — a small touch that makes a custom starter feel professional.

### Profiles — conditions with better ergonomics

```java
@Bean
@Profile("dev")
DataSource devDataSource() { return new EmbeddedDatabaseBuilder()... }

@Bean
@Profile("!dev")                     // negation
DataSource prodDataSource() { ... }

@Bean
@Profile({"dev", "test"})            // OR
NotificationChannel noopChannel() { return n -> {}; }
```

`@Profile` is literally implemented as a `@Conditional(ProfileCondition.class)`. Profiles are covered fully in Lesson 12; the point here is that they're the same mechanism.

**Profile anti-pattern to be aware of:** `@Profile("prod")` scattered across business logic means your production code path is the one you never test. Prefer **property-driven** conditions (`@ConditionalOnProperty`) so every environment runs the same code with different values, and reserve profiles for genuinely environment-shaped differences (an embedded DB vs a real one). This is a strong opinion but a widely shared one, and articulating it demonstrates real experience.

---

## Part 5 — Build & break

Work in `com.vigil.playground.lifecycle`.

**1. Prove the singleton.**
```java
@Service
public class CounterService {
    private final UUID instanceId = UUID.randomUUID();
    public UUID id() { return instanceId; }
}
```
Inject it into two different services. Print both `id()`s. Identical → one instance. Now add `@Scope("prototype")` and print again — **still identical**, because each singleton got its one injection. **You've just reproduced the prototype trap yourself.** Then fix it with `ObjectProvider` and watch the IDs diverge.

**2. Map the lifecycle empirically.** One bean implementing everything, with print statements:
```java
@Component
public class LifecycleDemo implements InitializingBean, DisposableBean, BeanNameAware {
    public LifecycleDemo() { System.out.println("1 constructor"); }
    public void setBeanName(String n) { System.out.println("2 setBeanName " + n); }
    @PostConstruct void post() { System.out.println("3 @PostConstruct"); }
    public void afterPropertiesSet() { System.out.println("4 afterPropertiesSet"); }
    @PreDestroy void pre() { System.out.println("5 @PreDestroy"); }
    public void destroy() { System.out.println("6 destroy"); }
}
```
Run, then stop the app **gracefully** (IntelliJ's red stop button sends SIGTERM). Observe the order. Then kill the process hard (Task Manager / `taskkill /F`) and observe that 5 and 6 **never print**. That's the lesson about not trusting `@PreDestroy` for correctness, learned by experiment.

**3. Prove the `@PostConstruct` proxy trap.**
```java
@Service
public class TrapDemo {
    @PostConstruct
    void init() {
        System.out.println("in @PostConstruct, this = " + this.getClass().getName());
    }
    @EventListener(ApplicationReadyEvent.class)
    void ready() {
        System.out.println("in ready, bean class = " + this.getClass().getName());
    }
    @Transactional
    public void tx() { }
}
```
Because the class has `@Transactional`, the bean *is* proxied — but inside `@PostConstruct`, `this` prints the **raw class name**, while the bean in the container is `TrapDemo$$SpringCGLIB$$0`. Print `ctx.getBean(TrapDemo.class).getClass().getName()` from a runner to compare. Seeing those two different names side by side is the moment the whole thing clicks.

**4. Request scope and its limits.**
```java
@Component @RequestScope
public class RequestContext {
    private final String id = UUID.randomUUID().toString().substring(0, 8);
    public String id() { return id; }
}
```
Inject into a `@RestController`, return `ctx.id()`. Hit it repeatedly — different every time. Now:
- Print `ctx.getClass().getName()` → it's a **proxy**.
- Remove `@RequestScope`, use `@Scope("request")` without `proxyMode` → read the exception, which literally tells you to add a scoped proxy.
- Call it from an `@Async` method → watch it fail. **That's the ThreadLocal boundary, felt directly.**

**5. Conditional beans.**
```java
@Configuration
public class ConditionalDemo {
    @Bean @ConditionalOnProperty(name = "vigil.demo.enabled", havingValue = "true")
    String demoBean() { return "I exist"; }

    @Bean @ConditionalOnClass(name = "com.example.NotReal")
    String neverExists() { return "impossible"; }
}
```
Start without the property → check `/actuator/beans` (absent) and `/actuator/conditions` (see the *negative* match with its reason). Add `vigil.demo.enabled=true` → it appears. **Reading `/actuator/conditions` is the skill; this exercise builds it.**

**6. Graceful shutdown.** Add `server.shutdown=graceful`, write an endpoint that `Thread.sleep(10_000)` then returns. Call it, and while it's running, stop the app. Observe that the request **completes** before the JVM exits. Now set `server.shutdown=immediate` and repeat. **That difference is what zero-downtime deployment is made of.**

### Explain out loud (60s)

1. Spring singleton vs the Singleton design pattern — the precise difference.
2. What happens when a prototype is injected into a singleton, and three ways to get a fresh one.
3. The scoped proxy: what it does on every method call, and where it finds the instance.
4. Three Spring features that break across `@Async` for the same underlying reason — and the reason.
5. How `@ConditionalOnMissingBean` produces "opinionated defaults you can override."

---

## Part 6 — Production rules

| Do / Never | Why |
|---|---|
| **Singleton beans hold no mutable instance state** | One instance serves every request thread, so a field is shared state — the resulting corruption only appears under production load |
| **Never inject a prototype into a singleton and expect a fresh instance** | The singleton is constructed once, so the injection happens once; use `ObjectProvider.getObject()`, `@Lookup` — or just `new` |
| **Never expect Spring to destroy a prototype** | It hands the object over and forgets it, so `@PreDestroy` never runs; whatever holds a resource must be closed by the caller |
| **Use `@RequestScope`, never a bare `@Scope("request")`** | The shorthand implies the scoped proxy; without it, injection into a singleton fails with "Scope 'request' is not active for the current thread" |
| **Never read request scope, `SecurityContextHolder` or MDC from an `@Async` or `@Scheduled` thread** | All three are `ThreadLocal`-backed, so they are simply absent on another thread — propagate explicitly with a `TaskDecorator` or pass the value |
| **Initialize in the constructor; use `@PostConstruct` only for work that needs the injected object** | The constructor gives you validated, `final` dependencies and runs before anything else can see a half-built bean |
| **Never put `@Transactional`, `@Async` or `@Cacheable` work in `@PostConstruct`** | Proxies are applied after initialization, so `this` is the raw target — use `ApplicationReadyEvent` and call a *different* bean |
| **Never rely on `@PreDestroy` for data integrity** | `kill -9`, an OOM kill or a crashed container skips it entirely; anything you can't lose must be durable before shutdown begins |
| **Set `server.shutdown=graceful` and a shutdown timeout in every real service** | It lets in-flight requests finish on SIGTERM instead of being severed, which is half of what makes a rolling deploy zero-downtime |
| **Use `@ConditionalOnMissingBean` only inside auto-configuration** | It depends on auto-config being evaluated last; between your own `@Configuration` classes there is no ordering guarantee, so the outcome is non-deterministic |
| **Prefer `@ConditionalOnProperty` over `@Profile` for features** | Profiles scattered through business logic mean the production path is the one you never exercise; properties keep one code path with different values |
| **Never enable `spring.main.lazy-initialization` in production** | It moves wiring errors from startup to the first request that touches them, and makes that request pay the initialization latency |

---

## Part 7 — Interview traps

**Q1. Default bean scope, and what "singleton" means precisely?**
`singleton` — one instance **per ApplicationContext**, not per JVM (and not the GoF Singleton pattern). Multiple contexts (common in tests) mean multiple instances.

**Q2. Why can't you put mutable state in a singleton bean?**
It's shared by every request thread (Tomcat: 200 by default), so instance fields become shared mutable state → race conditions and cross-request data leakage. Bugs appear only under production load and are near-impossible to reproduce.

**Q3. What happens when you inject a prototype into a singleton?**
It's resolved **once**, when the singleton is created — you get one instance forever. Fixes: `ObjectProvider.getObject()`, `@Lookup`, or a scoped proxy. Often the right answer is just `new`.

**Q4. Does Spring destroy prototype beans?**
No. Spring creates and configures them, then hands them over — `@PreDestroy` is never called. The caller owns cleanup, which is a strong reason to prefer `new` for short-lived objects that hold resources.

**Q5. How can a request-scoped bean be injected into a singleton?**
Through a **scoped proxy** (`proxyMode = TARGET_CLASS`, implied by `@RequestScope`). The proxy resolves the current thread's instance per method call via `RequestContextHolder`'s `ThreadLocal`. Without the proxy you get "Scope 'request' is not active for the current thread."

**Q6. Why do request scope, `SecurityContextHolder`, and MDC all break in `@Async`?**
All three are `ThreadLocal`-based, and `@Async` runs on a different thread. Fixes: `DelegatingSecurityContextAsyncTaskExecutor` for security, a `TaskDecorator` for MDC/context copying, or passing the context explicitly as a parameter.

**Q7. Order of `@PostConstruct`, `afterPropertiesSet`, and `initMethod`?**
`@PostConstruct` → `InitializingBean.afterPropertiesSet()` → `@Bean(initMethod)`. Prefer the constructor; then `@PostConstruct`; use `initMethod` for third-party classes.

**Q8. Why doesn't `@Transactional` work in `@PostConstruct`?**
Proxies are created in `BeanPostProcessor.postProcessAfterInitialization`, **after** `@PostConstruct` runs. Inside `@PostConstruct` you hold the raw target, so no advice applies. Use `ApplicationReadyEvent` or an `ApplicationRunner` and call a *different* bean.

**Q9. When does `@PreDestroy` not run?**
On `SIGKILL`/`kill -9`, an OOM kill, a JVM crash, a hard container stop, or `System.exit` in some configurations. So never depend on it for data integrity — durability must be achieved before shutdown.

**Q10. What is graceful shutdown and why does it matter?**
`server.shutdown=graceful` makes Boot stop accepting new connections on SIGTERM and let in-flight requests finish (within `spring.lifecycle.timeout-per-shutdown-phase`) before destroying beans. Combined with readiness probes and a `preStop` delay, it enables zero-downtime rolling deploys.

**Q11. How does `@ConditionalOnClass` avoid `NoClassDefFoundError`?**
Spring reads the annotation's metadata from the class file with **ASM**, without loading the referenced class, and asks the classloader whether the name is resolvable. Only if it is does it load and process the configuration class.

**Q12. Why does `@ConditionalOnMissingBean` work in auto-configuration but not reliably in your own config?**
Auto-configuration is processed **last**, so all user beans are already registered. Between your own `@Configuration` classes there's no guaranteed processing order, so the condition's outcome becomes non-deterministic.

**Q13. Should you use `@Profile` or `@ConditionalOnProperty` for feature toggles?**
`@ConditionalOnProperty`. Profiles scattered through code mean production runs a code path that's never exercised elsewhere; property-driven config keeps one code path with different values. Reserve profiles for genuinely environment-shaped differences.

**Q14. What's the downside of `spring.main.lazy-initialization=true`?**
Wiring errors move from startup to first request, the first request pays initialization latency, and `@Scheduled`/listener beans may not exist when expected. Fine for local dev, risky as a production default.

---

## Part 8 — Retention check

1. What's the difference between a Spring singleton and the Singleton design pattern?
2. Draw the request-scoped-bean-in-a-singleton diagram and explain what the proxy does per call.
3. Name three Spring features that break across `@Async` for the same underlying reason, and name the reason.
4. Why doesn't `@Transactional` work in `@PostConstruct`? Reference the exact lifecycle step.
5. When can you rely on `@PreDestroy`, and what must you do for things you can't lose?
6. Explain how `@ConditionalOnMissingBean` gives you "opinionated defaults you can override."

---

## Module 2 complete ✅

You now understand the container: what beans are, how they're defined, how they're injected, how many exist, when they live and die, and how to make them conditional. **That is the core of Spring.** Everything from here is application of it.

**Next:** [Lesson 09 — AOP concepts & how proxies really work](../03-aop/09-aop-and-proxies.md) — the other half of the magic, and the source of the most misunderstood bug in Spring.

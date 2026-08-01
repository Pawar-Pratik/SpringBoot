# Lesson 10 — AOP in practice

> **Why this lesson exists:** Knowing what AOP is gets you through an interview question. Knowing *when to use it and when not to* gets you respected in a code review. Custom aspects are powerful and dangerous in equal measure: they add invisible behaviour, which is exactly what makes a codebase hard to reason about. This lesson teaches the syntax, the three aspects genuinely worth writing, and the discipline to stop there.

**Time:** ~55 minutes · **Prereq:** Lesson 09

---

## 1. Setup

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

That brings in `aspectjweaver` for the **pointcut expression parser** (Spring borrows AspectJ's language, not its weaver). `@EnableAspectJAutoProxy` is applied automatically by Boot's `AopAutoConfiguration` — you don't add it.

```java
@Aspect          // marks it as an aspect — but this alone does NOTHING
@Component       // ← REQUIRED. Without it, it's not a bean and is never applied.
public class TimingAspect { }
```

**Forgetting `@Component` on an `@Aspect` is the #1 "my aspect isn't running" cause.** `@Aspect` is not a stereotype annotation.

---

## 2. Pointcut expressions

### `execution()` — the workhorse

```
execution( modifiers? return-type declaring-type? method-name(params) throws? )
                          ^ optional      ^ required
```

```java
// every public method in any class under com.vigil.monitor (.. = any depth)
execution(public * com.vigil.monitor..*.*(..))

// every method of any class whose name ends in Service, anywhere under com.vigil
execution(* com.vigil..*Service.*(..))

// specific method, any args
execution(* com.vigil.monitor.MonitorService.createMonitor(..))

// returns void, exactly one UUID parameter
execution(void com.vigil..*.delete*(java.util.UUID))

// first param is UUID, then anything
execution(* com.vigil..*.*(java.util.UUID, ..))
```

Wildcards: `*` = one element (any return type, any name, one package segment). `..` = any number (any depth of packages, any argument list).

### `@annotation()` and `@within()` — the ones you'll actually use

**Prefer annotation-driven pointcuts over package/name patterns.** Package patterns break silently when someone renames a package; an annotation is explicit at every call site and greppable.

```java
// method IS annotated
@annotation(com.vigil.audit.Audited)

// the method's DECLARING CLASS is annotated
@within(com.vigil.audit.Audited)

// the method's declaring class is annotated — checked at runtime (rarely needed; @within is cheaper)
@target(com.vigil.audit.Audited)

// an ARGUMENT's type is annotated
@args(com.vigil.validation.Sanitized)
```

Others you'll see:
```java
within(com.vigil.monitor..*)                 // type-based: any join point inside this package
bean(monitorService)                          // Spring-specific! by bean name; supports bean(*Service)
args(java.util.UUID, ..)                      // argument types
this(com.vigil.Marker)                        // PROXY implements this type
target(com.vigil.Marker)                      // TARGET object implements this type
```

**`this()` vs `target()`** is a classic trick question: `this` refers to the proxy, `target` to the wrapped object. With CGLIB they're usually the same type; with a JDK proxy the proxy implements only the interfaces, so `this(MonitorService)` wouldn't match while `target(MonitorService)` would.

### Combining and naming pointcuts

```java
@Aspect
@Component
public class PointcutLibrary {

    @Pointcut("within(com.vigil..*)")
    public void inVigil() {}                                    // method body is always empty

    @Pointcut("@annotation(com.vigil.audit.Audited)")
    public void isAudited() {}

    @Pointcut("execution(* com.vigil..*Repository.*(..))")
    public void repositoryLayer() {}

    @Pointcut("inVigil() && isAudited() && !repositoryLayer()")
    public void auditableBusinessMethod() {}
}
```

```java
@Around("com.vigil.aop.PointcutLibrary.auditableBusinessMethod()")   // fully qualified reference
public Object audit(ProceedingJoinPoint pjp) throws Throwable { ... }
```

**Named pointcuts in a shared class is the professional pattern.** Reusable, documented, testable in one place, and it stops the same fragile string from being copy-pasted into eight aspects.

### Binding the annotation and arguments

```java
@Around("@annotation(audited)")                     // parameter name binds the annotation instance
public Object audit(ProceedingJoinPoint pjp, Audited audited) throws Throwable {
    String action = audited.action();               // ✅ read the annotation's attributes
    ...
}
```

```java
@Around("@annotation(com.vigil.audit.Audited) && args(monitorId, ..)")
public Object audit(ProceedingJoinPoint pjp, UUID monitorId) throws Throwable { ... }
```

The parameter name in the expression must match the method parameter name — a string-to-name coupling, which is why `-parameters` compilation matters and why refactoring an aspect's parameter names can break it.

### What you can read from a `JoinPoint`

```java
@Around("...")
public Object advice(ProceedingJoinPoint pjp) throws Throwable {
    MethodSignature sig = (MethodSignature) pjp.getSignature();
    Method method       = sig.getMethod();              // the interface/proxy method
    String  className   = sig.getDeclaringTypeName();
    String  methodName  = sig.getName();
    Object[] args       = pjp.getArgs();
    Object   target     = pjp.getTarget();              // the real object
    Class<?> returnType = sig.getReturnType();

    // To read annotations reliably (JDK-proxy safe), resolve the TARGET's method:
    Method targetMethod = pjp.getTarget().getClass()
            .getMethod(sig.getName(), sig.getParameterTypes());
    Audited ann = targetMethod.getAnnotation(Audited.class);

    return pjp.proceed();
}
```

**Use `AnnotationUtils.findAnnotation(...)` / `AnnotatedElementUtils.findMergedAnnotation(...)`** rather than raw `getAnnotation` — they walk meta-annotations, interfaces, and superclasses, which raw reflection doesn't. This is a small detail that separates a working aspect from a correct one.

---

## 3. The three aspects genuinely worth writing

I'm giving you three, not fifteen, deliberately. These cover ~90% of legitimate custom-aspect use in real systems. Everything else is usually better solved by a filter, an interceptor, an event, or just a method call.

### Aspect 1 — Method timing / metrics

```java
@Aspect
@Component
@RequiredArgsConstructor
@Slf4j
public class TimingAspect {

    private final MeterRegistry meterRegistry;

    @Around("@annotation(com.vigil.common.Timed)")
    public Object time(ProceedingJoinPoint pjp) throws Throwable {
        var sig = (MethodSignature) pjp.getSignature();
        String metric = sig.getDeclaringType().getSimpleName() + "." + sig.getName();

        long start = System.nanoTime();
        String outcome = "success";
        try {
            return pjp.proceed();
        } catch (Throwable t) {
            outcome = t.getClass().getSimpleName();
            throw t;                                    // ⚠️ ALWAYS rethrow
        } finally {
            long ms = (System.nanoTime() - start) / 1_000_000;
            meterRegistry.timer("method.duration", "method", metric, "outcome", outcome)
                         .record(ms, TimeUnit.MILLISECONDS);
            if (ms > 1000) log.warn("SLOW: {} took {}ms", metric, ms);
        }
    }
}
```

**Note the `outcome` tag.** Timing that doesn't distinguish success from failure is nearly useless — failures are often fast (a connection refused returns instantly) and will drag your average down while hiding a real problem. That detail is the difference between a metric an SRE trusts and one they ignore.

> **Reality check:** Micrometer already provides `@Timed` and `@Observed` with more features (percentiles, SLO buckets, tracing integration). **Use theirs.** This exercise teaches you the mechanism so you understand what `@Observed` is doing — Lesson 35 uses the real thing.

### Aspect 2 — Audit logging (the strongest genuine use case)

Audit trails are a real compliance requirement — SOC 2, ISO 27001, GDPR, HIPAA all want "who did what, when." An aspect is the correct tool because the requirement is *literally* cross-cutting, and because a forgotten audit call is an audit *finding*, not just a bug.

```java
@Target(METHOD) @Retention(RUNTIME)
public @interface Audited {
    String action();
    boolean includeArgs() default false;
}
```

```java
@Aspect
@Component
@RequiredArgsConstructor
public class AuditAspect {

    private final AuditEventPublisher publisher;      // async/outbox write — NOT a sync DB insert

    @Around("@annotation(audited)")
    public Object audit(ProceedingJoinPoint pjp, Audited audited) throws Throwable {
        String actor = SecurityContextHolder.getContext().getAuthentication() != null
                ? SecurityContextHolder.getContext().getAuthentication().getName()
                : "system";
        String correlationId = MDC.get("correlationId");

        try {
            Object result = pjp.proceed();
            publisher.publish(AuditEvent.success(actor, audited.action(), correlationId,
                    audited.includeArgs() ? redact(pjp.getArgs()) : null));
            return result;
        } catch (Throwable t) {
            publisher.publish(AuditEvent.failure(actor, audited.action(), correlationId,
                    t.getClass().getSimpleName()));
            throw t;
        }
    }

    private Object[] redact(Object[] args) {
        // NEVER log raw args — passwords, tokens, PII, card numbers all travel as method args.
        return Arrays.stream(args).map(this::redactOne).toArray();
    }
}
```

**Three production details in there that matter more than the AOP:**
1. **Audit both success and failure.** "Attempted to delete monitor and was denied" is often the more interesting event.
2. **Redact arguments.** Blindly logging `pjp.getArgs()` is how passwords end up in log aggregators. This is a real incident pattern.
3. **Publish, don't insert.** A synchronous DB insert in an audit aspect doubles your write latency and couples business availability to the audit table. Publish an event; persist asynchronously or via an outbox (Lesson 32).

**Vigil uses this aspect for real** — every monitor/incident mutation is audited.

### Aspect 3 — Distributed lock / idempotency guard

```java
@Target(METHOD) @Retention(RUNTIME)
public @interface DistributedLock {
    String key();                      // SpEL, e.g. "#monitorId"
    long timeoutMs() default 5000;
}
```

```java
@Aspect
@Component
@RequiredArgsConstructor
@Order(Ordered.HIGHEST_PRECEDENCE + 10)     // acquire the lock BEFORE the tx starts
public class DistributedLockAspect {

    private final StringRedisTemplate redis;
    private final ExpressionParser parser = new SpelExpressionParser();

    @Around("@annotation(lockAnn)")
    public Object lock(ProceedingJoinPoint pjp, DistributedLock lockAnn) throws Throwable {
        String key = "lock:" + resolveSpel(pjp, lockAnn.key());
        String token = UUID.randomUUID().toString();

        Boolean acquired = redis.opsForValue()
                .setIfAbsent(key, token, Duration.ofMillis(lockAnn.timeoutMs()));

        if (!Boolean.TRUE.equals(acquired)) {
            throw new ConcurrentOperationException("locked: " + key);
        }
        try {
            return pjp.proceed();
        } finally {
            // Release only if WE still hold it — otherwise you can release someone else's lock
            // after your TTL expired. In production, do this with a Lua script for atomicity.
            if (token.equals(redis.opsForValue().get(key))) redis.delete(key);
        }
    }
}
```

**The `@Order` line is the whole lesson.** If the lock aspect runs *inside* the transaction, you release the lock before the transaction commits — so another node can acquire the lock and read pre-commit state. Ordering aspects relative to `@Transactional` is subtle, load-bearing, and a genuinely senior-level concern.

Also note the token check on release: releasing a lock you no longer own is a classic distributed-locking bug (the "Redlock" debate is worth a read someday). And for production, **use Redisson or ShedLock** rather than hand-rolling — they've handled the edge cases. But understanding *why* those libraries exist requires having seen this code.

### Aspect ordering

```java
@Aspect @Component @Order(1) class SecurityAspect { }    // outermost
@Aspect @Component @Order(2) class TimingAspect { }
@Aspect @Component @Order(3) class AuditAspect { }       // innermost
```

Lower `@Order` = outer = runs first on the way in, last on the way out. **`@Transactional`'s advice sits at `Ordered.LOWEST_PRECEDENCE` by default**, which means it's the innermost of your aspects unless you change it. Consequences:

- An aspect at default order runs **outside** the transaction → its `finally` executes **after** commit. Good for "publish only if committed"; bad if it needs to read uncommitted state.
- To run **inside** the transaction, order it *after* the transaction advisor (higher number), or better, use `@TransactionalEventListener(phase = AFTER_COMMIT)` which expresses the intent directly.

**"When exactly does my aspect run relative to commit?" is a question you must be able to answer for any aspect you write.** If you can't, you don't know what your code does.

---

## 4. When NOT to use AOP

This section matters more than the syntax. **Reviewers are far more impressed by restraint than by cleverness.**

### ❌ Don't use AOP for business logic

```java
// ❌ Genuinely terrible: business rules hidden in an aspect
@Around("execution(* com.vigil..MonitorService.create*(..))")
public Object applyQuota(ProceedingJoinPoint pjp) throws Throwable {
    if (monitorCount() >= plan.limit()) throw new QuotaExceededException();
    return pjp.proceed();
}
```

A developer reading `MonitorService.create()` sees no quota check. It isn't in the code, the tests, or the call graph. They'll spend an hour finding it. **Business rules belong in the code that owns them, where they're visible, testable, and debuggable.**

The test: *"if a new team member read only this method, would they be surprised by what happens?"* If yes, don't hide it in an aspect.

### ❌ Don't reimplement what Spring already gives you

| You want | Don't write an aspect — use |
|---|---|
| Transactions | `@Transactional` |
| Caching | `@Cacheable` / `@CacheEvict` |
| Retries | Spring Retry `@Retryable` or Resilience4j |
| Async | `@Async` |
| Method security | `@PreAuthorize` / `@PostAuthorize` |
| Metrics | Micrometer `@Timed` / `@Observed` |
| Argument validation | `@Validated` + Bean Validation |
| HTTP request logging | A `Filter` or `HandlerInterceptor` |
| Rate limiting | Gateway / filter / Bucket4j |

### ❌ Don't use AOP for HTTP-level concerns

Correlation IDs, request logging, CORS, and auth all belong in the **filter chain**, which sees the raw request/response, runs for *every* request (including 404s and errors that never reach a controller), and can short-circuit. An aspect on controllers misses all of that. Lesson 17 covers picking the right layer — the decision matrix there is a common interview question.

### ⚠️ Know the real costs

1. **Invisibility.** `Ctrl+click` doesn't navigate into an aspect. Behaviour exists that isn't in the call graph.
2. **Debugging pain.** Stack traces gain proxy frames; breakpoints in your method don't show the aspect.
3. **Ordering bugs.** Multi-aspect interactions (especially with transactions) are subtle and rarely tested.
4. **Silent non-application.** A typo'd pointcut matches nothing and produces **no error**. Your security aspect just... doesn't run. **This is why every aspect needs a test that proves it applies.**
5. **Performance.** Small per-call cost, but proxying beans on a hot path in a tight loop is measurable.

### ✅ The legitimate list, complete

Cross-cutting, non-business, infrastructural, and identical everywhere:
- audit logging
- timing/metrics (when the built-ins don't fit)
- distributed locking / idempotency guards
- multi-tenant context enforcement (e.g. asserting a tenant filter is set)
- legacy integration shims you can't modify the callers of
- feature-flag gates around whole methods (arguably)

That's roughly the whole list. **If your aspect isn't on it, argue for it carefully.**

---

## 5. Testing aspects

**Untested aspects are worse than no aspect**, because a silently non-matching pointcut gives you false confidence in a security or audit control.

```java
@SpringBootTest
class AuditAspectTest {

    @Autowired MonitorService service;         // must be the PROXY, from the context
    @MockitoBean AuditEventPublisher publisher;

    @Test
    void auditsSuccessfulCreate() {
        service.createMonitor(new CreateMonitorRequest("API", "https://x", 60));

        var captor = ArgumentCaptor.forClass(AuditEvent.class);
        verify(publisher).publish(captor.capture());
        assertThat(captor.getValue().action()).isEqualTo("MONITOR_CREATED");
    }

    @Test
    void auditsFailure() {
        assertThatThrownBy(() -> service.createMonitor(invalidRequest()))
                .isInstanceOf(ValidationException.class);
        verify(publisher).publish(argThat(e -> e.outcome() == FAILURE));
    }

    @Test
    void aspectIsActuallyApplied() {                       // ← the guard test. Write this one.
        assertThat(AopUtils.isAopProxy(service)).isTrue();
    }
}
```

Notes: this **must** be an integration test (`@SpringBootTest`), because a plain `new MonitorService(...)` has no proxy and the aspect won't run. And `@MockitoBean` is the Boot 3.4+ replacement for the deprecated `@MockBean`.

Bonus: assert the pointcut matches what you think it does:
```java
var pc = new AspectJExpressionPointcut();
pc.setExpression("@annotation(com.vigil.audit.Audited)");
assertThat(pc.matches(MonitorService.class.getMethod("createMonitor", CreateMonitorRequest.class),
                      MonitorService.class)).isTrue();
```
That's a genuinely nice technique very few people know.

---

## 6. Build this

Work in `com.vigil.playground.aop2`.

**1. `@Timed` aspect.** Custom annotation + `@Around` recording duration and a success/failure outcome, logging a warning over 500ms. Apply to a service method with a random sleep. Verify both paths appear correctly.

**2. `@Audited` aspect with annotation binding.** Read `action()` off the annotation. Log actor (hardcode `"system"` for now), action, outcome, and duration. Handle the exception path and **rethrow**. Then deliberately *forget* the rethrow and observe how badly things break — the caller thinks the operation succeeded. **That's why `throw t;` is non-negotiable in `@Around`.**

**3. A pointcut library class.** Named pointcuts: `inVigil()`, `serviceLayer()`, `repositoryLayer()`, `isAudited()`. Combine them. Reference them from both aspects by fully-qualified name.

**4. Prove ordering, including relative to transactions.** Three aspects printing `>> name` on entry and `<< name` on exit, with `@Order(1/2/3)`. Predict the output, then run. Now add `@Transactional` to the target method with transaction TRACE logging on and determine **where the commit happens relative to each aspect's exit line.** Write down the answer. This is the exercise that turns AOP from a party trick into an engineering tool.

**5. Break it on purpose:**
- Remove `@Component` from an aspect → **nothing happens, no error**. Sit with that.
- Typo the pointcut (`@anotation`) → nothing happens, no error.
- Point an aspect at a `private` method → nothing happens.
- Have `@Around` return `null` instead of `pjp.proceed()`'s result → watch the caller NPE mysteriously.

**Every one of these fails silently.** That's the argument for the `aspectIsActuallyApplied()` guard test.

**6. Write the tests** from §5, including the guard test and the pointcut-matching assertion.

---

## 7. Interview traps

**Q1. How do you create an aspect in Spring Boot?**
Add `spring-boot-starter-aop`; annotate a class `@Aspect` **and** `@Component`; write advice methods with pointcut expressions. Boot auto-enables `@EnableAspectJAutoProxy`. Forgetting `@Component` is the classic mistake — `@Aspect` isn't a stereotype.

**Q2. Difference between `@annotation`, `@within`, `within`, and `execution`?**
`@annotation` — the *method* carries the annotation. `@within` — the method's *declaring class* carries it. `within` — the join point is inside a given type/package (no annotation involved). `execution` — matches a method signature pattern. Prefer annotation-based pointcuts: explicit at the call site and refactor-safe.

**Q3. Why prefer annotation-driven pointcuts over `execution(* com.vigil..*Service.*(..))`?**
Package/name patterns break silently on refactoring, apply to code the author didn't intend, and are invisible to someone reading the target class. An annotation appears at every affected method and is greppable.

**Q4. How do aspect ordering and `@Transactional` interact?**
`@Order` controls nesting (lower = outer). The transaction advisor is `LOWEST_PRECEDENCE` by default, so custom aspects are usually *outside* it — meaning their `finally` runs **after** commit. If you need to run inside the transaction, order accordingly; if you need to act only after a successful commit, use `@TransactionalEventListener(AFTER_COMMIT)` instead.

**Q5. What must every `@Around` advice do?**
Call `proceed()` (unless deliberately short-circuiting), **return its result**, and **rethrow** any exception. Swallowing an exception or returning `null` silently corrupts caller semantics — and it's a bug that surfaces far from its cause.

**Q6. When would you NOT use AOP?**
For business logic (hides rules from readers), for anything Spring already provides (transactions, caching, retry, security, metrics, validation), and for HTTP-level concerns (use filters/interceptors, which see all requests including errors and can short-circuit).

**Q7. What's the biggest risk of custom aspects?**
Silent non-application: a typo'd pointcut, a missing `@Component`, a `private` target method, or a self-invoked call produces **no error at all**. For a security or audit aspect that's a control that quietly doesn't exist — hence the mandatory test asserting the aspect actually applies.

**Q8. How do you test an aspect?**
Integration test (`@SpringBootTest`) so the proxy exists — a `new`-constructed object has no advice. Mock the aspect's collaborators, exercise success and failure paths, and add a guard assertion (`AopUtils.isAopProxy(bean)`) plus optionally an `AspectJExpressionPointcut` match assertion.

**Q9. Give a real production use case for a custom aspect.**
Audit logging for compliance (SOC 2 / GDPR): genuinely cross-cutting, must not be forgotten, needs actor + action + outcome + correlation ID, must redact sensitive arguments, and should publish an event rather than block on a synchronous insert.

**Q10. `this()` vs `target()` in a pointcut?**
`this` matches on the **proxy**'s type, `target` on the **target object**'s type. Identical under CGLIB; with a JDK proxy the proxy only implements the interfaces, so `this(ConcreteClass)` won't match while `target(ConcreteClass)` will.

---

## 8. Retention check

1. Why must an `@Aspect` also be `@Component`, and what happens if it isn't?
2. Give three things a typo'd pointcut and a forgotten `@Component` have in common. (Hint: it's the reason for one specific test.)
3. Where does your aspect's `finally` block run relative to the transaction commit, by default? How would you change it?
4. Name four things you should *not* write an aspect for, and what to use instead.
5. Why is redacting arguments essential in an audit aspect?
6. What are the three obligations of `@Around` advice?

---

## Module 3 complete ✅

You now understand both of Spring's engines: **the container** (creates and wires objects) and **AOP** (wraps them to add behaviour). Everything from here is Spring using those two engines to solve specific problems — web, data, security, messaging.

**Next:** [Lesson 11 — Auto-configuration demystified](../04-spring-boot/11-autoconfiguration.md) — the feature that made Spring Boot famous, which you now have every tool to understand completely.

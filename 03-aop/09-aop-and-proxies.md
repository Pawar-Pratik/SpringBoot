# Lesson 09 — AOP concepts & how proxies really work

> **Why this lesson exists:** AOP is the second of Spring's two engines (the container is the first). You already met proxies in Lesson 04; now you'll learn the vocabulary interviewers use, the exact proxying rules and their silent failure modes, and the *definitive* treatment of the self-invocation bug. If you master one debugging skill in Spring, make it this one — it explains `@Transactional`, `@Cacheable`, `@Async`, `@Retryable`, `@PreAuthorize` and lazy loading with a single mental model.

**Time:** ~65 minutes · **Prereq:** Lesson 04, Lesson 06

---

## 1. The idea in one sentence

> **A proxy is a wrapper, and a wrapper can only intercept calls that pass through it — so every Spring AOP rule, limitation and silent failure follows from asking one question: did this call arrive from outside the object, or from inside it?**

Spring chose runtime proxies over AspectJ's compiler so that no special build step is needed. That one pragmatic decision produces every constraint in this lesson: method-execution join points only, no `final`/`private`/`static` advice, and self-invocation.

---

## 2. The problem AOP solves

Lesson 05 showed you this:

```java
public void createMonitor(CreateMonitorRequest req) {
    long start = System.nanoTime();
    log.info("createMonitor start user={}", currentUser());
    if (!security.hasRole("ADMIN")) throw new AccessDeniedException("nope");
    Transaction tx = txManager.begin();
    try {
        Monitor m = repo.save(new Monitor(req));       // ← the only line that matters
        tx.commit();
    } catch (RuntimeException e) {
        tx.rollback();
        throw e;
    } finally {
        metrics.timer("createMonitor").record(System.nanoTime() - start);
        log.info("createMonitor end");
    }
}
```

Logging, security, transactions and metrics are **cross-cutting concerns**: they apply across many classes, they're orthogonal to business logic, and OOP has no good tool for them.

- **Inheritance** doesn't work — you can only extend one class, and these concerns cut across unrelated hierarchies.
- **Composition/delegation** helps but still requires the call in every method.
- **Copy-paste** guarantees inconsistency, and an inconsistent security check is a vulnerability.

**AOP's proposition:** describe the concern *once*, describe *where* it applies with a pattern, and let the framework weave it in. Your business method contains only business logic.

> **Fun fact:** AOP came out of Xerox PARC in the late 1990s (Gregor Kiczales et al.), producing **AspectJ** — a genuine extension to the Java language with its own compiler. Spring adopted AspectJ's *vocabulary and pointcut syntax* but implemented it with runtime proxies instead of bytecode weaving, because proxies need no special compiler or agent. **That single pragmatic decision is the source of every Spring AOP limitation you'll ever hit** — including self-invocation. Being able to state that causal link is a genuinely impressive interview moment.

---

## 3. The vocabulary (interviewers use these exact words)

| Term | Definition | Concrete example |
|---|---|---|
| **Aspect** | A module bundling a cross-cutting concern | `TransactionAspect`, `AuditAspect` |
| **Join point** | A point in execution where advice *could* apply | In Spring AOP: **always a method execution** |
| **Advice** | The code that runs at a join point | "begin a transaction before, commit after" |
| **Pointcut** | An expression selecting join points | `@within(org.springframework.transaction.annotation.Transactional)` |
| **Target** | The original, un-advised object | Your real `MonitorService` instance |
| **Proxy** | The wrapper the container hands out | `MonitorService$$SpringCGLIB$$0` |
| **Weaving** | Linking aspects to targets | Spring: at **runtime**, via proxy creation |
| **Introduction** | Adding new interfaces/methods to a type | Rare; AspectJ feature |

**The one to remember: in Spring AOP, a join point is always a *method execution*.** AspectJ additionally supports field access, constructor calls, and static initializers. Spring does not — because you can't intercept a field read with a proxy. Knowing this boundary explains half of Spring AOP's limitations.

### The five advice types

```java
@Aspect
@Component
public class DemoAspect {

    @Before("execution(* com.vigil..*Service.*(..))")
    public void before(JoinPoint jp) { }                     // runs before; can't stop the call

    @AfterReturning(pointcut = "...", returning = "result")
    public void afterReturning(JoinPoint jp, Object result) { }   // only on success

    @AfterThrowing(pointcut = "...", throwing = "ex")
    public void afterThrowing(JoinPoint jp, Throwable ex) { }     // only on exception

    @After("...")
    public void after(JoinPoint jp) { }                      // finally — success OR exception

    @Around("...")
    public Object around(ProceedingJoinPoint pjp) throws Throwable {   // the powerful one
        // before
        Object result = pjp.proceed();     // ← YOU control whether/when the target runs
        // after
        return result;                     // ← you can even change the return value
    }
}
```

**`@Around` is the only one that can:**
- **prevent** the target from executing (don't call `proceed()`) → that's how caching returns a hit
- **modify** arguments (`proceed(newArgs)`)
- **modify** the return value
- **retry** (call `proceed()` in a loop) → that's how `@Retryable` works
- **wrap in try/finally** → that's how `@Transactional` works

Which means **`@Transactional`, `@Cacheable`, `@Retryable`, `@Async` and `@TimeLimiter` are all `@Around` advice.** One concept, five features. That unification is the payoff of this lesson.

**Execution order** when advice types stack: `@Around` (before part) → `@Before` → **target** → `@AfterReturning`/`@AfterThrowing` → `@After` → `@Around` (after part).

---

## 4. How proxies are actually created

Recall Lesson 06 step 10f: proxies are created by a `BeanPostProcessor` after initialization. Specifically, `AnnotationAwareAspectJAutoProxyCreator`:

```
For each newly initialized bean:
  1. Collect all advisors in the context (@Aspect beans + @Transactional/@Async/@Cacheable
     support advisors registered by @EnableTransactionManagement etc.)
  2. Ask each advisor's pointcut: "do you match ANY method of this bean's class?"
  3. If none match  → return the bean unchanged.        ← no proxy, no overhead
  4. If any match   → create a proxy wrapping it, and RETURN THE PROXY.
     The container stores the proxy. Everyone who injects this bean gets the proxy.
```

Two consequences worth internalizing:

- **Proxying is per-bean, not per-method.** One matching method proxies the whole bean; the proxy then decides *per call* whether to apply advice.
- **Nothing you wrote knows about this.** Your class is untouched. That's why you can unit-test it with `new` and get pure business behaviour with no transaction machinery — a real advantage of proxy-based AOP over bytecode weaving.

### JDK dynamic proxies vs CGLIB — the decision table

```
Does the bean implement at least one interface?
├── YES → is proxyTargetClass=true?
│        ├── NO  → JDK dynamic proxy (implements the same interfaces)
│        └── YES → CGLIB subclass                    ← Spring Boot's default
└── NO  → CGLIB subclass (the only option)
```

**Spring Boot sets `proxyTargetClass=true` by default**, so you almost always get CGLIB. That was a deliberate choice, because the JDK-proxy behaviour surprised people:

```java
public interface MonitorOperations { void create(String n); }

@Service
public class MonitorService implements MonitorOperations {
    public void create(String n) { }
    public void extraMethodNotOnInterface() { }      // ← invisible through a JDK proxy!
}
```

With a JDK proxy, the container's object implements `MonitorOperations` but **is not a `MonitorService`**, so:
```java
@Autowired MonitorService svc;     // 💥 with JDK proxying: bean isn't assignable to MonitorService
```
That error (`BeanNotOfRequiredTypeException`) baffled a generation of developers. CGLIB subclassing avoids it because the proxy *is* a `MonitorService`.

| | JDK dynamic proxy | CGLIB |
|---|---|---|
| Mechanism | `java.lang.reflect.Proxy` + `InvocationHandler` | Generated subclass, method overriding |
| Needs an interface | ✅ yes | ❌ no |
| Proxy type is | the interface(s) only | a subclass of your class |
| Can't proxy | classes without interfaces | `final` classes, `final` methods, `private` methods |
| Constructor | not called on target's behalf | **target's constructor may be called** (see below) |
| Boot default | — | ✅ |

> **The CGLIB constructor subtlety:** because CGLIB creates a subclass, the subclass constructor runs — historically requiring a no-arg constructor and causing side effects to run twice. Modern Spring uses Objenesis to instantiate the proxy **without** calling a constructor, which is why constructor-injected beans with no default constructor proxy fine today. If someone asks "doesn't CGLIB need a default constructor?", the answer is "it did; Spring uses Objenesis now."

### The silent failure modes — commit these to memory

```java
@Service
public final class Svc {                  // ❌ CGLIB can't subclass final
    @Transactional public void a() { }
}
// → BeanCreationException at startup: "Cannot subclass final class"  (at least it's loud)
```

```java
@Service
public class Svc {
    @Transactional
    private void a() { }                  // ❌ private → not overridable → SILENTLY IGNORED

    @Transactional
    final void b() { }                    // ❌ final → not overridable → SILENTLY IGNORED

    @Transactional
    static void c() { }                   // ❌ static → not a virtual call → SILENTLY IGNORED
}
```

**No exception. No warning. No log line.** The annotation is simply inert. `private`, `final`, and `static` annotated methods are the three silent killers.

> **Boot 4 / recent Framework note:** newer versions have added startup warnings for some of these cases (e.g. non-public `@Transactional` methods being detected), which helps — but don't rely on it. Know the rule.

---

## 5. THE SELF-INVOCATION PROBLEM — definitive treatment

You met it in Lesson 04. Here it is in full, because it is *the* Spring AOP interview question and *the* Spring AOP production bug.

```java
@Service
public class CheckService {

    public void runAllChecks(List<Monitor> monitors) {
        for (Monitor m : monitors) {
            runCheck(m);                      // ← plain this.runCheck(m)
        }
    }

    @Transactional
    public void runCheck(Monitor m) {
        // no transaction when called from runAllChecks. None. Zero.
    }
}
```

### The trace, precisely

```
Controller holds:        CheckService$$SpringCGLIB  (the PROXY)
                                 │
   proxy.runAllChecks(list)      │  proxy checks: does runAllChecks need advice?
                                 │  → no @Transactional → just delegate
                                 ▼
                         target.runAllChecks(list)     ← now executing INSIDE the target
                                 │
                                 │  `this` == the target, NOT the proxy
                                 ▼
                         this.runCheck(m)  →  target.runCheck(m)
                                              ↑
                                     the proxy is nowhere in this call path.
                                     @Transactional has no opportunity to run.
```

The proxy is a *wrapper*. Once execution is inside the wrapped object, the wrapper is out of the picture. **A proxy can only intercept calls that pass through it.**

### The full list of what this breaks

| Annotation | Symptom when self-invoked |
|---|---|
| `@Transactional` | No transaction; no rollback; each `save` auto-commits individually |
| `@Async` | Runs synchronously on the caller's thread |
| `@Cacheable` | Cache never consulted or populated; every call hits the DB |
| `@CacheEvict` | Cache never cleared → stale data served indefinitely |
| `@Retryable` | No retry on failure |
| `@PreAuthorize` / `@Secured` | **Authorization check skipped** ← this one is a security bug |
| `@Validated` | Method parameter validation skipped |
| `@Observed` / `@Timed` | No metric recorded |
| `@Transactional(propagation = REQUIRES_NEW)` | Silently joins the outer transaction (or has none) |

That `@PreAuthorize` row deserves emphasis: self-invocation can turn a missing annotation effect into an **authorization bypass**. Mention that in an interview and you'll be visibly ahead.

### The fixes, ranked with honest trade-offs

**1. Extract to a separate bean — the right answer almost always**

```java
@Service
@RequiredArgsConstructor
public class CheckOrchestrator {
    private final CheckExecutor executor;       // ← Spring injects the PROXY

    public void runAllChecks(List<Monitor> monitors) {
        monitors.forEach(executor::runCheck);   // ✅ goes through the proxy
    }
}

@Service
public class CheckExecutor {
    @Transactional
    public void runCheck(Monitor m) { ... }
}
```

Why it's genuinely better, not just a workaround: needing self-invocation with different transactional semantics usually means the class holds two responsibilities — *orchestration* (loop, no transaction) and *unit of work* (one transaction per item). Splitting them names that distinction. **And this exact pattern — "loop without a transaction, one transaction per item so one failure doesn't roll back the batch" — is what Vigil's check scheduler needs.** A real requirement, not a contrived example.

**2. Self-injection**

```java
@Service
public class CheckService {
    @Autowired @Lazy
    private CheckService self;                  // the proxy, injected into itself

    public void runAllChecks(List<Monitor> ms) { ms.forEach(self::runCheck); }

    @Transactional public void runCheck(Monitor m) { }
}
```
Works. `@Lazy` prevents the cycle from failing at startup. Reads as a hack, and reviewers will (rightly) ask why the class isn't split. Acceptable in legacy code you can't restructure.

**3. `TransactionTemplate` — programmatic, and honestly underrated**

```java
@Service
@RequiredArgsConstructor
public class CheckService {
    private final TransactionTemplate tx;

    public void runAllChecks(List<Monitor> ms) {
        for (Monitor m : ms) {
            tx.executeWithoutResult(status -> runCheck(m));   // ✅ explicit boundary
        }
    }
}
```
No proxy involved at all, so no self-invocation issue. The transaction boundary is *visible in the code*, which some very experienced teams prefer over annotations precisely because it can't be silently defeated. Good answer to "when would you not use `@Transactional`?"

**4. `AopContext.currentProxy()`** — needs `@EnableAspectJAutoProxy(exposeProxy = true)`, uses a ThreadLocal, and hard-couples your code to Spring AOP. Know it exists; don't reach for it.

**5. AspectJ load-time or compile-time weaving** — the *actual* fix, because there's no proxy: the aspect is woven into the bytecode of your class, so `this.method()` is advised too. Enable with `@EnableLoadTimeWeaving` + `-javaagent:spring-instrument.jar`, or `@EnableTransactionManagement(mode = AdviceMode.ASPECTJ)`. Costs: an agent in your runtime, slower startup, harder debugging, and behaviour that differs from every other Spring app your team knows. Almost never worth it — **but naming it as the only true fix is exactly the depth interviewers are probing for.**

### How to diagnose it in 30 seconds

```java
// 1. Is the bean proxied at all?
System.out.println(ctx.getBean(CheckService.class).getClass().getName());
// CheckService                      → NOT proxied (no matching pointcut — check your annotations)
// CheckService$$SpringCGLIB$$0      → CGLIB proxy ✅
// com.sun.proxy.$Proxy123           → JDK proxy

// 2. Is a transaction actually active inside the method?
log.info("tx active: {}", TransactionSynchronizationManager.isActualTransactionActive());
```

Or turn on the logs and *see* it:
```properties
logging.level.org.springframework.transaction.interceptor=TRACE
logging.level.org.springframework.orm.jpa.JpaTransactionManager=DEBUG
```
You'll get lines like `Getting transaction for [com.vigil.CheckService.runCheck]`. **If that line is missing, the proxy wasn't involved.** That's your proof.

**Practise the diagnostic, not just the theory.** "I'd print the bean's class name and enable transaction-interceptor TRACE logging" is a far better interview answer than a definition.

---

## 6. The same mechanism elsewhere: Hibernate lazy loading

```java
Monitor m = repo.findById(id).orElseThrow();
Organization org = m.getOrganization();     // a Hibernate proxy: Organization$HibernateProxy$xY
org.getName();                              // triggers SELECT ... or throws LazyInitializationException
```

Hibernate generates a subclass of `Organization` that overrides every getter to fetch-on-first-access. **Same pattern, different framework** — and the same limitations apply:

- A `final` entity class can't be proxied → Hibernate silently falls back to eager loading.
- `final` getters can't be intercepted → same.
- `entity.getId()` often *doesn't* trigger a query, because the proxy already holds the ID. Great trivia, and genuinely useful: you can read a lazy association's ID without a database hit.

Once you see proxies as a general technique rather than a Spring feature, `LazyInitializationException` becomes obvious rather than mysterious (Lesson 20).

---

## 7. Build & break

Work in `com.vigil.playground.aop`. **Exercise 3 is the one that matters most.**

**1. Prove which proxy type you get.**
```java
public interface Greeter { String greet(String n); }

@Service
public class GreeterImpl implements Greeter {
    @Transactional public String greet(String n) { return "hi " + n; }
}
```
```java
@Component @RequiredArgsConstructor
class ProxyReporter implements ApplicationRunner {
    private final ApplicationContext ctx;
    public void run(ApplicationArguments a) {
        Object bean = ctx.getBean(Greeter.class);
        System.out.println("class:      " + bean.getClass().getName());
        System.out.println("is CGLIB:   " + bean.getClass().getName().contains("CGLIB"));
        System.out.println("is JDK:     " + Proxy.isProxyClass(bean.getClass()));
        System.out.println("interfaces: " + Arrays.toString(bean.getClass().getInterfaces()));
    }
}
```
Run it → CGLIB. Now add `spring.aop.proxy-target-class=false` and re-run → JDK proxy. **Then try `@Autowired GreeterImpl` with JDK proxying on** and read `BeanNotOfRequiredTypeException`. You've just experienced the exact reason Boot defaults to CGLIB.

**2. Watch a proxy vanish.** Remove `@Transactional` from `greet` and re-run exercise 1. The class name is now plain `GreeterImpl` — **no proxy at all**, because no pointcut matched. Proves the "per-bean, only if something matches" rule.

**3. ⭐ Reproduce self-invocation and fix it four ways.**

```java
@Service
@Slf4j
public class SelfInvocationDemo {

    public void outer() {
        log.info("outer  → tx active: {}", TransactionSynchronizationManager.isActualTransactionActive());
        inner();                                     // ← the bug
    }

    @Transactional
    public void inner() {
        log.info("inner  → tx active: {}", TransactionSynchronizationManager.isActualTransactionActive());
    }
}
```

Call `outer()` from a runner. Expected output:
```
outer  → tx active: false
inner  → tx active: false      ← 💥 should be true!
```
Now call `inner()` directly from the runner:
```
inner  → tx active: true       ← ✅ proxy was involved
```

**Seeing those two lines differ, from code you wrote, is worth more than reading this section ten times.**

Then fix it four ways and verify `tx active: true` each time:
- extract `inner` into a second bean
- `@Autowired @Lazy` self-injection
- `TransactionTemplate`
- `AopContext.currentProxy()` with `@EnableAspectJAutoProxy(exposeProxy = true)`

**4. Feel the silent failures.** Make `inner()` `private`, then `final`, then `static`. Each time, note there is **no error** and `tx active` is `false`. Then make the *class* `final` and observe that this one fails loudly at startup — and note the asymmetry.

**5. Turn on the evidence.** Add:
```properties
logging.level.org.springframework.transaction.interceptor=TRACE
```
Re-run exercise 3. You'll see `Getting transaction for [...inner]` **only** in the direct-call case. That log line's presence or absence is your diagnostic tool forever.

### Explain out loud (60s)

1. Why Spring chose proxies over AspectJ weaving, and the limitation that choice created.
2. The self-invocation trace: which object holds which reference at each hop.
3. The three silent killers (`private`, `final`, `static`) and why the *class* being `final` fails loudly instead.
4. JDK proxy vs CGLIB, and the error Boot's default exists to prevent.
5. Why `@Transactional`, `@Cacheable` and `@Retryable` must all be `@Around` advice.

---

## 8. Production rules

| Do / Never | Why |
|---|---|
| **Never annotate a `private`, `final` or `static` method with `@Transactional`, `@Cacheable`, `@Async` or `@PreAuthorize`** | CGLIB advises by overriding; none of the three can be overridden, so the annotation is inert with no error, no warning and no log line |
| **Never call an advised method through `this.`** | Execution is already inside the target, so the proxy is out of the picture — on `@PreAuthorize` this turns a silent bug into an authorization bypass |
| **Fix self-invocation by extracting the advised method into its own bean** | The injected reference is the proxy, so the call goes through it again — and the extraction usually improves the design on its own |
| **Treat a bean as proxied only after you've seen `$$SpringCGLIB` or `$Proxy` in its class name** | Proxying is per-bean and only happens if a pointcut matched; everything else you debug before checking this is wasted effort |
| **Leave `proxyTargetClass=true` (Boot's default) alone** | A JDK proxy implements only your interfaces, so injecting the concrete class fails with `BeanNotOfRequiredTypeException` and methods off the interface vanish |
| **Expect `@Around` whenever an annotation changes control flow** | `@Transactional`, `@Cacheable`, `@Retryable` and `@Async` all need the ability to skip, repeat or wrap `proceed()` — knowing that explains all four at once |
| **Remember a join point in Spring AOP is always a method execution** | Field reads and constructor calls can't be intercepted by a wrapper, so anything needing those requires real AspectJ weaving |
| **Don't put proxied behaviour inside `@PostConstruct`** | Proxies are created after initialization, so advice on anything you call on `this` there simply isn't attached yet |

---

## 9. Interview traps

**Q1. What is AOP and what problem does it solve?**
A way to modularize cross-cutting concerns (transactions, security, logging, metrics, caching) that would otherwise be duplicated across many classes. You declare the concern once (an aspect) plus a pattern for where it applies (a pointcut), and the framework weaves it in — leaving business methods containing only business logic.

**Q2. Define aspect, join point, pointcut, advice, weaving.**
Aspect = the module. Join point = a point where advice could apply (in Spring AOP, **always a method execution**). Pointcut = an expression selecting join points. Advice = the code that runs. Weaving = linking them (Spring: at runtime via proxies; AspectJ: at compile/load time via bytecode).

**Q3. Spring AOP vs AspectJ?**
Spring AOP: proxy-based, runtime, method execution only, works on Spring beans only, no special compiler/agent. AspectJ: bytecode weaving at compile or load time, supports field access/constructors/static init, works on any object, and **self-invocation is advised** — at the cost of an agent or special compiler.

**Q4. Which advice type can prevent the target from running, and what real features use that?**
`@Around` — by not calling `proceed()`. Caching (`@Cacheable` returns a hit without invoking the method) and retry (calling `proceed()` repeatedly) both depend on it. `@Transactional` uses `@Around` for try/commit/rollback.

**Q5. JDK dynamic proxy vs CGLIB — which does Boot use, and what breaks with each?**
Boot defaults to CGLIB (`proxyTargetClass=true`). JDK proxies require an interface and the resulting object is *not* an instance of your class (→ `BeanNotOfRequiredTypeException` when injecting by concrete type, and non-interface methods are unreachable). CGLIB subclasses, so it can't handle `final` classes/methods or `private` methods.

**Q6. Why is `@Transactional` on a `private` method silently ignored?**
CGLIB advises by overriding methods; `private` (and `final`, and `static`) methods can't be overridden, so no interception point exists. No error is raised — the annotation is inert.

**Q7. Explain self-invocation and give three fixes.** *(the single most likely AOP question)*
Proxy AOP intercepts only calls arriving through the proxy. `this.method()` executes inside the target and bypasses it, so the annotation has no effect. Fixes: extract into a separate bean (best — usually reveals a real separation of concerns); self-inject the proxy with `@Lazy`; use `TransactionTemplate` for explicit boundaries; `AopContext.currentProxy()`; or AspectJ weaving (the only fix that makes self-calls genuinely advised).

**Q8. When is a bean proxied, and does an un-annotated bean pay any cost?**
At `BeanPostProcessor.postProcessAfterInitialization`, and only if some advisor's pointcut matches at least one of its methods. If nothing matches, the original object is returned unchanged — no proxy, no overhead.

**Q9. How do you check at runtime whether AOP is actually applied?**
Print `bean.getClass().getName()` — a `$$SpringCGLIB$$` or `$Proxy` suffix means proxied. For transactions specifically, `TransactionSynchronizationManager.isActualTransactionActive()` inside the method, and `logging.level.org.springframework.transaction.interceptor=TRACE`.

**Q10. Name five Spring features built on the same proxy mechanism.**
`@Transactional`, `@Async`, `@Cacheable`, `@Retryable`, `@PreAuthorize` (plus `@Validated`, `@Observed`, and Hibernate's lazy loading, which uses the same technique in a different framework).

**Q11. Does CGLIB need a default constructor?**
It historically did, because subclass instantiation called a superclass constructor. Modern Spring uses **Objenesis** to allocate the proxy without invoking a constructor, so constructor-injected beans proxy fine.

---

## 10. Retention check

1. Why does Spring AOP only support method-execution join points? Tie it to the implementation choice.
2. Trace, object by object, why `@Transactional` doesn't apply when a method is called via `this.`.
3. Name the three method modifiers that silently disable proxy-based annotations, and say why "silently" is the dangerous part.
4. Explain why `@Cacheable` and `@Retryable` must be `@Around` advice.
5. Give the two commands/log settings you'd use to prove whether a transaction is active.
6. Why does Spring Boot default to CGLIB, and what specific error did that decision prevent?

---

**Next:** [Lesson 10 — AOP in practice](10-aop-in-practice.md) — writing your own aspects, pointcut syntax, and the discipline of *not* overusing them.

# Lesson 04 — Annotations, reflection & proxies

> **This is the most important lesson in the course.** Everything people call "Spring magic" is built from three Java features: annotations (metadata), reflection (reading metadata and acting on it at runtime), and proxies (wrapping objects to add behaviour). Learn these three and Spring becomes a *predictable* framework you can reason about — including parts of it you've never seen.
>
> Do not skim this. Read it twice if you need to.

**Time:** ~90 minutes · **Prereq:** Lessons 01–03

---

## Part 1 — Annotations: metadata, and nothing more

### An annotation does absolutely nothing by itself

This is the single most liberating fact in this lesson:

```java
@Service
public class MonitorService { }
```

`@Service` has **no behaviour**. It does not register anything. It does not run code. It is a sticky note on the class. If you deleted Spring from your project and kept `@Service`, the code would compile and run identically — with nothing happening.

Something else has to **come along, look for the sticky note, and act on it.** That something is the Spring container. Annotations are a *declarative vocabulary*; the framework is the *interpreter*.

Say that back to yourself: **annotations are inert metadata; frameworks give them meaning.** Almost every "how does Spring know?" question dissolves into "some Spring class scanned for that annotation at startup."

### Writing your own annotation

```java
package com.vigil.audit;

import java.lang.annotation.*;

@Target(ElementType.METHOD)              // where it can be placed
@Retention(RetentionPolicy.RUNTIME)      // CRITICAL — see below
@Documented
public @interface Audited {
    String action();                     // required attribute (no default)
    Severity severity() default Severity.INFO;   // optional attribute
}
```

```java
@Audited(action = "MONITOR_DELETED", severity = Severity.WARN)
public void deleteMonitor(UUID id) { ... }
```

Note `@interface` — an annotation *is* a special kind of interface. Its attributes look like methods because they are: you read them via reflection by calling `annotation.action()`.

### `@Retention` — the one that actually matters

```java
RetentionPolicy.SOURCE   // discarded by the compiler. Not in the .class file at all.
RetentionPolicy.CLASS    // in the .class file, but NOT loaded into the JVM at runtime (default!)
RetentionPolicy.RUNTIME  // in the .class file AND visible to reflection ← Spring needs this
```

**Every Spring annotation is `RUNTIME`.** It must be, because Spring reads it while your app is starting.

| Retention | Example | Who reads it |
|---|---|---|
| `SOURCE` | `@Override`, Lombok's `@Getter` | The compiler / annotation processor |
| `CLASS` | `@Nullable` (some flavours) | Bytecode tools, static analysis |
| `RUNTIME` | `@Service`, `@Transactional`, `@GetMapping` | **The framework, via reflection** |

> **This explains a real confusion:** `@Override` genuinely does nothing at runtime — it's a compile-time assertion that gets thrown away. `@Transactional` genuinely *must* survive to runtime. Same syntax, completely different lifecycle. When someone says "annotations are just comments," they're right about `SOURCE` and wrong about `RUNTIME`.

### `@Target` — where an annotation is legal

```java
ElementType.TYPE            // class, interface, enum, record
ElementType.METHOD
ElementType.FIELD
ElementType.PARAMETER       // @RequestParam, @PathVariable
ElementType.CONSTRUCTOR
ElementType.ANNOTATION_TYPE // meta-annotations!
```

### Meta-annotations & composition — how `@SpringBootApplication` works

An annotation can be annotated with other annotations. Spring exploits this heavily.

```java
@Target(TYPE)
@Retention(RUNTIME)
@Component                     // ← @Service IS-A @Component
public @interface Service {
    @AliasFor(annotation = Component.class)
    String value() default "";
}
```

`@Service`, `@Repository`, `@Controller`, `@RestController`, `@Configuration` are all **meta-annotated with `@Component`**. Spring's scanner doesn't look for `@Service` specifically — it looks for anything *meta-annotated* with `@Component`, at any depth. So `@Service` is discovered for free.

The famous example:

```java
@SpringBootConfiguration      // = @Configuration
@EnableAutoConfiguration      // the auto-config trigger (Lesson 11)
@ComponentScan                // scan this package and below
public @interface SpringBootApplication { }
```

Three annotations wearing one hat. **This is why you can write your own:**

```java
@Target(TYPE)
@Retention(RUNTIME)
@RestController
@RequestMapping("/api/v1")
@Validated
public @interface ApiV1Controller { }
```

Now `@ApiV1Controller` on a class gives you all three behaviours. Real teams do this to enforce conventions. Being able to explain *why* it works — Spring's `MergedAnnotations` API resolves meta-annotations recursively — is a strong senior signal.

### Why annotations became Java's dominant configuration style

> **Fun fact / history:** Spring 1.0 (2004) configured everything in XML:
> ```xml
> <bean id="monitorService" class="com.vigil.MonitorService">
>     <constructor-arg ref="monitorRepository"/>
> </bean>
> ```
> A real application had thousands of lines of this, and it was the #1 complaint about Spring — "XML hell." Java 5 (2004) introduced annotations; Spring 2.5 (2007) added `@Component`/`@Autowired`; Spring 3.0 (2009) added `@Configuration`/`@Bean` so config could be pure Java. Spring Boot (2014) then made annotations plus convention the default.
>
> The trade-off is real and worth knowing both sides of: XML kept configuration *external* and *centralized* (you could see the whole wiring in one place, and change it without recompiling); annotations put configuration *next to the code* (concise, refactor-safe, type-checked, but scattered). Modern Spring quietly gives you both — annotations for your beans, external properties for environment config. Being able to state that trade-off fairly is much more impressive than "XML was bad."

---

## Part 2 — Reflection: the engine that reads the metadata

**Reflection is the ability of a program to inspect and manipulate itself at runtime** — to ask a class what its methods are, read annotations, create instances, call methods, and set private fields, all using *names as data* rather than compile-time symbols.

```java
Class<?> clazz = Class.forName("com.vigil.MonitorService");     // load by name (a String!)

for (Method m : clazz.getDeclaredMethods()) {                    // enumerate methods
    if (m.isAnnotationPresent(Audited.class)) {                   // read metadata
        Audited a = m.getAnnotation(Audited.class);
        System.out.println(m.getName() + " audits " + a.action());
    }
}

Constructor<?> ctor = clazz.getDeclaredConstructors()[0];        // inspect constructor
Class<?>[] paramTypes = ctor.getParameterTypes();                // ← what dependencies it needs
Object instance = ctor.newInstance(dependency1, dependency2);    // CREATE the object

Field f = clazz.getDeclaredField("secretKey");
f.setAccessible(true);                                           // bypass `private`!
f.set(instance, "value");                                        // ← this is field injection

Method method = clazz.getMethod("check", Monitor.class);
method.invoke(instance, monitor);                                // ← this is how MVC calls your controller
```

Read those lines again. Every one corresponds to something Spring does:

| Reflection call | What Spring uses it for |
|---|---|
| `Class.forName(...)` | Loading auto-configuration classes named in a text file |
| `getDeclaredMethods()` + `isAnnotationPresent` | Finding `@GetMapping`, `@Bean`, `@EventListener`, `@Scheduled` methods |
| `getParameterTypes()` on a constructor | **Working out what to inject** |
| `ctor.newInstance(...)` | Creating your beans |
| `field.setAccessible(true); field.set(...)` | `@Autowired` on a private field, `@Value` injection |
| `method.invoke(...)` | Calling your controller method when a request arrives |

**That's dependency injection.** There is no magic. Spring reads your constructor's parameter types via reflection, finds objects of those types in its map, and calls `newInstance`. You could write a crude version in an afternoon — and in the Build section, you will.

### Reflection's costs and dangers (know these — interviewers probe here)

**1. It's slower than direct calls.** A reflective `invoke` involves access checks and boxing. The JIT has gotten remarkably good at optimizing hot reflective calls, so it's often not the bottleneck it once was, but it's never free. This is why Spring reflects *once at startup* to build its object graph, then mostly makes normal calls afterwards. **That design decision is why Spring apps have slow startup but normal runtime speed.**

**2. It breaks compile-time safety.** `getMethod("chekc")` compiles fine and fails at runtime. Every string-based Spring feature (bean names, SpEL expressions, property keys, JPQL) inherits this. It's the fundamental tax of a reflective framework, and it's why **Spring's error messages are so unusually detailed** — they're compensating.

**3. It breaks encapsulation.** `setAccessible(true)` reads your private fields. Great for frameworks (JPA sets entity fields, Jackson populates DTOs, JUnit injects mocks); terrifying as a general practice.

**4. It fights ahead-of-time compilation.** A GraalVM native image must know at build time which classes will be reflected on, because there's no dynamic class loading. This is why Spring Boot 3 introduced **AOT processing**: at build time it computes the bean definitions and generates reflection hints. Understanding this makes native-image questions easy (Lesson 36).

> **Fun fact:** Since Java 9's module system, `setAccessible(true)` on JDK internals triggers warnings and, increasingly, hard failures. You may see `--add-opens java.base/java.lang=ALL-UNNAMED` in JVM args — that's a library asking for permission the module system withdrew. Modern Spring avoids needing it.

---

## Part 3 — Proxies: the mechanism behind `@Transactional`, `@Cacheable`, `@Async`

Here's the question that separates people who "use Spring" from people who *understand* it:

```java
@Service
public class MonitorService {
    @Transactional
    public void createMonitor(CreateMonitorRequest req) {
        repo.save(new Monitor(req));
    }
}
```

Your method body contains **no transaction code**. No `begin`, no `commit`, no `rollback`. Yet a transaction happens. **How?**

The answer: **the object you're holding is not your object.**

### The proxy pattern

When Spring sees `@Transactional`, it does not modify your class. It creates a *different* object — a **proxy** — that implements the same interface (or extends your class), holds a reference to your real instance, and wraps every call:

```
Your code calls:  monitorService.createMonitor(req)
                          │
                          ▼
            ┌──────────────────────────────┐
            │  MonitorService$$SpringCGLIB │   ← the PROXY (what's in the container)
            │                              │
            │  createMonitor(req) {        │
            │     tx = txManager.begin();  │   ← added behaviour BEFORE
            │     try {                    │
            │        target.createMonitor(req);  ──────┐
            │        tx.commit();          │           │
            │     } catch (RuntimeEx e) {   │           │
            │        tx.rollback();        │   ← added behaviour AFTER
            │        throw e;              │           │
            │     }                        │           │
            │  }                           │           │
            └──────────────────────────────┘           │
                                                        ▼
                                     ┌──────────────────────────────┐
                                     │  MonitorService (YOUR class) │
                                     │  createMonitor(req) {        │
                                     │     repo.save(...);          │  ← your code, untouched
                                     │  }                           │
                                     └──────────────────────────────┘
```

**Every annotation that "adds behaviour around a method" works this way:** `@Transactional`, `@Cacheable`, `@Async`, `@Retryable`, `@PreAuthorize`, `@Validated`, `@Observed`. One mechanism, many features. Learn it once.

### The two proxy technologies

| | **JDK dynamic proxy** | **CGLIB proxy** |
|---|---|---|
| Built into the JDK | ✅ | ❌ (bundled inside spring-core) |
| Requires | The bean to implement an **interface** | Nothing — **subclasses** your class |
| Creates | A class implementing the same interfaces | A dynamically generated **subclass** |
| Can't proxy | Classes without interfaces | `final` classes, `final` methods, `private` methods |
| Spring Boot default | — | ✅ (`proxyTargetClass=true` since Boot 2) |

**The consequences you must remember:**

```java
@Service
public final class MonitorService {        // ❌ CGLIB can't subclass a final class
    @Transactional
    public void doThing() { }
}
```

```java
@Service
public class MonitorService {
    @Transactional
    private void doThing() { }             // ❌ private → can't be overridden → NOT proxied
                                           //    Silently does nothing. No error.
    @Transactional
    final void doOther() { }               // ❌ final → same problem
}
```

**Silent failure is the danger.** No exception, no warning — your transaction just isn't there. This is a top-tier interview question because it's a real bug people ship.

> **Fun fact:** CGLIB ("Code Generation Library") was an independent project that got so essential that Spring **repackaged it inside spring-core** (`org.springframework.cglib`) to avoid version conflicts with applications that used their own copy. Similarly, Spring inlines its own copy of ASM (the bytecode library CGLIB uses). If you ever wondered why `spring-core` is big — that's part of why. Modern Spring is gradually moving toward ByteBuddy-style approaches and AOT-generated code instead.

### THE SELF-INVOCATION TRAP — memorize this

This is the #1 real-world Spring bug, and the #1 question that separates candidates.

```java
@Service
public class ReportService {

    public void generateAll(List<UUID> orgIds) {
        for (UUID id : orgIds) {
            generateOne(id);            // ⚠️ plain `this.generateOne(id)`
        }
    }

    @Transactional
    public void generateOne(UUID id) {
        // ...
    }
}
```

**`@Transactional` on `generateOne` does nothing when called from `generateAll`.**

Why? Trace the object references:

1. The container holds the **proxy**. Your controller calls `proxy.generateAll(...)`.
2. The proxy has no advice for `generateAll` (no annotation), so it delegates to `target.generateAll(...)`.
3. Inside the **target**, `this` is the *real object*, not the proxy.
4. `this.generateOne(id)` is an ordinary Java method call straight into the target. **The proxy is never involved.** No transaction.

```
Controller → [PROXY].generateAll()  →  [TARGET].generateAll()
                                              │
                                              └─ this.generateOne()  →  [TARGET].generateOne()
                                                                         ↑ proxy bypassed!
```

**This applies identically to `@Cacheable`, `@Async`, `@Retryable`, `@PreAuthorize`.** "My `@Async` method runs synchronously" — 95% of the time, this is why.

**Fixes, in order of preference:**

1. **Move the annotated method to a different bean.** This is almost always right, because the need for self-invocation usually reveals that two responsibilities are stuck in one class.
   ```java
   @Service class ReportOrchestrator {          // no @Transactional
       private final ReportGenerator generator; // ← injected PROXY
       void generateAll(List<UUID> ids) { ids.forEach(generator::generateOne); }
   }
   @Service class ReportGenerator {
       @Transactional void generateOne(UUID id) { ... }
   }
   ```
2. **Self-inject the proxy** — works, but a code smell that signals a design problem:
   ```java
   @Service class ReportService {
       @Autowired @Lazy private ReportService self;   // @Lazy avoids a circular-dependency failure
       void generateAll(List<UUID> ids) { ids.forEach(self::generateOne); }
   }
   ```
3. **`AopContext.currentProxy()`** — requires `@EnableAspectJAutoProxy(exposeProxy = true)`. Rare, ugly, but exists.
4. **AspectJ load-time/compile-time weaving** — rewrites the bytecode itself so there's no proxy at all and self-invocation works. Powerful, but adds a weaving agent to your build/runtime. Rarely worth it; knowing it's the "real" fix is a great interview detail.

**Say this out loud until it's automatic:** *"Proxy-based AOP only intercepts calls that come through the proxy. `this.method()` bypasses the proxy, so the annotation has no effect."*

### The other proxy consequence: lazy loading in JPA

```java
Monitor m = monitorRepo.findById(id).orElseThrow();
Organization org = m.getOrganization();      // this is a Hibernate PROXY, not a real Organization
org.getName();                                // ← NOW the SQL runs (or throws LazyInitializationException)
```

Hibernate uses the exact same technique — a generated subclass that intercepts getters and fetches from the DB on first access. Same mechanism, different framework. Once you see proxies, `LazyInitializationException` stops being mysterious (Lesson 20).

---

## Part 4 — Putting it together: how Spring actually starts

You now have every piece. Here's the whole thing in ~10 steps:

```
1.  main() calls SpringApplication.run(VigilApplication.class, args)

2.  Spring creates an ApplicationContext (basically: a Map<String, Object> plus lifecycle machinery)

3.  It reads @SpringBootApplication on your class via REFLECTION,
    resolves its META-ANNOTATIONS → finds @ComponentScan and @EnableAutoConfiguration

4.  COMPONENT SCAN: walks the classpath under com.vigil, reading each .class file's
    ANNOTATIONS (via ASM bytecode parsing — faster than loading every class),
    collecting anything meta-annotated with @Component

5.  AUTO-CONFIGURATION: reads AutoConfiguration.imports from every starter JAR,
    evaluates each @Conditional (is Tomcat on the classpath? is a DataSource bean absent?),
    and keeps the ones that apply                                        (Lesson 11)

6.  Now it has BEAN DEFINITIONS — recipes, not objects: class, scope, dependencies, init method

7.  It topologically sorts them by dependency and instantiates each:
    read the constructor's PARAMETER TYPES via reflection → look up matching beans
    → recurse if needed → ctor.newInstance(deps)          ← DEPENDENCY INJECTION

8.  BeanPostProcessors get a shot at each new bean. This is where PROXIES are created:
    "does this bean have @Transactional/@Async/@Cacheable anywhere? then wrap it"   (Lesson 08)

9.  Lifecycle callbacks fire: @PostConstruct, InitializingBean, ApplicationRunner

10. Embedded Tomcat starts; DispatcherServlet maps @GetMapping methods to URLs
    (via reflection, of course) and begins accepting requests             (Lesson 14)
```

**That's Spring.** Reflection reads metadata, builds a map of objects, wraps some of them in proxies, and hands them to each other. Every remaining lesson in this course is detail on top of that skeleton.

If you can narrate those ten steps in an interview, you will immediately be treated as a different class of candidate — because nearly nobody can.

---

## Part 5 — Build this: write your own mini Spring

**This is the most valuable exercise in the course. Do not skip it.** In about 100 lines you'll build a working DI container with annotation scanning and a proxy-based aspect. After this, Spring's behaviour is something you *derived*, not something you memorized.

Create `com.vigil.playground.minispring`.

**Step 1 — Your annotations:**
```java
@Target(TYPE)   @Retention(RUNTIME) public @interface MyComponent {}
@Target(METHOD) @Retention(RUNTIME) public @interface MyTransactional {}
```

**Step 2 — Some beans** (note: no framework code inside them):
```java
@MyComponent
public class MonitorRepository {
    public void save(String name) { System.out.println("  SQL: insert " + name); }
}

@MyComponent
public class MonitorService {
    private final MonitorRepository repo;
    public MonitorService(MonitorRepository repo) { this.repo = repo; }

    @MyTransactional
    public void createMonitor(String name) { repo.save(name); }

    public void createTwo(String a, String b) {   // for the self-invocation experiment
        createMonitor(a);
        createMonitor(b);
    }
}
```

**Step 3 — The container.** Register classes explicitly (real component scanning needs classpath walking; skip that complexity):

```java
public class MiniContainer {
    private final Map<Class<?>, Object> beans = new HashMap<>();

    public void register(Class<?>... classes) {
        for (Class<?> c : classes) {
            if (!c.isAnnotationPresent(MyComponent.class))
                throw new IllegalArgumentException(c + " is not @MyComponent");
            beans.put(c, null);           // definitions first, instances later
        }
        for (Class<?> c : new ArrayList<>(beans.keySet())) getBean(c);
    }

    @SuppressWarnings("unchecked")
    public <T> T getBean(Class<T> type) {
        Object existing = beans.get(type);
        if (existing != null) return (T) existing;

        try {
            Constructor<?> ctor = type.getDeclaredConstructors()[0];
            // ---- DEPENDENCY INJECTION: resolve each parameter recursively ----
            Object[] args = Arrays.stream(ctor.getParameterTypes())
                                  .map(this::getBean)
                                  .toArray();
            Object instance = ctor.newInstance(args);

            // ---- AOP: wrap in a proxy if any method is @MyTransactional ----
            Object bean = needsTx(type) ? proxy(instance) : instance;

            beans.put(type, bean);
            return (T) bean;
        } catch (Exception e) {
            throw new RuntimeException("Failed to create bean " + type.getName(), e);
        }
    }

    private boolean needsTx(Class<?> type) {
        return Arrays.stream(type.getDeclaredMethods())
                     .anyMatch(m -> m.isAnnotationPresent(MyTransactional.class));
    }

    // JDK dynamic proxies need an interface, so use CGLIB-style... or cheat with an
    // InvocationHandler over an interface. Simplest honest version: use java.lang.reflect.Proxy
    // if your bean implements an interface, else print what WOULD happen.
    private Object proxy(Object target) {
        Class<?>[] interfaces = target.getClass().getInterfaces();
        if (interfaces.length == 0) {
            System.out.println("  [no interface — real Spring would use CGLIB here]");
            return target;                        // ← observe this limitation yourself!
        }
        return Proxy.newProxyInstance(
            target.getClass().getClassLoader(), interfaces,
            (p, method, args) -> {
                boolean tx = target.getClass()
                        .getMethod(method.getName(), method.getParameterTypes())
                        .isAnnotationPresent(MyTransactional.class);
                if (!tx) return method.invoke(target, args);
                System.out.println("BEGIN TX");
                try {
                    Object r = method.invoke(target, args);
                    System.out.println("COMMIT");
                    return r;
                } catch (InvocationTargetException e) {
                    System.out.println("ROLLBACK");
                    throw e.getCause();
                }
            });
    }
}
```

**Step 4 — Make it work with an interface** so the proxy engages. Extract `MonitorOperations` with `createMonitor` and `createTwo`, have `MonitorService implements MonitorOperations`, and get the bean by interface type. (You'll need `getBean` to check "which registered class implements this interface" — a small addition, and *exactly* the by-type resolution Spring does in Lesson 07.)

**Step 5 — Run the experiments and explain each result:**

```java
var c = new MiniContainer();
c.register(MonitorRepository.class, MonitorService.class);
MonitorOperations svc = c.getBean(MonitorOperations.class);

System.out.println(svc.getClass().getName());   // 1. NOT MonitorService! It's $Proxy0

svc.createMonitor("api");                        // 2. BEGIN TX / SQL / COMMIT ✅

svc.createTwo("a", "b");                         // 3. NO TX AT ALL — you just reproduced
                                                 //    the self-invocation bug yourself 🎯
```

**Sit with result #3.** You wrote the framework, so you know *exactly* why: `createTwo` runs in the target, and `this.createMonitor(...)` never touches the proxy. You will never be confused by `@Transactional` again.

**Bonus experiments:**
- Add a bean whose constructor needs a class you didn't register → see your own "no such bean" error, then compare it to Spring's (Spring's is much better; note *what extra context* it gives).
- Make `MonitorService` depend on a bean that depends back on it → your container recurses forever. That's a circular dependency, and now you understand *why* Spring has to detect it (Lesson 07).
- Add a second implementation of `MonitorOperations` → your container can't decide. That's exactly `NoUniqueBeanDefinitionException`, and it's why `@Qualifier` and `@Primary` exist.

---

## Part 6 — Interview traps

**Q1. What does an annotation actually do?**
Nothing. It's metadata attached to a program element. A framework must read it (via reflection, if `RetentionPolicy.RUNTIME`) and implement behaviour. `@Override` is `SOURCE` retention and truly vanishes; `@Transactional` is `RUNTIME` and is read by Spring at startup.

**Q2. Why must Spring annotations be `RUNTIME` retention?**
Because Spring inspects them while the application runs (at context startup). `CLASS` retention (the default!) keeps them in the bytecode but the JVM doesn't expose them to reflection; `SOURCE` discards them entirely.

**Q3. How does Spring find your `@Service` classes?**
Component scanning: it walks the classpath under the scanned packages and reads each class's annotations — using ASM to parse bytecode rather than loading every class, for speed. It matches anything meta-annotated with `@Component`, which is why `@Service`/`@Repository`/`@Controller` are all found by the same rule.

**Q4. How does dependency injection actually work?**
Spring reflects on the bean's constructor, reads its parameter types, resolves a bean for each type (recursively creating them if needed), and calls `Constructor.newInstance(args)`. Field injection instead uses `Field.setAccessible(true)` + `Field.set()`.

**Q5. What is a proxy, and which annotations depend on one?**
An object that implements the same type as your bean, wraps it, and adds behaviour before/after delegating. `@Transactional`, `@Cacheable`, `@Async`, `@Retryable`, `@PreAuthorize`, `@Validated`, `@Observed` — all proxy-based.

**Q6. JDK dynamic proxy vs CGLIB — which does Spring Boot use and why does it matter?**
JDK proxies implement interfaces (bean must have one); CGLIB generates a subclass (no interface needed, but can't proxy `final` classes/methods or `private` methods). Spring Boot defaults to CGLIB (`proxyTargetClass=true`) so proxying works without interfaces. It matters because `final`/`private` annotated methods are **silently not advised**.

**Q7. Explain the self-invocation problem.** *(If you get one AOP question, it's this one.)*
Proxy AOP only intercepts calls arriving through the proxy. When one method of a bean calls another with `this.method()`, the call happens inside the target object and bypasses the proxy — so `@Transactional`/`@Async`/`@Cacheable` on the inner method has no effect, silently. Fix by extracting the method into a separate bean (best), self-injecting `@Lazy`, or using AspectJ weaving.

**Q8. Why is `@Transactional` on a `private` method a bug?**
CGLIB proxies work by overriding methods; `private` methods can't be overridden, so the proxy can't intercept them. There's no error — the annotation is simply ignored. Same for `final` methods and `final` classes.

**Q9. Reflection's downsides?**
Performance cost vs direct invocation; loss of compile-time safety (string-based lookups fail at runtime); breaks encapsulation via `setAccessible`; and it's hostile to ahead-of-time compilation — which is why Spring Boot 3's AOT engine generates reflection hints for GraalVM native images.

**Q10. Why is Spring startup slow but runtime fast?**
All the reflective work — scanning, condition evaluation, dependency resolution, proxy generation — happens once at startup. After that, calls are mostly ordinary method invocations through pre-built proxies. Mitigations for startup: lazy initialization, AOT processing, CDS, native images.

**Q11. What are meta-annotations, and give a real use.**
Annotations on annotations. Spring resolves them recursively (`MergedAnnotations`), which is how `@Service` inherits `@Component`'s meaning and how `@SpringBootApplication` bundles three annotations. You can compose your own — e.g. `@ApiV1Controller` combining `@RestController` + `@RequestMapping("/api/v1")` + `@Validated` — to enforce team conventions in one place.

---

## Part 7 — Retention check

Answer out loud, no notes:

1. Someone says "Spring is too magical." Rebut it in four sentences using the words *annotation*, *reflection*, *proxy*, and *conditional*.
2. A colleague reports "`@Async` runs on the main thread." Give your first hypothesis and the diagnostic you'd run.
3. Why can't Spring proxy a `final` class with CGLIB, and what error/behaviour do you actually observe?
4. Narrate the 10 startup steps from §Part 4. (This is worth practising until fluent — it's the single highest-leverage answer in Spring interviews.)
5. Why does GraalVM native image need Spring's AOT processing? Connect it to reflection.
6. In your mini-container, what line of code *is* dependency injection? What line *is* AOP?

---

## Module 1 complete ✅

You now know the mechanisms. From here, Spring is just an extremely well-engineered application of them.

**Next:** [Lesson 05 — Why Spring exists](../02-spring-core/05-why-spring-exists.md) — the problem Spring was invented to solve, and how to argue about it like someone who's been in the industry.

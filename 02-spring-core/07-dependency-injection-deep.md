# Lesson 07 — Dependency injection, deeply

> **Why this lesson exists:** "Constructor or field injection?" is the most common Spring interview question after "what is DI." Everyone knows the *answer*; almost nobody can give the four independent reasons. And when a real app throws `NoUniqueBeanDefinitionException` or a circular-dependency error, the people who fix it in 30 seconds are the ones who understand the *resolution algorithm*. That's this lesson.

**Time:** ~70 minutes · **Prereq:** Lesson 06

---

## 1. The idea in one sentence

> **Spring resolves dependencies by *type* and uses everything else — `@Primary`, `@Qualifier`, bean names, generics — only to break ties; and constructor injection is the right default because it is the only style in which a wrongly-wired object cannot be constructed at all.**

Those two halves answer most of what you'll be asked here. Ambiguity errors are the tie-breaker chain running out of options, and circular-dependency errors are the compiler-like guarantee of constructor injection doing its job at startup.

---

## 2. The three injection styles

### Constructor injection — the correct default

```java
@Service
public class IncidentService {

    private final IncidentRepository repo;
    private final NotificationChannel channel;
    private final Clock clock;

    public IncidentService(IncidentRepository repo,
                           NotificationChannel channel,
                           Clock clock) {
        this.repo = repo;
        this.channel = channel;
        this.clock = clock;
    }
}
```

**`@Autowired` is not needed on the constructor** when there's only one — Spring has inferred it since 4.3. Adding it is harmless noise. (With *multiple* constructors you must mark the one to use, or provide a `@Bean` method.)

With Lombok, which most teams use:

```java
@Service
@RequiredArgsConstructor            // generates a constructor for all final fields
public class IncidentService {
    private final IncidentRepository repo;
    private final NotificationChannel channel;
    private final Clock clock;
}
```

Concise and correct. **Caveat worth knowing:** `@RequiredArgsConstructor` generates parameters in *field declaration order*, so if two fields have the same type, reordering fields silently swaps the injected values. Rare, but it has caused real bugs. Also, Lombok is a compile-time annotation processor — if your IDE's annotation processing is off, you get thousands of phantom errors. That's the #1 Lombok support question.

### Field injection — avoid

```java
@Service
public class IncidentService {
    @Autowired private IncidentRepository repo;      // ⚠️
    @Autowired private NotificationChannel channel;  // ⚠️
}
```

Spring does this with `field.setAccessible(true); field.set(bean, dep)` (Lesson 04). It works. It's also what your IDE warns about, and Spring's own docs recommend against it.

### Setter injection — for genuinely optional dependencies

```java
@Service
public class IncidentService {
    private MetricsCollector metrics = MetricsCollector.noop();

    @Autowired(required = false)
    public void setMetrics(MetricsCollector metrics) { this.metrics = metrics; }
}
```

Legitimate but rare. The common modern alternative is `ObjectProvider<T>` (§5).

---

## 3. Why constructor injection wins — the four independent reasons

Most candidates give one reason. Give all four and you've visibly out-answered the room.

### Reason 1 — Immutability, and therefore thread safety

Only constructor injection lets fields be `final`. `final` means:
- assigned exactly once, guaranteed by the compiler
- **safely published across threads** — the JVM's final-field semantics guarantee other threads see the fully constructed value

Field injection can't use `final` (the field is set *after* construction), so every dependency is mutable for the object's lifetime. Since beans are singletons shared across all request threads (Lesson 02 §7), immutability isn't a style preference — it's a correctness property.

### Reason 2 — The object is never in an invalid state

```java
// Constructor injection: impossible to construct an incomplete IncidentService.
new IncidentService(repo, channel, clock);      // compiler enforces every dependency
```

```java
// Field injection: this compiles and gives you a landmine.
var service = new IncidentService();            // all fields null
service.openIncident(monitor);                  // 💥 NPE
```

That second case is exactly what happens in a unit test — which leads to reason 3.

### Reason 3 — Testability without a framework

```java
// Constructor injection: plain Java, no Spring, no reflection, no magic.
@Test
void opensIncident() {
    var service = new IncidentService(fakeRepo, recordingChannel, fixedClock);
    ...
}
```

```java
// Field injection: you now need reflection or Spring to build the object.
@ExtendWith(MockitoExtension.class)
class IncidentServiceTest {
    @Mock IncidentRepository repo;
    @InjectMocks IncidentService service;    // Mockito reflectively sets private fields
}
```

`@InjectMocks` works, but you've made your test depend on a framework doing reflection to construct an object that *should* be constructible. And it fails silently in a specific nasty way: add a new dependency to the class and forget to add a `@Mock`, and `@InjectMocks` leaves it **null** — your test NPEs at runtime instead of failing to compile. With constructor injection, adding a dependency **breaks compilation of every test**, which is exactly the feedback you want.

### Reason 4 — It makes bad design visible

```java
@Service
public class OrderService {
    public OrderService(OrderRepository a, PaymentGateway b, InventoryService c,
                        EmailService d, AuditLog e, PricingEngine f, TaxCalculator g,
                        ShippingService h, FraudDetector i, LoyaltyService j) {   // 😬
```

Ten constructor parameters is *ugly*, and the ugliness is the point: this class does too much. **Field injection hides that** — you can stack 20 `@Autowired` fields and it still looks tidy. Constructor injection makes coupling physically painful, which pushes you to split the class.

This is the most sophisticated of the four arguments: **constructor injection preserves a design signal that field injection suppresses.** Say that in an interview and you'll get a nod.

### And the bonus reason: circular dependencies fail loudly

Constructor injection makes a cycle **impossible to construct**, so Spring throws at startup. Field injection *allows* cycles (Spring can set fields after both objects exist), so a bad design silently survives into production. See §6.

### The counterargument, honestly

Constructor injection is verbose without Lombok, and it makes cycles hard — which occasionally means real refactoring work rather than a quick fix. Some teams use field injection in tests or one-off `@Configuration` classes for brevity. That's defensible in small doses. **What isn't defensible is field injection in business services**, and Spring's own documentation says so.

---

## 4. How Spring resolves *which* bean to inject

The algorithm, in order:

```
1. Find all bean definitions assignable to the required TYPE.
   (Includes subtypes and interface implementations, and respects generics.)

2. Exactly one match?  → inject it. Done.

3. Zero matches?
   → is the injection point optional (Optional<T>, @Nullable, required=false,
     ObjectProvider<T>)?  → inject empty/null.
   → else throw NoSuchBeanDefinitionException.

4. Multiple matches? Narrow, in this order:
   a. Is there a @Qualifier on the injection point matching a bean's name/qualifier?
   b. Is exactly one candidate marked @Primary?
   c. Does the parameter/field NAME match a bean name exactly?   ← the subtle one
   d. Is there a @Priority ordering? (jakarta.annotation.Priority — lowest wins)
   → still ambiguous?  → throw NoUniqueBeanDefinitionException.
```

**Step 4c surprises people.** With two `NotificationChannel` beans named `slackChannel` and `emailChannel`:

```java
public IncidentService(NotificationChannel slackChannel) { ... }   // ✅ works — name matched!
public IncidentService(NotificationChannel channel)      { ... }   // ❌ NoUniqueBeanDefinition
```

It's real, and it means **your parameter names are load-bearing** — a rename can break wiring. Worse, parameter names are only preserved if compiled with `-parameters` (Spring Boot's Maven plugin enables it, so it works, but not everywhere). **Don't rely on it.** Use `@Qualifier` when you mean a specific bean.

### `@Primary` — the default choice

```java
@Component @Primary
public class SlackChannel implements NotificationChannel { }

@Component
public class EmailChannel implements NotificationChannel { }
```

Now plain `NotificationChannel` injection gets Slack. Use `@Primary` when there's an obvious default and the others are exceptions.

### `@Qualifier` — naming a specific bean

```java
@Component("slack")  class SlackChannel  implements NotificationChannel { }
@Component("email")  class EmailChannel  implements NotificationChannel { }

public IncidentService(@Qualifier("email") NotificationChannel channel) { ... }
```

**But string qualifiers are fragile** — a typo compiles and fails at startup. The professional pattern is a **custom qualifier annotation**, which is type-safe and refactorable:

```java
@Qualifier
@Retention(RUNTIME)
@Target({FIELD, METHOD, PARAMETER, TYPE})
public @interface Slack { }
```
```java
@Component @Slack
public class SlackChannel implements NotificationChannel { }

public IncidentService(@Slack NotificationChannel channel) { ... }   // ✅ compiler-checked
```

Do this in real projects. It's a small thing that marks a codebase as thoughtfully built.

### `@Primary` vs `@Qualifier` precedence

`@Qualifier` at the injection point **wins** over `@Primary`. Think of `@Primary` as the default and `@Qualifier` as the explicit override. And note the asymmetry: `@Primary` is declared on the *bean*, `@Qualifier` at the *injection point*.

> **Boot 4 delta:** `@Fallback` (Framework 6.2+) is the inverse of `@Primary` — it marks a bean as *only* used if no other candidate exists. Very useful for library-provided defaults that any user bean should quietly displace, without needing `@ConditionalOnMissingBean`.

---

## 5. Injecting collections — the strategy pattern, for free

This is one of Spring's most elegant features and it's *underused*:

```java
@Service
public class NotificationDispatcher {

    private final List<NotificationChannel> channels;      // ALL implementations injected

    public NotificationDispatcher(List<NotificationChannel> channels) {
        this.channels = channels;
    }

    public void broadcast(Incident incident) {
        channels.forEach(c -> c.send(incident.title(), incident.body()));
    }
}
```

Add a `TeamsChannel` class next year, and `NotificationDispatcher` **does not change**. No `if/else`, no registry, no factory. That's Open/Closed satisfied by wiring.

```java
// Map keyed by bean name — for lookup by a runtime string
private final Map<String, NotificationChannel> byName;     // {"slack": ..., "email": ...}

public void send(String channelType, Incident i) {
    var channel = byName.get(channelType);
    if (channel == null) throw new UnsupportedChannelException(channelType);
    channel.send(i.title(), i.body());
}
```

**Ordering matters when the collection is a pipeline** (filters, validators, enrichers):

```java
@Component @Order(1)   class SchemaValidator  implements CheckValidator { }
@Component @Order(2)   class UrlValidator     implements CheckValidator { }
@Component @Order(3)   class RateLimitGuard   implements CheckValidator { }
```

`@Order` (or implementing `Ordered`) controls injected list order — lower runs first. **Without it, the order is effectively arbitrary** (based on scan order), which produces a wonderful class of bug where behaviour changes after an unrelated refactor. If order matters, declare it.

**The self-selecting handler pattern** — real production code, and a great thing to show in an interview:

```java
public interface CheckExecutor {
    boolean supports(MonitorType type);
    CheckOutcome execute(Monitor m);
}

@Service
public class CheckDispatcher {
    private final List<CheckExecutor> executors;
    public CheckDispatcher(List<CheckExecutor> executors) { this.executors = executors; }

    public CheckOutcome run(Monitor m) {
        return executors.stream()
                .filter(e -> e.supports(m.getType()))
                .findFirst()
                .orElseThrow(() -> new NoExecutorException(m.getType()))
                .execute(m);
    }
}
```

Adding SSL-expiry monitoring = write one class with `@Service`. Nothing else in the system changes. **Vigil uses exactly this** for HTTP/TCP/SSL/heartbeat checks.

**Empty collection gotcha:** if there are zero implementations, injecting `List<T>` fails with `NoSuchBeanDefinitionException` in older versions; modern Spring injects an empty list for constructor collection parameters, but be explicit if you need to tolerate none:
```java
public Dispatcher(Optional<List<NotificationChannel>> channels) { ... }
// or
public Dispatcher(ObjectProvider<NotificationChannel> provider) { ... }
```

---

## 6. Optional and lazy dependencies

```java
// 1. Optional<T> — clean, explicit
public AuditService(Optional<MetricsCollector> metrics) {
    this.metrics = metrics.orElseGet(MetricsCollector::noop);
}

// 2. @Autowired(required = false) — legacy style, leaves null
@Autowired(required = false) private MetricsCollector metrics;

// 3. ObjectProvider<T> — the most capable option
public AuditService(ObjectProvider<MetricsCollector> provider) {
    this.metrics = provider.getIfAvailable(MetricsCollector::noop);
}

// 4. @Lazy — inject a proxy; the real bean is created on first use
public ReportService(@Lazy HeavyPdfRenderer renderer) { ... }
```

**`ObjectProvider` is the one to know.** It gives you:
- `getIfAvailable(supplier)` — default when absent
- `getIfUnique()` — null when ambiguous rather than throwing
- `stream()` / `orderedStream()` — lazy iteration over all candidates
- **deferred lookup** — nothing resolved at construction time

That last point is why libraries use it: it avoids forcing bean creation order and dodges cycles.

**`@Lazy` on an injection point** doesn't inject the bean — it injects a *proxy* that creates the real bean on first method call. Uses: breaking a cycle (a band-aid), and deferring genuinely expensive initialization. Cost: you lose fail-fast (a broken bean now explodes at first request instead of at startup), and the proxy has all the usual proxy limitations.

---

## 7. Circular dependencies

```java
@Service
class MonitorService {
    MonitorService(IncidentService incidents) { ... }
}

@Service
class IncidentService {
    IncidentService(MonitorService monitors) { ... }     // 🔁
}
```

Spring cannot build this. To create `MonitorService` it needs `IncidentService`, which needs `MonitorService`, which doesn't exist yet.

```
The dependencies of some of the beans in the application context form a cycle:

┌─────┐
|  monitorService defined in file [.../MonitorService.class]
↑     ↓
|  incidentService defined in file [.../IncidentService.class]
└─────┘

Action: Relying upon circular references is discouraged... Update your application to
remove the dependency cycle between beans.
```

> **Important:** since **Spring Boot 2.6**, circular references are **prohibited by default**. Previously Spring silently resolved field/setter cycles using a three-level cache of partially constructed singletons ("early references"), which meant cycles quietly existed in thousands of applications. The default was flipped because those cycles caused subtle bugs — most nastily, a bean could receive a *pre-proxy* reference to its collaborator, so `@Transactional` on the injected bean silently didn't apply. Knowing *why* the default changed is a strong answer.

**Fixes, best to worst:**

**1. Extract the shared logic (correct 90% of the time).** A cycle almost always means a third concept is hiding inside the pair.

```java
// Both services needed "recompute uptime for a monitor" → that's its own thing.
@Service class UptimeCalculator { ... }

@Service class MonitorService  { MonitorService(UptimeCalculator calc) { ... } }
@Service class IncidentService { IncidentService(UptimeCalculator calc) { ... } }
```

**2. Invert with an event.** If A calls B only to *notify* it, publish an event instead:

```java
@Service
class MonitorService {
    private final ApplicationEventPublisher events;
    void markDown(Monitor m) {
        events.publishEvent(new MonitorWentDownEvent(m.getId()));   // fire and forget
    }
}

@Service
class IncidentService {
    @EventListener
    void onMonitorDown(MonitorWentDownEvent e) { openIncident(e.monitorId()); }
}
```
The compile-time dependency is *gone in one direction*. This is a genuinely good architectural answer, not a workaround. (Lesson 32.)

**3. Push the dependency to the method level.** If B is only needed in one method, take it as a parameter instead of a field.

**4. `@Lazy` on one side.** Works, hides the design problem:
```java
MonitorService(@Lazy IncidentService incidents) { ... }
```

**5. `spring.main.allow-circular-references=true`.** Re-enables the old behaviour globally. **This is technical debt with a config key** — it silences the diagnostic without fixing anything. If you inherit a legacy app you may need it temporarily; say so explicitly, and say you'd track removing it.

**Self-reference is a special case:** a bean depending on itself (for the self-invocation workaround in Lesson 04) is allowed with `@Lazy`.

---

## 8. Where `@Autowired` still appears

Constructor injection removes most `@Autowired`, but you'll see it here:

```java
// On a setter, for optional deps
@Autowired(required = false) public void setMetrics(...) { }

// On an arbitrary method — Spring calls it after construction with resolved args
@Autowired
void configure(List<Validator> validators, Environment env) { ... }

// Selecting one of several constructors
@Service
class FlexibleService {
    FlexibleService() { }
    @Autowired FlexibleService(Repo repo) { }        // ← Spring uses this one
}
```

Also know the **JSR-330 equivalents**, since you'll meet them in older or Jakarta-flavoured code:

| Spring | JSR-330 (`jakarta.inject`) |
|---|---|
| `@Autowired` | `@Inject` |
| `@Qualifier("x")` | `@Named("x")` |
| `@Component` | `@Named` |
| `@Scope("prototype")` | `@Singleton` (inverse default) |

Spring supports both. `@Inject` has no `required=false` equivalent. Use Spring's annotations unless you have a portability requirement — this comes up as a trivia question and the right answer is "they're interchangeable, Spring supports JSR-330 for portability."

---

## 9. Generics-aware injection (a nice detail to know)

Spring resolves generic types, which enables a clean pattern:

```java
public interface CheckHandler<T extends CheckConfig> {
    CheckOutcome handle(T config);
}

@Component class HttpCheckHandler implements CheckHandler<HttpCheckConfig> { }
@Component class TcpCheckHandler  implements CheckHandler<TcpCheckConfig>  { }

@Service
class Runner {
    // Injects ONLY HttpCheckHandler — Spring matches the type parameter.
    Runner(CheckHandler<HttpCheckConfig> handler) { ... }
}
```

Two beans implement `CheckHandler`, but there's no ambiguity because the generic parameters differ. This works because Spring reads generic type information from the class file (generics are erased at the *usage* site but retained in class metadata). Spring Data and Spring Events lean on this heavily — it's how `@EventListener` knows which event type you want, and how `ApplicationListener<UserCreatedEvent>` only receives that event.

---

## 10. Build & break

Work in `com.vigil.playground.di2`. **These are the exact scenarios you'll be asked about.**

**1. Strategy pattern via collection injection.**
```java
public interface CheckExecutor {
    boolean supports(String type);
    String execute(String target);
}
```
Write `HttpCheckExecutor`, `TcpCheckExecutor`, `SslCheckExecutor` (each just returns a string). Write `CheckDispatcher` injecting `List<CheckExecutor>` and dispatching via `supports`. Then add a fourth executor and confirm you changed nothing else. **This is the pattern to show in an interview when asked "how would you make this extensible?"**

**2. Prove `@Order` matters.** Create three `CheckValidator` beans that print their names, inject `List<CheckValidator>`, print the order. Add `@Order` and confirm it changes. Remove the annotations and note that the order is not something you should rely on.

**3. Resolve ambiguity three ways.** Two `NotificationChannel` implementations. Inject the interface — get `NoUniqueBeanDefinitionException`. **Read the message carefully; it lists the candidates.** Then fix it:
- with `@Primary`
- with `@Qualifier("emailChannel")`
- with a **custom qualifier annotation** `@Email`
- and finally rename the constructor parameter to `emailChannel` and note that it works by name alone — then explain to yourself why you shouldn't rely on that.

**4. Manufacture and fix a circular dependency.** Two services constructor-injecting each other. Read the full error (Spring draws you an ASCII diagram — appreciate it). Then fix it **three ways**, and rank them:
- extract a third service
- replace one direction with an `ApplicationEventPublisher` + `@EventListener`
- `@Lazy`

Then try `spring.main.allow-circular-references=true` and note that the app starts. Understand precisely what you've hidden.

**5. Field-injection failure mode.** Write a service with `@Autowired` fields. Then write a test that does `new TheService()` and calls a method. Watch the NPE. Now convert to constructor injection and note that the equivalent mistake **won't compile**.

### Explain out loud (60s)

1. All four reasons for constructor injection, without repeating yourself.
2. The resolution algorithm when two beans of one type exist, in order.
3. Why `@InjectMocks` can leave a dependency null, and why a constructor can't.
4. The collection-injection strategy pattern, and what it buys you in maintenance.
5. Why Boot 2.6 banned circular references — mention proxies.

---

## 11. Production rules

| Do / Never | Why |
|---|---|
| **Constructor injection for every business bean; never field injection in a service** | Only constructors give you `final` fields, and `final` is what safely publishes a singleton's dependencies to all 200 request threads |
| **Don't add `@Autowired` to a single constructor** | Spring has inferred it since 4.3, so it is noise that implies the annotation is doing something |
| **Never rely on parameter-name matching to select a bean** | It is real (step 3 of resolution) and it is fragile: a rename silently rewires, and names only survive compilation with `-parameters` |
| **Prefer a custom qualifier annotation (`@Slack`) over `@Qualifier("slack")`** | A string typo compiles and fails at startup; an annotation is compiler-checked and survives refactoring |
| **`@Primary` on the bean for the obvious default, `@Qualifier` at the injection point for the exception** | They are asymmetric on purpose — one declares a default, the other overrides it, and the injection point always wins |
| **Inject `List<T>` for strategy and plugin sets** | Adding an implementation then requires no change to the consumer — Open/Closed satisfied by wiring rather than by an `if/else` chain |
| **Declare `@Order` whenever an injected list is a pipeline** | Without it the order follows scan order, so an unrelated refactor can silently reorder your validators |
| **Never set `spring.main.allow-circular-references=true`** | It restores a mechanism that could inject a *pre-proxy* reference, silently disabling `@Transactional` on the injected bean |
| **Fix a cycle by extracting a third bean or publishing an event, not with `@Lazy`** | A cycle nearly always means a hidden third concept; `@Lazy` keeps the design and costs you fail-fast startup validation |
| **Use `ObjectProvider<T>` for optional dependencies, not `@Autowired(required = false)`** | It gives you `getIfAvailable`, `getIfUnique` and deferred lookup instead of a field that might be null |
| **Construct the service explicitly in tests; avoid `@InjectMocks`** | A dependency you forgot to mock is left null and NPEs at runtime — with a constructor, the same mistake fails to compile |
| **With `@RequiredArgsConstructor`, never leave two same-typed `final` fields adjacent and unqualified** | Parameters are generated in field-declaration order, so reordering fields silently swaps the injected values |

---

## 12. Interview traps

**Q1. Constructor vs field injection — why constructor?**
Four independent reasons: (1) `final` fields → immutability and safe cross-thread publication; (2) the object can never exist in a partially initialized state; (3) testable with plain `new`, no reflection or Mockito needed, and adding a dependency breaks test *compilation* rather than NPE-ing at runtime; (4) an over-long parameter list is a visible design signal that field injection hides. Bonus: cycles fail at startup instead of silently working.

**Q2. Is `@Autowired` required on constructors?**
Not since Spring 4.3, when there's exactly one constructor. With multiple constructors you must mark one.

**Q3. How does Spring pick between multiple candidates of the same type?**
`@Qualifier` at the injection point → then `@Primary` on a candidate → then parameter/field **name** matching a bean name → then `@Priority`. Otherwise `NoUniqueBeanDefinitionException`.

**Q4. `@Primary` vs `@Qualifier`?**
`@Primary` sits on the bean and declares a default. `@Qualifier` sits at the injection point and names a specific bean. `@Qualifier` wins. Prefer a custom qualifier *annotation* over a string for type safety.

**Q5. What happens if you inject `List<SomeInterface>`?**
You get every bean implementing it, ordered by `@Order`/`Ordered` (arbitrary otherwise). It's the idiomatic way to implement strategy/plugin patterns — new implementations require no change to the consumer.

**Q6. What is a circular dependency, how does Spring react, and how do you fix it properly?**
Two or more beans requiring each other. Since Boot 2.6 it fails at startup by default. Best fix: extract the shared responsibility into a third bean, or invert one direction with an application event. `@Lazy` and `allow-circular-references=true` are band-aids — the latter is a config-key-shaped piece of technical debt.

**Q7. Why did Spring Boot 2.6 disable circular references by default?**
Because the old resolution used partially constructed "early references," which could inject a **pre-proxy** instance — silently disabling `@Transactional`/`@Async` on the injected bean — and made initialization order fragile and hard to reason about.

**Q8. What's `ObjectProvider` for?**
Deferred, flexible dependency lookup: `getIfAvailable(default)`, `getIfUnique()`, `orderedStream()`. Useful for optional dependencies, avoiding eager creation, and dodging ordering/cycle problems. Preferred over `@Autowired(required=false)` in modern code.

**Q9. What does `@Lazy` do at an injection point, and what does it cost?**
Injects a proxy; the real bean is created on first method invocation. Costs: you lose fail-fast startup validation, and you inherit proxy limitations. Acceptable for genuinely expensive beans; a smell when used to hide a cycle.

**Q10. Can you inject by name? Should you?**
Yes — parameter/field name matching a bean name is step 3 of resolution, and `@Resource(name=...)` does it explicitly. You shouldn't rely on it: names are strings, renames break wiring silently, and parameter names require `-parameters` compilation to be retained.

**Q11. `@Resource` vs `@Autowired` vs `@Inject`?**
`@Autowired` (Spring): by **type**, then narrowed. `@Resource` (Jakarta): by **name** first, then type. `@Inject` (JSR-330): behaves like `@Autowired` but has no `required=false`. Interviewers ask this to see whether you know `@Resource` is name-first.

---

## 13. Retention check

1. Give all four reasons for constructor injection, without repeating yourself.
2. Walk the resolution algorithm for two candidate beans of the same type.
3. Why can `@InjectMocks` leave a dependency null, and why doesn't constructor injection have that failure mode?
4. Show the collection-injection strategy pattern, and say what it buys you in maintenance terms.
5. Explain why Boot 2.6 banned circular references, mentioning proxies.
6. When is `@Lazy` legitimate, and what do you give up?

---

**Next:** [Lesson 08 — Scopes, lifecycle & conditional beans](08-scopes-lifecycle-conditional.md) — how many of a bean exist, when it's born and dies, and how to make it exist only under certain conditions.

# Lesson 02 — Java essentials for Spring

> **Why this lesson exists:** You're coming from another language, so you don't need "what is a loop." You need the ~15 Java-specific ideas that Spring code is *made of*, and you need to know which Java habits Spring will punish you for. That's what this is.

**Time:** ~70 minutes · **Prereq:** Lesson 01

---

## 1. The shape of a Java file

```java
package com.vigil.monitor;                    // 1. must match the folder path

import java.time.Instant;                     // 2. explicit imports, no wildcards in practice
import java.util.List;

public class Monitor {                        // 3. public class name MUST match file name

    private final String url;                 // 4. fields
    private Instant lastCheckedAt;

    public Monitor(String url) {              // 5. constructor
        this.url = url;
    }

    public String getUrl() {                  // 6. methods
        return url;
    }
}
```

Rules that are *enforced*, not conventions:
- **`package` must mirror the directory structure.** `com.vigil.monitor` → `src/main/java/com/vigil/monitor/`. Get this wrong and it won't compile.
- **One public class per file, name matches file.** `Monitor.java` contains `public class Monitor`.
- **No top-level code.** Everything lives in a class. Execution starts at `public static void main(String[] args)`.

Conventions that are *universal* (violating them marks you as an outsider):

| Thing | Convention | Example |
|---|---|---|
| Package | all lowercase, reverse domain | `com.vigil.monitor` |
| Class / Interface / Enum / Record | `PascalCase`, **noun** | `MonitorService` |
| Method | `camelCase`, **verb** | `findByStatus` |
| Constant | `SCREAMING_SNAKE_CASE` | `MAX_RETRIES` |
| Variable | `camelCase` | `lastCheckedAt` |
| Generic type param | single capital | `T`, `E`, `K`, `V`, `R` |

---

## 2. Types, and the reference/primitive split

Java is **statically and strongly typed**. Every variable's type is fixed at compile time.

### Primitives vs wrappers — a distinction that will bite you

```java
int count = 0;              // primitive: a raw value, cannot be null, lives cheaply
Integer boxedCount = 0;     // object: a reference, CAN be null, allocated on heap
```

| Primitive | Wrapper | Notes |
|---|---|---|
| `int` | `Integer` | 32-bit |
| `long` | `Long` | 64-bit — IDs, timestamps, epoch millis |
| `double` | `Double` | **never for money** |
| `boolean` | `Boolean` | |
| `char` | `Character` | |
| `byte`, `short`, `float` | ... | rare in app code |

**Why both exist:** generics and collections can only hold *objects*. `List<int>` is illegal; `List<Integer>` is fine. Java auto-converts between them ("autoboxing").

**Where this bites you in Spring — memorize these three:**

```java
// 1. NULLABLE DATABASE COLUMNS.
@Entity
class Monitor {
    private int timeoutMs;      // ❌ if the column is NULL, Hibernate throws or silently gives 0
    private Integer timeoutMs;  // ✅ null means "not set" and you can tell the difference
}
```

```java
// 2. THE == TRAP.
Integer a = 1000, b = 1000;
a == b;            // false!  comparing references, not values
a.equals(b);       // true
// (a == b is true for values -128..127 because the JVM caches those Integer objects.
//  This makes the bug intermittent and therefore evil.)
// Rule: == for primitives and enums. .equals() for EVERYTHING else.
```

```java
// 3. UNBOXING NULL = NPE.
Integer maybeNull = repo.countSomething();   // returns null
int x = maybeNull;                            // 💥 NullPointerException, on a line with no dot
```

### Money: use `BigDecimal`, always

```java
System.out.println(0.1 + 0.2);   // 0.30000000000000004
```

Floating point can't represent 0.1. Use `BigDecimal` for money, and construct it **from a String**:

```java
new BigDecimal("0.1").add(new BigDecimal("0.2"));   // exactly 0.3
new BigDecimal(0.1);                                 // ❌ 0.1000000000000000055511151231257827...
```

`BigDecimal` is immutable — `a.add(b)` returns a new object, it does not mutate `a`. And compare with `compareTo() == 0`, not `equals()`, because `equals()` considers scale (`2.0` ≠ `2.00`).

> **Interview trap:** "How do you store money?" — `BigDecimal` in Java, `NUMERIC/DECIMAL` in Postgres. A senior answer adds: "or store minor units as a `long` (cents) if you control all arithmetic — some systems prefer that for performance and exactness, but you must handle currency exponents." Saying `double` ends that line of questioning badly.

### Dates: `java.time` only

```java
Instant now      = Instant.now();                 // a point on the UTC timeline — USE THIS by default
LocalDate  d     = LocalDate.of(2026, 7, 30);     // a date, NO time, NO zone (birthdays)
LocalDateTime dt = LocalDateTime.now();           // date+time, NO zone — ambiguous, avoid for events
ZonedDateTime z  = ZonedDateTime.now(ZoneId.of("Asia/Kolkata"));  // for display to a human
Duration dur     = Duration.ofSeconds(30);        // machine time span
Period   p       = Period.ofMonths(1);            // human time span (calendar-aware)
```

**Rule: store `Instant` (UTC), convert to a zone only at the presentation edge.** All `java.time` types are immutable and thread-safe.

`Date`, `Calendar` and `SimpleDateFormat` are the old API. They are mutable, `SimpleDateFormat` is **not thread-safe** (a genuinely famous production bug — a shared static formatter in a web app produces garbage dates under concurrency), and months were 0-indexed. You'll see them in legacy code. Never write new code with them.

> **Fun fact:** `java.time` was designed by Stephen Colebourne, who first built it as the Joda-Time library because `java.util.Date` was so bad. Joda-Time became so ubiquitous that Java absorbed a refined version of it as JSR-310 in Java 8. Then Joda-Time's own docs started telling people to migrate off it. The best possible outcome for a library.

---

## 3. Classes, interfaces, and why Spring cares so much about interfaces

### `final`, `static`, and access modifiers

```java
public class MonitorService {
    private static final int MAX_RETRIES = 3;   // constant: one per class, immutable
    private final MonitorRepository repo;       // instance field, assigned once in constructor

    public MonitorService(MonitorRepository repo) { this.repo = repo; }
}
```

- `final` field → must be assigned exactly once (at declaration or in the constructor). This is your main tool for **immutability**, which matters enormously because Spring beans are shared across threads.
- `static` → belongs to the class, not an instance. **`static` mutable state in a Spring app is a bug** — it's shared across all requests and threads.
- Access: `private` (class only) → *package-private* (no keyword — same package) → `protected` (+ subclasses) → `public`. **Default to `private`.** Package-private is underused and great for keeping implementation classes invisible outside a feature package.

### Interfaces: the contract

```java
public interface NotificationChannel {
    void send(Notification n);                       // abstract: implementers must provide

    default boolean supportsRichText() {             // default: implementers MAY override
        return false;
    }

    static NotificationChannel noop() {              // static: utility on the interface
        return n -> {};
    }
}
```

```java
public class SlackChannel implements NotificationChannel {
    @Override
    public void send(Notification n) { /* ... */ }
}
```

**Why this is the single most important OOP concept for Spring:**

```java
// Your service depends on the ABSTRACTION:
class IncidentService {
    private final NotificationChannel channel;
    IncidentService(NotificationChannel channel) { this.channel = channel; }
}
```

Now `IncidentService` works with Slack, email, webhooks, or a fake in tests — **without changing a line**. That's the Dependency Inversion Principle, and it's the entire reason Spring's DI container is useful. Every good Spring codebase is a graph of small classes talking through interfaces.

> **`default` methods** were added in Java 8 to let interfaces evolve without breaking every implementer. Spring's own APIs are full of them — that's why you can implement a huge-looking Spring interface and only override one method.

### Abstract classes vs interfaces

| | Interface | Abstract class |
|---|---|---|
| Multiple inheritance | ✅ implement many | ❌ extend only one |
| State (fields) | ❌ (only `static final` constants) | ✅ |
| Constructors | ❌ | ✅ |
| Use for | **Capability / contract** | Shared partial implementation |

**Default to interfaces.** Reach for an abstract class only when several implementations genuinely share state and logic (e.g. a `AbstractHttpCheck` holding a shared HTTP client and timeout handling, with subclasses differing only in how they validate the response).

### Records — your DTO workhorse (Java 16+)

```java
public record MonitorResponse(
        UUID id,
        String name,
        String url,
        MonitorStatus status,
        Instant lastCheckedAt
) {}
```

That one declaration gives you: private final fields, a canonical constructor, accessors (`id()`, not `getId()`), `equals`, `hashCode`, and `toString`. Immutable by construction.

You can add validation and extra members:

```java
public record CreateMonitorRequest(String name, String url, int intervalSeconds) {
    public CreateMonitorRequest {                       // compact constructor
        if (intervalSeconds < 30) {
            throw new IllegalArgumentException("interval must be >= 30s");
        }
    }
    public Duration interval() { return Duration.ofSeconds(intervalSeconds); }
}
```

**Records are the correct default for DTOs, API request/response bodies, and value objects.** Jackson (JSON) supports them natively; Spring's `@ConfigurationProperties` supports them; Bean Validation annotations work on their components.

**Do NOT use records for JPA entities.** JPA requires a no-arg constructor and mutability for its proxying and dirty-checking. This is a genuinely good interview question — see Lesson 21.

### Enums are real classes

```java
public enum MonitorStatus {
    UP("Operational", "#22c55e"),
    DEGRADED("Degraded", "#f59e0b"),
    DOWN("Down", "#ef4444"),
    PAUSED("Paused", "#6b7280");

    private final String label;
    private final String colour;

    MonitorStatus(String label, String colour) {   // constructor is implicitly private
        this.label = label;
        this.colour = colour;
    }

    public String label()  { return label; }
    public String colour() { return colour; }
    public boolean isFailing() { return this == DOWN || this == DEGRADED; }
}
```

Java enums can hold state, implement interfaces, and have methods. Use them for any fixed set of options — and use `==` to compare them (they're singletons).

> **Real production wisdom:** when persisting enums, always `@Enumerated(EnumType.STRING)`, never the default `ORDINAL`. `ORDINAL` stores the *index*, so inserting a new value in the middle silently corrupts every existing row. This has caused real outages. Lesson 21 revisits it.

---

## 4. Collections — the 5 you need

```java
List<Monitor>          list = new ArrayList<>();       // ordered, duplicates OK, index access
Set<String>            set  = new HashSet<>();         // unique, unordered
Map<UUID, Monitor>     map  = new HashMap<>();         // key → value
Queue<CheckJob>        q    = new ArrayDeque<>();      // FIFO
Optional<Monitor>      opt  = repo.findById(id);       // 0-or-1 (Lesson 03)
```

| Interface | Default impl | When you'd choose another |
|---|---|---|
| `List` | `ArrayList` | `LinkedList` — almost never; `ArrayList` wins in practice due to cache locality |
| `Set` | `HashSet` | `LinkedHashSet` (insertion order), `TreeSet` (sorted), `EnumSet` (enums, very fast) |
| `Map` | `HashMap` | `LinkedHashMap` (order/LRU), `TreeMap` (sorted), `ConcurrentHashMap` (**thread-safe**) |

**Program to the interface:** `List<Monitor> x = new ArrayList<>();`, never `ArrayList<Monitor> x = ...`. Return `List`, accept `Collection`. Spring's whole API style follows this.

### Creating collections

```java
List<String> a = List.of("a", "b");          // IMMUTABLE (Java 9+) — throws on .add()
List<String> b = new ArrayList<>(List.of("a", "b"));  // mutable copy
Map<String,Integer> m = Map.of("a", 1, "b", 2);        // immutable, max 10 pairs
List<String> empty = List.of();
```

`List.of(...)` is the modern default. Just remember it's **immutable** — the `UnsupportedOperationException` from calling `.add()` on it is a rite of passage. Also: `List.of()` rejects `null` elements, which is usually a feature.

### `equals` and `hashCode` — the contract you must not break

`HashMap` and `HashSet` find objects by hash first, then confirm with `equals`. So:

**The contract:** if `a.equals(b)` then `a.hashCode() == b.hashCode()`. Always override both together, or neither.

```java
public class Monitor {
    private final UUID id;

    @Override public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Monitor other)) return false;   // pattern matching, Java 16+
        return Objects.equals(id, other.id);
    }
    @Override public int hashCode() { return Objects.hash(id); }
}
```

**The production bug this prevents:** put a mutable object in a `HashSet`, then change a field used in `hashCode()`. The object is now in the wrong bucket. `set.contains(obj)` returns `false` for an object that is literally inside the set. Records handle `equals`/`hashCode` for you — one more reason to prefer them for value objects. JPA entities have a *special* rule here that trips up nearly everyone (Lesson 21).

---

## 5. Exceptions — and Spring's strong opinion about them

```java
// CHECKED — extends Exception. Compiler forces you to catch or declare.
public class MonitorConfigException extends Exception { }

// UNCHECKED — extends RuntimeException. Compiler ignores it.
public class MonitorNotFoundException extends RuntimeException {
    public MonitorNotFoundException(UUID id) {
        super("Monitor not found: " + id);
    }
}
```

```java
try {
    check.execute();
} catch (IOException | TimeoutException e) {          // multi-catch
    log.warn("check failed for {}", monitor.getId(), e);
    throw new CheckFailedException("check failed", e);  // ALWAYS keep the cause
} finally {
    // always runs
}
```

```java
// try-with-resources: auto-closes anything implementing AutoCloseable
try (var conn = dataSource.getConnection()) {
    // ...
}   // conn.close() guaranteed, even on exception
```

### The Spring convention: prefer unchecked exceptions

Spring's entire design philosophy is **unchecked exceptions for unrecoverable conditions**. `DataAccessException`, `BeansException`, `RestClientException` — all unchecked.

**Why?** Checked exceptions force every caller in the chain to either handle a problem it can't fix or pollute its signature with `throws`. The result in practice is thousands of `catch (SQLException e) { throw new RuntimeException(e); }` blocks — noise with no benefit. Spring wraps JDBC's checked `SQLException` into an unchecked, *portable* `DataAccessException` hierarchy so your code can catch `DuplicateKeyException` regardless of whether you're on Postgres or Oracle. That translation is a genuinely great piece of API design and worth citing in an interview.

**And a consequence you must know:** `@Transactional` rolls back on **unchecked** exceptions by default, and **does not roll back on checked exceptions.** This surprises everyone once. (Lesson 23.)

### Rules for exception hygiene

```java
catch (Exception e) { }                       // ❌ swallowing — the worst thing in this file
catch (Exception e) { e.printStackTrace(); }  // ❌ not a log; goes to stdout unstructured
catch (Exception e) { log.error("failed: " + e.getMessage()); }  // ❌ loses the stack trace
catch (Exception e) { log.error("check failed for {}", id, e); } // ✅ message + context + cause
```

Also: **never use exceptions for control flow.** `catch (NotFoundException e) { return createNew(); }` is slow (stack capture is expensive) and hides intent.

---

## 6. `null` — Java's original sin, and how Spring copes

Tony Hoare, who invented the null reference in 1965, called it his "billion-dollar mistake." He was being conservative.

Your defences, in order of preference:

**1. Don't produce nulls.** Return `Optional<T>` (Lesson 03), empty collections instead of null, throw for genuinely exceptional absence.

**2. Fail fast at the boundary.**
```java
public MonitorService(MonitorRepository repo) {
    this.repo = Objects.requireNonNull(repo, "repo");
}
```

**3. Annotate intent.** Spring uses `@Nullable` / `@NonNull` (from `org.springframework.lang`, or JSpecify in newer versions) so your IDE and static analysis can warn you. Spring packages are `@NonNullApi` by default — meaning in Spring's own API, a parameter is non-null unless marked `@Nullable`. That's why IntelliJ can warn you about passing null into Spring methods.

**4. Safe idioms.**
```java
Objects.equals(a, b);                    // null-safe equality
Objects.requireNonNullElse(x, fallback); // default value
if ("UP".equals(status))                 // ✅ constant first — no NPE if status is null
if (status.equals("UP"))                 // ❌ NPE if status is null
```

> **Boot 4 delta:** Spring Framework 7 migrated to **JSpecify** annotations (`org.jspecify.annotations.Nullable`) for null-safety, which tools like NullAway and IntelliJ understand better. Same idea, standardized vocabulary.

---

## 7. Threads: the minimum you must know now

You will not write `new Thread()` in Spring. But you *must* understand this, because it determines correctness:

**A Spring bean is a singleton shared by every concurrent request.**

Your web server has a pool of threads (Tomcat: 200 by default). Ten users hitting your endpoint = up to ten threads executing the *same instance* of your `MonitorService`, simultaneously.

```java
@Service
public class BadService {
    private int requestCount = 0;              // ❌ SHARED MUTABLE STATE
    private String currentUser;                // ❌ CATASTROPHIC — user A sees user B's data

    public void handle(String user) {
        this.currentUser = user;               // race condition
        requestCount++;                        // not atomic: read, add, write
    }
}
```

```java
@Service
public class GoodService {
    private final MonitorRepository repo;      // ✅ final, immutable reference
    private final AtomicLong requestCount = new AtomicLong();  // ✅ thread-safe if you need a counter

    public void handle(String user) {          // ✅ 'user' is a local/parameter → per-thread stack
        requestCount.incrementAndGet();
    }
}
```

**The rule, and it's simple: Spring beans hold only immutable dependencies. All request-specific data lives in method parameters and local variables.** Local variables live on the thread's own stack, so they are automatically safe.

Getting this wrong produces the worst class of bug there is: works perfectly in dev, corrupts data in production under load, and can't be reproduced.

> **Fun fact:** `requestCount++` compiles to three bytecode operations (read, increment, write). Two threads can both read `5`, both write `6`, and you've lost an increment. This is why `AtomicLong` exists — it uses a CPU-level compare-and-swap instruction. And why `SimpleDateFormat`'s shared internal `Calendar` field made it a legendary source of corrupted dates.

Lesson 32 and 36 go deeper — including virtual threads, which change the economics but not this rule.

---

## 8. Build this

In your `vigil` project, create `src/main/java/com/vigil/playground/` and write real code. Don't skip this — typing it is the point.

**1. An enum with behaviour** — `MonitorStatus` exactly as in §3, with `label()`, `colour()`, `isFailing()`.

**2. An interface + two implementations:**
```java
public interface NotificationChannel {
    void send(String subject, String body);
    default String name() { return getClass().getSimpleName(); }
}
```
Write `ConsoleChannel` (prints) and `SlackChannel` (also prints, but pretends). Note: you wrote a *polymorphic* system with no `if/else`.

**3. A record DTO with validation** — `CreateMonitorRequest(String name, String url, int intervalSeconds)` with a compact constructor rejecting intervals under 30 and blank names.

**4. A class with `equals`/`hashCode`** — a `Monitor` keyed on `id`. Then:
```java
Set<Monitor> set = new HashSet<>();
Monitor m = new Monitor(id, "API");
set.add(m);
System.out.println(set.contains(new Monitor(id, "Different name")));  // predict, then run
```

**5. Break it on purpose — feel the traps:**
```java
Integer a = 127, b = 127;    System.out.println(a == b);   // predict!
Integer c = 128, d = 128;    System.out.println(c == d);   // predict!
System.out.println(0.1 + 0.2 == 0.3);
Integer nullable = null;     int boom = nullable;          // read the exception carefully
List<String> immutable = List.of("a");  immutable.add("b"); // read this exception too
```
Predict each result *before* running. Wrong predictions are the valuable ones.

---

## 9. Interview traps

**Q1. Why should Spring beans be stateless?**
Beans are singletons by default; every concurrent request shares the instance. Mutable instance fields become shared mutable state across Tomcat's thread pool → race conditions and cross-request data leakage. Keep dependencies `final` and request data in locals/parameters.

**Q2. `==` vs `.equals()`?**
`==` compares references for objects (identity), values for primitives. `.equals()` compares logical equality as defined by the class. Enums are the exception where `==` is preferred (singletons, null-safe). The `Integer` cache (-128..127) makes `==` on boxed integers *intermittently* correct, which is worse than always wrong.

**Q3. The `equals`/`hashCode` contract, and what breaks if you violate it?**
Equal objects must have equal hash codes. Break it and hash-based collections lose your objects: `contains()` returns false for a contained element, `Map` lookups miss, duplicates appear in `Set`. Unequal objects *may* share a hash code (collision) — that's legal, just slower.

**Q4. Checked vs unchecked, and why does Spring prefer unchecked?**
Checked = compiler-enforced handling; unchecked = not. Spring prefers unchecked because most infrastructure failures aren't locally recoverable, so forcing `throws`/`catch` up the stack adds noise without safety. Spring translates JDBC's `SQLException` into a portable unchecked `DataAccessException` hierarchy. **And:** `@Transactional` only rolls back on unchecked by default.

**Q5. Interface vs abstract class — which and why?**
Interface for capability/contract, multiple inheritance, and the default in DI-based design. Abstract class when implementations share real state and logic. Prefer interfaces: they keep classes decoupled and make testing/mocking trivial.

**Q6. Why records for DTOs but not JPA entities?**
Records are final and immutable with no no-arg constructor. JPA requires a non-final class, a no-arg constructor, and mutable fields for proxy-based lazy loading and dirty checking. Records are perfect for DTOs (immutable, `equals`/`hashCode` free, Jackson-supported).

**Q7. Why is `BigDecimal` required for money, and what's the constructor trap?**
Binary floating point can't represent decimal fractions like 0.1, so `double` accumulates error. `new BigDecimal(0.1)` inherits the double's error — always use the `String` constructor or `BigDecimal.valueOf(double)`. Compare with `compareTo`, since `equals` distinguishes `2.0` from `2.00`.

**Q8. `Instant` vs `LocalDateTime` — which for a "created at" column?**
`Instant`. It's an unambiguous point on the UTC timeline. `LocalDateTime` has no zone, so it's ambiguous across servers and broken by DST. Store UTC, convert at the presentation edge.

**Q9. Why is `SimpleDateFormat` dangerous?**
Mutable and not thread-safe. Shared as a `static` field in a web app, concurrent threads corrupt its internal state, producing wrong dates or exceptions — intermittently. `java.time`'s `DateTimeFormatter` is immutable and thread-safe.

---

## 10. Retention check

1. Why does `List<int>` not compile, and what do you write instead?
2. You have a Spring `@Service` with a `private String currentUserEmail` field set at the top of each method. What exactly goes wrong, and under what conditions?
3. Give two reasons a record can't be a JPA entity.
4. Why does `if (status.equals("UP"))` risk an NPE when `if ("UP".equals(status))` doesn't?
5. What does Spring do to `SQLException`, and why is that good design?

---

**Next:** [Lesson 03 — Modern Java: lambdas, streams, Optional](03-java-modern-features.md) — the functional style that Spring's modern APIs are entirely built from.

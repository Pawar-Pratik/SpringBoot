# Lesson 03 — Modern Java: lambdas, streams, Optional

> **Why this lesson exists:** Modern Spring APIs are *functional*. `SecurityFilterChain` config, `RestClient`, `Optional`-returning repositories, `CompletableFuture`, Resilience4j, and reactive streams are all built on lambdas. If lambdas feel foreign, all modern Spring config looks like line noise. After this lesson it will read like prose.

**Time:** ~60 minutes · **Prereq:** Lesson 02

---

## 1. The idea in one sentence

> **A lambda is nothing but an object implementing a one-method interface — so every "magical" modern Spring API is an ordinary method taking an ordinary argument, and you can read any of them by asking "which single-method interface does this lambda implement?"**

That question dissolves Spring Security's `http.authorizeHttpRequests(auth -> ...)`, `RestClient`'s fluent chain, `orElseGet`, `@Async` and Resilience4j in one move. Streams and `Optional` are then just two libraries built on top of it — one for pipelines, one for absence.

---

## 2. Lambdas: passing behaviour as a value

Before Java 8, if you wanted to pass *code* to a method, you needed a whole class:

```java
// The old way — an anonymous inner class
button.addListener(new ClickListener() {
    @Override
    public void onClick(Event e) {
        System.out.println("clicked");
    }
});
```

Six lines of ceremony for one line of intent. A lambda is the same thing with the ceremony deleted:

```java
button.addListener(e -> System.out.println("clicked"));
```

### Syntax, all of it

```java
() -> doSomething()                      // no args
x  -> x * 2                              // one arg, parens optional, return implicit
(x, y) -> x + y                          // multiple args
(String s) -> s.length()                 // explicit type (rarely needed)
x -> {                                   // block body: braces require explicit return
    var trimmed = x.trim();
    return trimmed.toUpperCase();
}
```

### The crucial insight: a lambda IS an object implementing an interface

This is the part that unlocks everything. A lambda is not a new kind of thing — it's shorthand for implementing a **functional interface** (an interface with exactly one abstract method).

```java
@FunctionalInterface                 // optional annotation; makes the compiler enforce "exactly one"
interface Check {
    CheckResult run(Monitor m);
}

Check httpCheck = m -> new CheckResult(m.getId(), 200, Duration.ofMillis(120));
```

`httpCheck` is an object of a type implementing `Check`. The compiler infers which interface from context ("target typing"). **That means anywhere a Spring API takes a single-method interface, you can pass a lambda.** Which is why this:

```java
http.authorizeHttpRequests(auth -> auth
        .requestMatchers("/public/**").permitAll()
        .anyRequest().authenticated());
```

...is not special syntax. It's a lambda implementing a `Customizer<AuthorizeHttpRequestsConfigurer<...>>`, whose one method receives the configurer so you can mutate it. Once you see that, Spring Security config stops being scary.

### The built-in functional interfaces (`java.util.function`)

You need to recognize these five instantly; they appear all over Spring:

| Interface | Signature | Meaning | Spring example |
|---|---|---|---|
| `Function<T,R>` | `R apply(T)` | transform | `.map(...)` |
| `Consumer<T>` | `void accept(T)` | side effect | `.forEach(...)`, `Customizer` |
| `Supplier<T>` | `T get()` | lazy produce | `orElseGet(...)`, `@Bean` factories |
| `Predicate<T>` | `boolean test(T)` | test | `.filter(...)` |
| `BiFunction<T,U,R>` | `R apply(T,U)` | two-arg transform | `.reduce(...)` |

Plus `UnaryOperator<T>` (a `Function<T,T>`) and `Runnable` (`void run()` — used by `@Async`, executors, `CompletableFuture`).

**`Supplier` deserves a spotlight**, because *laziness* is a real production concern:

```java
// Bad: buildExpensiveDefault() runs even when the value IS present
monitor.orElse(buildExpensiveDefault());

// Good: the lambda only runs if the Optional is empty
monitor.orElseGet(() -> buildExpensiveDefault());
```

Same pattern in logging:
```java
log.debug("state: " + expensiveToString());   // ❌ string built even when DEBUG is off
log.debug("state: {}", monitor);              // ✅ SLF4J defers formatting
```

### Method references — lambda shorthand

```java
monitors.forEach(m -> System.out.println(m));   // lambda
monitors.forEach(System.out::println);          // method reference — identical meaning

names.stream().map(s -> s.toUpperCase())        // lambda
names.stream().map(String::toUpperCase)         // reference to an INSTANCE method of the arg

list.stream().map(m -> new Dto(m))              // lambda
list.stream().map(Dto::new)                     // CONSTRUCTOR reference

repo.findById(id).orElseThrow(() -> new NotFoundException(id))  // needs the arg, so lambda
```

Four forms: `Type::staticMethod`, `instance::method`, `Type::instanceMethod`, `Type::new`. Use them when they read better; don't contort code to use them.

### The `final`/effectively-final rule

```java
int total = 0;
list.forEach(x -> total += x);    // ❌ won't compile
```

A lambda can only reference local variables that are **final or effectively final** (never reassigned after initialization). Why: the lambda may outlive the method call (e.g. run on another thread), so Java captures the *value*, not the variable. Allowing mutation would create confusing, unsafe semantics.

Instance fields *can* be mutated from a lambda — which is exactly how you accidentally introduce a race condition. Use streams' `collect`/`reduce` to accumulate instead.

---

## 3. Streams: declarative data pipelines

A stream is **not** a collection. It's a *pipeline description*: a source, zero or more lazy intermediate operations, and one terminal operation that actually runs everything.

```java
List<String> failingUrls = monitors.stream()          // source
        .filter(m -> m.getStatus().isFailing())       // intermediate (lazy)
        .map(Monitor::getUrl)                         // intermediate (lazy)
        .sorted()                                     // intermediate (lazy)
        .toList();                                    // TERMINAL — now it runs
```

**Nothing executes until the terminal operation.** And elements flow through *one at a time*, not stage-by-stage — so `.filter().findFirst()` on a million-element list may only touch three elements. That laziness is the whole point.

### The operations that matter

**Intermediate (return a Stream):**
```java
.filter(Predicate)              // keep matching
.map(Function)                  // transform each
.flatMap(Function)              // transform each into a stream, then flatten (1→many)
.distinct()                     // dedupe (uses equals/hashCode!)
.sorted() / .sorted(Comparator)
.limit(n) / .skip(n)
.peek(Consumer)                 // debugging only — never for side effects
```

**Terminal:**
```java
.toList()                       // Java 16+, immutable — your default
.collect(Collectors.toList())   // older, mutable
.collect(Collectors.toSet())
.collect(Collectors.groupingBy(Monitor::getStatus))              // Map<Status, List<Monitor>>
.collect(Collectors.toMap(Monitor::getId, Function.identity()))  // Map<UUID, Monitor>
.collect(Collectors.joining(", ", "[", "]"))
.forEach(Consumer)
.count()
.anyMatch(p) / .allMatch(p) / .noneMatch(p)
.findFirst() / .findAny()       // → Optional
.min(cmp) / .max(cmp)           // → Optional
.reduce(identity, accumulator)
.mapToInt(...).sum() / .average()
```

### The patterns you'll write constantly in Spring

```java
// 1. Entities → DTOs. You will write this in every project, forever.
List<MonitorResponse> dtos = monitors.stream()
        .map(monitorMapper::toResponse)
        .toList();

// 2. Group for a dashboard
Map<MonitorStatus, Long> counts = monitors.stream()
        .collect(Collectors.groupingBy(Monitor::getStatus, Collectors.counting()));

// 3. Index by id for O(1) lookup (avoids an N+1 query — see Lesson 24)
Map<UUID, Monitor> byId = monitors.stream()
        .collect(Collectors.toMap(Monitor::getId, Function.identity()));

// 4. flatMap: one org has many monitors, and you want all monitors across orgs
List<Monitor> all = orgs.stream()
        .flatMap(org -> org.getMonitors().stream())
        .toList();

// 5. Multi-key sorting
monitors.sort(Comparator.comparing(Monitor::getStatus)
                        .thenComparing(Monitor::getName));

// 6. Aggregate a numeric field
double avgMs = results.stream()
        .mapToLong(CheckResult::getResponseTimeMs)
        .average()
        .orElse(0.0);
```

### Where streams go wrong — read this part twice

**1. Don't use a stream where a loop is clearer.**
```java
// ❌ clever
IntStream.range(0, 5).forEach(i -> doThing(i));
// ✅ obvious
for (int i = 0; i < 5; i++) doThing(i);
```
Streams shine at *transformation pipelines*. For simple iteration with side effects, a `for` loop is more readable and easier to debug (you can set a breakpoint and step).

**2. Never use a stream to hide a database query.** This is the #1 real-world stream disaster:
```java
// ❌ 1 query for orgs, then N queries as the lazy collection loads per org.
//    Looks elegant. Executes 1,001 queries.
orgs.stream().flatMap(o -> o.getMonitors().stream()).toList();
```
That's the N+1 problem wearing a functional disguise. Fix it in the query layer (Lesson 24), not the stream.

**3. `parallelStream()` — almost always the wrong answer.**
It uses the shared `ForkJoinPool.commonPool()`, so one bad parallel stream can starve every other one in the JVM. It has real overhead (splitting, merging), it's useless for I/O-bound work, and in a web app your requests are *already* parallel across threads — you're stealing from your own throughput. Use it only for genuinely CPU-bound work on large in-memory datasets, after measuring. In interviews, "I'd measure first, and it's rarely a win in a request-handling web app" is the senior answer.

**4. Streams are single-use.** Consuming one twice throws `IllegalStateException`.

**5. Side effects inside streams break parallelism and readability.**
```java
List<String> out = new ArrayList<>();
items.stream().forEach(i -> out.add(i.name()));   // ❌ not thread-safe, and pointless
List<String> out = items.stream().map(Item::name).toList();  // ✅
```

---

## 4. `Optional` — explicit absence

`Optional<T>` is a container holding either a value or nothing. Its purpose is to make "this might not exist" **visible in the type signature** instead of a runtime surprise.

```java
Optional<Monitor> found = monitorRepository.findById(id);   // Spring Data returns Optional
```

That signature is documentation the compiler enforces: you cannot pretend absence is impossible.

### The full API you need

```java
// Creating
Optional.of(value);                  // throws if value is null
Optional.ofNullable(maybeNull);      // → empty if null
Optional.empty();

// Consuming — in order of preference
monitor.map(Monitor::getUrl)                 // transform if present → Optional<String>
       .filter(url -> url.startsWith("https"))
       .orElse("http://default");            // eager default

monitor.orElseGet(() -> loadDefault());      // lazy default
monitor.orElseThrow(() -> new MonitorNotFoundException(id));   // ← MOST COMMON IN SPRING
monitor.orElseThrow();                       // NoSuchElementException, poor message
monitor.ifPresent(m -> log.info("found {}", m.getName()));
monitor.ifPresentOrElse(this::handle, this::handleMissing);
monitor.flatMap(m -> m.getOwner());          // when the mapper itself returns Optional
monitor.stream()                             // Java 9+: 0-or-1 element stream, great in flatMap
```

**The canonical Spring service method — you'll write this hundreds of times:**

```java
public MonitorResponse getMonitor(UUID id) {
    return monitorRepository.findById(id)
            .map(mapper::toResponse)
            .orElseThrow(() -> new MonitorNotFoundException(id));
}
```

### Optional anti-patterns

```java
if (opt.isPresent()) { use(opt.get()); }     // ❌ you've reinvented a null check, with extra steps
opt.ifPresent(this::use);                     // ✅

opt.get();                                     // ❌ throws NoSuchElementException with no context
opt.orElseThrow(() -> new NotFoundException(id));  // ✅ meaningful error

// ❌ Never as a field, parameter, or in a collection:
private Optional<String> name;               // not serializable-friendly, adds allocation, pointless
void doThing(Optional<String> maybe);        // caller should just use an overload
List<Optional<Monitor>> results;             // filter the empties out instead
```

**The rule (from the Java architects themselves): `Optional` is designed for return types, to express "no result."** Everywhere else it's noise.

> **Fun fact:** Brian Goetz (Java Language Architect) has said Optional was *specifically* intended as a library return type for "no result" cases, and that treating it as a general-purpose Maybe monad was never the goal. So the widespread "never use Optional as a field" advice isn't dogma — it's the designers' stated intent. Citing this in an interview is a nice signal that you read primary sources.

---

## 5. Other modern syntax you'll see everywhere

### `var` — local type inference (Java 10+)

```java
var monitors = new ArrayList<Monitor>();               // ✅ type obvious from the right side
var response = restClient.get().retrieve().body(String.class);  // ✅
var x = service.process();                            // ❌ what type is x? unreadable
```
Locals only — not fields, not parameters, not return types. Use it when it removes redundancy, not when it removes information.

### Text blocks (Java 15+) — a gift for SQL and JSON

```java
String sql = """
        SELECT m.id, m.name, count(cr.id) AS checks
        FROM monitor m
        LEFT JOIN check_result cr ON cr.monitor_id = m.id
        WHERE m.org_id = :orgId
        GROUP BY m.id, m.name
        """;
```
No more `" + " +`. Indentation is stripped relative to the closing `"""`. You'll use these constantly for `@Query` and test JSON.

### Switch expressions & pattern matching (Java 17/21)

```java
// switch as an EXPRESSION that returns a value — no fall-through, must be exhaustive
String severity = switch (status) {
    case DOWN -> "critical";
    case DEGRADED -> "warning";
    case UP, PAUSED -> "none";
};

// pattern matching for instanceof (Java 16+)
if (exception instanceof HttpClientErrorException e && e.getStatusCode().is4xxClientError()) {
    return handleClientError(e);          // 'e' is already typed — no cast needed
}

// pattern matching in switch (Java 21) — great for exception → HTTP status mapping
String message = switch (throwable) {
    case TimeoutException t   -> "Timed out after " + t.getMessage();
    case ConnectException c   -> "Connection refused";
    case SSLException s       -> "TLS failure: " + s.getMessage();
    default                   -> "Unknown failure";
};
```

Exhaustiveness is the underrated win: with an enum or sealed type, adding a new case makes the switch **fail to compile** until you handle it. That turns a whole class of runtime bug into a compile error.

### Sealed interfaces (Java 17) — closed hierarchies

```java
public sealed interface CheckOutcome permits Success, Failure, Timeout {}
public record Success(int statusCode, long ms) implements CheckOutcome {}
public record Failure(String reason)          implements CheckOutcome {}
public record Timeout(long afterMs)           implements CheckOutcome {}
```

Now `switch` over `CheckOutcome` is provably exhaustive. Sealed interface + records + switch patterns = algebraic data types in Java. This is a genuinely elegant way to model outcomes, and Vigil's check results will use it.

---

## 6. `CompletableFuture` — a preview

You'll meet this properly in Lesson 32, but recognize the shape:

```java
CompletableFuture<CheckResult> future = CompletableFuture
        .supplyAsync(() -> httpChecker.check(monitor), executor)
        .thenApply(result -> enrich(result))
        .exceptionally(ex -> CheckResult.failed(monitor, ex));
```

It's a promise with a functional API. `@Async` methods can return one. Note it takes an explicit `executor` — defaulting to the common ForkJoinPool is a classic production mistake.

---

## 7. Build & break

In `com.vigil.playground`, build a small analytics utility over sample data. **Type it. Predict outputs before running.**

```java
record CheckResult(UUID monitorId, String monitorName, MonitorStatus status,
                   long responseTimeMs, Instant checkedAt) {}
```

Create a list of ~10 hand-written results across several monitors and statuses, then implement:

1. `List<String> failingMonitorNames(List<CheckResult> rs)` — names where status is failing, distinct, sorted.
2. `Map<MonitorStatus, Long> countByStatus(List<CheckResult> rs)` — `groupingBy` + `counting`.
3. `Map<String, Double> avgResponseTimeByMonitor(List<CheckResult> rs)` — `groupingBy` + `averagingLong`.
4. `Optional<CheckResult> slowest(List<CheckResult> rs)` — `max` with a `Comparator`. Note *why* it returns `Optional`.
5. `String summaryLine(List<CheckResult> rs)` — `"3 UP, 1 DEGRADED, 2 DOWN"` using `joining`.
6. `double p95(List<CheckResult> rs)` — sort response times, index into the 95th percentile. (You'll reuse this in Vigil for real; p95 latency is what actually matters in monitoring, because averages hide the pain.)
7. Rewrite #1 as a plain `for` loop. Compare readability honestly — decide for yourself when each wins.

**Break it on purpose:**
```java
var s = list.stream().filter(x -> true);
s.toList();
s.toList();                              // read the exception

Optional<String> empty = Optional.empty();
empty.get();                             // read the exception

// Predict: how many times does the print run?
list.stream().peek(System.out::println).filter(r -> r.responseTimeMs() > 100).findFirst();
```

That last one is the laziness lesson. It won't print all 10.

### Explain out loud (60s)

1. What a functional interface is, and why that makes Spring Security's config readable.
2. Prove streams are lazy, using `peek` and `findFirst`.
3. `map` vs `flatMap`, with a concrete 1→many example.
4. `orElse` vs `orElseGet` — the exact difference, and when it costs you.
5. The one place `Optional` belongs, and two places it doesn't.

---

## 8. Production rules

| Do / Never | Why |
|---|---|
| **`orElseGet(supplier)` whenever the default costs anything** | `orElse(expensiveCall())` evaluates its argument *even when the value is present* — the work happens on the happy path |
| **`Optional` as a return type only — never a field, parameter or collection element** | It exists to say "no result" to a caller; anywhere else it adds allocation and breaks serialization and JPA |
| **`orElseThrow(() -> new DomainException(id))`, never `.get()`** | `.get()` throws `NoSuchElementException` with no context, so the stack trace tells you nothing about what was missing |
| **Never `parallelStream()` in a request path** | It runs on the shared common ForkJoinPool, so one bad call starves the whole JVM — and your requests are already parallel across Tomcat threads |
| **Never stream over a lazy JPA collection** | `flatMap(o -> o.getMonitors().stream())` is the N+1 problem in a functional disguise; fix it in the query, not the pipeline |
| **No side effects inside a stream; `peek` is for debugging only** | Accumulating into an external list isn't thread-safe and defeats the pipeline — `map(...).toList()` already does it |
| **Give `Collectors.toMap` a merge function** | It throws `IllegalStateException` on the first duplicate key, which real data supplies sooner than test data does |
| **`var` only when the right-hand side names the type** | `var x = service.process()` removes information rather than redundancy, and the reader pays for it on every future read |
| **`switch` expressions over enums and sealed types, with no `default`** | Exhaustiveness turns "someone added a case and forgot a branch" from a production bug into a compile error |
| **Always pass an explicit executor to `CompletableFuture`** | The default is the common ForkJoinPool, shared with every parallel stream in the JVM and sized for CPU work, not I/O |
| **`log.debug("state {}", monitor)`, never string concatenation** | SLF4J defers formatting, so the cost disappears entirely when the level is off |
| **Reach for a `for` loop when you're iterating for side effects** | Streams win at transformation pipelines; for plain iteration a loop is more readable and you can actually step through it |

---

## 9. Interview traps

**Q1. What is a functional interface, and why do lambdas need one?**
An interface with exactly one abstract method. A lambda is an implementation of one — the compiler uses the target type to know which interface. `@FunctionalInterface` makes the constraint compiler-checked. This is why lambdas work with *any* single-method interface, including your own and Spring's `Customizer`.

**Q2. Are streams lazy? Prove it.**
Yes — intermediate ops build a pipeline and do nothing until a terminal op runs. Proof: `.peek(print).filter(...).findFirst()` on a large list prints only until the first match, not for every element. Elements are pulled through one at a time, so short-circuiting ops can skip most of the data.

**Q3. `map` vs `flatMap`?**
`map` is 1→1 (`Stream<Order>` → `Stream<UUID>`). `flatMap` is 1→many-then-flattened (`Stream<Order>` → `Stream<OrderLine>` from each order's list). Use `flatMap` whenever your mapper returns a stream/collection/Optional and you don't want nesting.

**Q4. When would you use `parallelStream()`?**
Rarely. Only for large, CPU-bound, in-memory work with no shared mutable state and no I/O — and only after measuring. Risks: it uses the shared common ForkJoinPool (so it interferes globally), splitting/merging overhead, and in a web server requests are already parallel, so it competes with your own throughput.

**Q5. `orElse` vs `orElseGet` vs `orElseThrow`?**
`orElse(v)` — the argument is **always evaluated**, even when the Optional has a value. `orElseGet(supplier)` — lazy, only runs when empty; use it whenever the default is expensive or has side effects. `orElseThrow(supplier)` — the standard Spring pattern for "not found."

**Q6. Should `Optional` be used for fields or method parameters?**
No. It's designed as a return type expressing "no result." As a field it adds indirection and breaks serialization/JPA expectations; as a parameter it forces callers to wrap values — use overloads or `@Nullable` instead.

**Q7. Why must lambdas capture only effectively-final locals?**
Because the lambda can outlive the enclosing method invocation (and may run on another thread). Java captures the *value*, so permitting reassignment would give ambiguous, unsafe semantics. Instance fields are capturable — which is exactly how people accidentally create races.

**Q8. What's the risk of `.collect(Collectors.toMap(...))`?**
It throws `IllegalStateException` on duplicate keys — often surprising with real data. Provide a merge function: `toMap(k, v, (a, b) -> a)`. It also NPEs on null values, unlike `HashMap`.

**Q9. How do sealed interfaces improve on plain interfaces?**
They restrict which types may implement them, so the compiler knows the full set — enabling exhaustive `switch` without a `default`. Adding a new implementation becomes a compile error at every decision point, which is exactly what you want.

---

## 10. Retention check

1. Why is `Customizer<T>` in Spring Security config just a lambda, and what does that tell you about reading unfamiliar Spring APIs?
2. You see `orElse(expensiveCall())` in a code review. What do you comment?
3. A colleague uses `flatMap` over a JPA lazy collection to "flatten elegantly." What's your concern?
4. Why does `.filter().findFirst()` on a 1M-element list not evaluate 1M predicates?
5. Name the one place `Optional` belongs, and two places it doesn't.

---

**Next:** [Lesson 04 — Annotations, reflection & proxies](04-annotations-reflection-proxies.md). This is the most important lesson in the entire course. After it, Spring will never feel like magic again.

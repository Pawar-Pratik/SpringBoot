# Lesson 20 — JPA & Hibernate fundamentals

> **Why this lesson exists:** this is the most important lesson in Module 6. The **persistence context** is the concept that explains almost every surprising JPA behaviour: why an entity saves without `save()`, why you get `LazyInitializationException`, why `save()` sometimes issues no SQL, why an update fires at a moment you didn't choose. Most people who use JPA daily cannot explain it. Learn it properly and you'll debug things your colleagues consider mysterious.

**Time:** ~90 minutes · **Prereq:** Lesson 19

---

## 1. The idea in one sentence

> **The persistence context is a transaction-scoped map of managed entities plus a snapshot of how each one looked when it was loaded — and nearly every surprising JPA behaviour is either that snapshot being compared at flush, or an entity having fallen out of the map.**

Dirty checking without `save()`, the silent write from a GET, `LazyInitializationException`, deferred and reordered SQL: one mechanism, four symptoms. Ask "is this entity managed, and what does its snapshot say?" and the mystery usually evaporates.

---

## 2. What JPA is, and what it isn't

| | |
|---|---|
| **JPA** | A *specification* (Jakarta Persistence API). Interfaces and annotations: `EntityManager`, `@Entity`, `@OneToMany`, JPQL. Ships no implementation. |
| **Hibernate** | The dominant *implementation*. Also has features beyond JPA (`@Formula`, filters, its own criteria API, second-level cache providers). |
| **Spring Data JPA** | A layer *above* JPA that generates repository implementations from interfaces. Not an ORM itself. |

```
Your code
   ↓
Spring Data JPA        ← generates repository implementations
   ↓
JPA (specification)    ← EntityManager, annotations
   ↓
Hibernate              ← the actual ORM engine
   ↓
JDBC                   ← Lesson 19
   ↓
PostgreSQL
```

**Boot's default is Hibernate**, and 95% of Spring shops use it. Alternatives (EclipseLink, OpenJPA) exist but are rare. **A lot of what people call "JPA behaviour" is actually Hibernate behaviour** — flush timing details, `@DynamicUpdate`, batch fetching. Knowing which is which is a nice distinction to draw.

### The ORM impedance mismatch

Objects and relational tables model the world differently, and the gaps are where every ORM problem lives:

| Object world | Relational world |
|---|---|
| Object references, arbitrary graphs | Foreign keys, flat rows |
| Inheritance | No inheritance |
| Identity (`==`), equality (`equals`) | Primary keys |
| Navigate a reference freely | Must JOIN, and JOINs cost |
| Collections in memory | Result sets |

**The mismatch that causes most pain: navigating a reference is free in Java and expensive in SQL.** `monitor.getOrganization().getName()` looks like a field access and is actually a network round trip. That single asymmetry produces the N+1 problem, `LazyInitializationException`, and most ORM performance complaints. Hold onto it.

---

## 3. The persistence context — the core concept

> **The persistence context is a first-level cache and change tracker for managed entities, scoped to a transaction.**

Mechanically, it's `Map<EntityKey, Object>` plus a snapshot of each entity's loaded state. `EntityManager` is your handle to it. In Spring, **one persistence context = one transaction** (by default).

### What it gives you

**1. Identity guarantee — within one context, one row = one object**

```java
@Transactional
public void demo(UUID id) {
    Monitor a = em.find(Monitor.class, id);      // SELECT ... (hits the DB)
    Monitor b = em.find(Monitor.class, id);      // NO SQL — returned from the context

    System.out.println(a == b);                  // true! Same object reference.
}
```

That's not a cache optimization detail; it's a **correctness guarantee**. You can't have two divergent in-memory copies of one row within a transaction.

**2. Automatic dirty checking — the behaviour that shocks people**

```java
@Transactional
public void renameMonitor(UUID id, String newName) {
    Monitor m = monitorRepository.findById(id).orElseThrow();
    m.setName(newName);
    // NO save() CALL. NO repository.update(). Nothing.
}   // ← on commit, Hibernate compares m against its loaded snapshot,
    //   sees name changed, and issues UPDATE monitor SET name = ? WHERE id = ?
```

**A managed entity's changes are persisted automatically at flush.** Calling `save()` is not required. This is the single most surprising JPA behaviour to newcomers, and the mechanism — snapshot comparison at flush — is exactly what an interviewer wants to hear.

**The corollary is a real bug source:**
```java
@Transactional
public MonitorResponse get(UUID id) {
    Monitor m = repo.findById(id).orElseThrow();
    m.setName(m.getName().trim());        // "just tidying the response"
    return mapper.toResponse(m);
}   // 💥 You just wrote to the database from a GET.
```
Accidental mutation of a managed entity is a silent write. **`@Transactional(readOnly = true)` on read paths prevents this** (§5) — one of the most valuable defaults in this lesson.

**3. Write-behind: SQL is deferred and batched**

```java
@Transactional
public void createThree() {
    repo.save(new Monitor("a"));
    repo.save(new Monitor("b"));
    repo.save(new Monitor("c"));
    // no SQL yet (unless the ID generator forces it — see §4)
}   // ← flush at commit: all three INSERTs, possibly batched into one round trip
```

Deferring lets Hibernate batch statements, order them to respect foreign keys, and skip writes for entities that were changed and changed back.

### Entity states — memorize this diagram

```
                    new Monitor(...)
                          │
                          ▼
                   ┌─────────────┐
                   │  TRANSIENT  │   not in the context, no DB row, no ID
                   └──────┬──────┘
                          │ persist() / save()
                          ▼
   find() ──────▶  ┌─────────────┐  ◀────── merge()
   query   ──────▶ │   MANAGED   │           (returns a MANAGED COPY)
                   │             │
                   │ tracked ✅   │  changes auto-persisted at flush
                   └──┬───────┬──┘
            remove()  │       │  detach() / clear() / close() / tx ends
                      ▼       ▼
              ┌───────────┐  ┌──────────┐
              │  REMOVED  │  │ DETACHED │  has an ID, exists in DB,
              │  (DELETE  │  │          │  NOT tracked — changes are IGNORED
              │  at flush)│  └──────────┘
              └───────────┘
```

**The detached-entity trap — a very common production bug:**

```java
// ❌ No @Transactional → the context closed when findById's own transaction ended
public void renameMonitor(UUID id, String newName) {
    Monitor m = monitorRepository.findById(id).orElseThrow();   // managed, then DETACHED
    m.setName(newName);                                          // change goes nowhere
}   // no UPDATE. No error. Silent data loss.
```

```java
// ✅ Either open a transaction
@Transactional
public void renameMonitor(UUID id, String newName) {
    Monitor m = monitorRepository.findById(id).orElseThrow();
    m.setName(newName);
}   // UPDATE at commit

// ✅ Or explicitly save the detached entity
public void renameMonitor(UUID id, String newName) {
    Monitor m = monitorRepository.findById(id).orElseThrow();
    m.setName(newName);
    monitorRepository.save(m);          // merge() → SELECT then UPDATE
}
```

**"My update silently didn't persist" is almost always a detached entity.** Say that in an interview and you sound like someone who has debugged it — because everyone eventually does.

### `persist()` vs `merge()` — and what `save()` actually does

| | `persist(entity)` | `merge(entity)` |
|---|---|---|
| Argument becomes managed | ✅ the *same* object | ❌ returns a **managed copy** |
| For | new/transient entities | detached entities |
| If the entity has an ID that exists | throws | updates it |
| Return value | `void` | the managed instance — **you must use it** |

```java
Monitor detached = new Monitor(existingId, "new name");
Monitor managed = em.merge(detached);
managed.setName("x");        // ✅ tracked
detached.setName("y");       // ❌ ignored — detached is still detached
```

**Spring Data's `save()` is not a simple insert.** It's:
```java
// SimpleJpaRepository, essentially:
public <S extends T> S save(S entity) {
    if (entityInformation.isNew(entity)) { em.persist(entity); return entity; }
    else { return em.merge(entity); }
}
```

**Consequence:** calling `save()` on an entity with an assigned ID triggers `merge()`, which issues a **SELECT** to check whether the row exists before the INSERT/UPDATE. If you assign UUIDs yourself and bulk-`save()` 10,000 entities, you get **10,000 extra SELECTs**. Fixes: implement `Persistable<ID>` with an `isNew()` flag, or use `JdbcClient` for bulk inserts (Lesson 19). **This is a great "have you actually profiled JPA?" answer.**

---

## 4. Flush — when does SQL actually run?

```java
FlushModeType.AUTO    // default: before a query that could be affected, and at commit
FlushModeType.COMMIT  // only at commit
```

**Flush is triggered by:**
1. **Transaction commit** — always
2. **Before executing a JPQL/Criteria query** whose results could be affected by pending changes
3. **Explicit `em.flush()` / `repository.flush()` / `saveAndFlush()`**
4. **Sometimes by ID generation** (see §4 — `IDENTITY` forces an immediate insert)

The point of #2 is consistency: if you've changed an entity in memory and then run a query that would match it, Hibernate flushes first so the query sees your change.

```java
@Transactional
public void demo() {
    Monitor m = new Monitor("api");
    repo.save(m);                                       // maybe no SQL yet

    long count = repo.count();                          // ← triggers FLUSH first, then SELECT count
    // count includes 'api' ✅

    m.setName("renamed");
    // no SQL

}   // ← FLUSH: UPDATE, then COMMIT
```

**`flush()` is not `commit()`.** Flush writes pending SQL within the current transaction; the changes are still uncommitted and roll back on failure. People conflate these — a good discriminating question.

**When you'd flush manually:**
- To force a constraint violation early so you can handle it
- Before a native query that must see your pending changes
- Mid-batch, followed by `em.clear()`, to bound memory (Lesson 24)

**Flush order is fixed by Hibernate**, not by your call order: inserts (in entity-persist order), updates, collection deletes, collection updates/inserts, then entity deletes. **This causes a genuinely confusing bug:**

```java
@Transactional
public void replaceMonitorName(UUID orgId) {
    monitorRepository.delete(oldMonitor);        // name = "api"
    monitorRepository.save(new Monitor(orgId, "api"));
}   // 💥 ConstraintViolationException — the INSERT runs BEFORE the DELETE
```
Fix: `flush()` after the delete, or reuse the existing row. Knowing *why* it happens (fixed flush ordering, deletes last) rather than just the workaround is the senior answer.

---

## 5. Identity generation, and the trap in each strategy

```java
@Id @GeneratedValue(strategy = GenerationType.IDENTITY)   Long id;   // DB auto-increment
@Id @GeneratedValue(strategy = GenerationType.SEQUENCE)   Long id;   // DB sequence
@Id @GeneratedValue(strategy = GenerationType.UUID)       UUID id;   // JPA 3.1+
@Id                                                       UUID id;   // you assign it
```

| Strategy | How | Trap |
|---|---|---|
| `IDENTITY` | `SERIAL`/`AUTO_INCREMENT`; DB assigns on insert | ⚠️ **Disables JDBC batch inserts.** Hibernate must insert immediately to learn the ID, so it can't defer or batch. |
| `SEQUENCE` | `nextval()`, pre-fetched | ✅ **Best for Postgres/Oracle.** Batching works. Use an allocation size. |
| `TABLE` | A table simulating a sequence | ❌ Slow, contentious. Avoid. |
| `UUID` / assigned | Generated in the app | ✅ No DB round trip; but `save()` triggers `merge()` → extra SELECT (§2) |

**The `IDENTITY` batching problem is a genuinely valuable thing to know.** You configure `hibernate.jdbc.batch_size=50`, measure no improvement, and conclude batching "doesn't work." It's your ID strategy.

**`SEQUENCE` with allocation size — the practical setup:**
```java
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "monitor_seq")
@SequenceGenerator(name = "monitor_seq", sequenceName = "monitor_id_seq", allocationSize = 50)
private Long id;
```
Hibernate fetches 50 IDs per round trip instead of one. **`allocationSize` must match the sequence's `INCREMENT BY` in the database**, or you get collisions — a nasty, intermittent bug. Set both, or let Hibernate create the sequence.

### UUID vs sequential ID — the real trade-off

| | Sequential (`BIGSERIAL`) | UUID v4 (random) | UUID v7 (time-ordered) |
|---|---|---|---|
| Storage | 8 bytes | 16 bytes | 16 bytes |
| Index locality | ✅ Excellent (appends) | ❌ **Poor — random inserts fragment the B-tree** | ✅ Good |
| Guessable / enumerable | ❌ **`/monitors/1` invites enumeration** | ✅ Safe | ✅ Safe |
| Generatable client-side | ❌ needs a DB round trip | ✅ | ✅ |
| Merging data across shards/systems | ❌ collisions | ✅ | ✅ |

**Random UUID primary keys have a real cost:** inserts land at random points in the B-tree, causing page splits, worse cache locality, and index bloat. At high insert volume that's measurable.

**UUID v7 fixes it** by prefixing a timestamp so values are roughly ordered — you keep unguessability and regain insert locality. **Vigil uses UUID v7** for `check_result` (millions of rows, append-heavy) and this is exactly the kind of decision worth explaining in an interview: *"I chose UUIDv7 because I needed non-enumerable IDs for a multi-tenant API but couldn't afford random-insert index fragmentation on the high-volume table."*

**And the enumeration point is a real security concern:** with `/api/v1/monitors/1`, an attacker walks your entire dataset by incrementing. Combined with a missing authorization check, that's a full data breach. UUIDs don't *replace* authorization, but they remove the trivial discovery path.

---

## 6. Lazy loading and `LazyInitializationException`

```java
@Entity
public class Monitor {
    @ManyToOne(fetch = FetchType.LAZY)          // ← default for @ManyToOne is EAGER! set LAZY
    private Organization organization;

    @OneToMany(mappedBy = "monitor")            // ← default for collections is LAZY ✅
    private List<CheckResult> results;
}
```

**Default fetch types (memorize — a guaranteed interview question):**

| Association | JPA default | What you should use |
|---|---|---|
| `@ManyToOne` | **EAGER** | **LAZY** |
| `@OneToOne` | **EAGER** | **LAZY** |
| `@OneToMany` | LAZY ✅ | LAZY |
| `@ManyToMany` | LAZY ✅ | LAZY |

**`@ManyToOne` defaulting to EAGER is one of JPA's worst defaults.** Every `Monitor` load also loads its `Organization`, which loads *its* associations... and one `findAll()` becomes a JOIN cascade or dozens of extra queries. **Set `fetch = LAZY` on every to-one association and fetch explicitly when needed.** Note: `EAGER` can't be overridden per-query, while `LAZY` can be upgraded with a fetch join — so LAZY is strictly more flexible.

### How lazy loading works (Lesson 09, again)

```java
Monitor m = repo.findById(id).orElseThrow();
Organization org = m.getOrganization();
// org is Organization$HibernateProxy$aB3xY — a generated SUBCLASS, not a real Organization

org.getName();          // ← NOW: SELECT * FROM organization WHERE id = ?
org.getId();            // ← usually NO query — the proxy already holds the ID
```

**Same proxy mechanism as `@Transactional`.** Which means the same limitations: a `final` entity class or `final` getters can't be proxied, so Hibernate silently falls back to eager loading.

### The exception

```java
public MonitorResponse get(UUID id) {                    // no @Transactional
    Monitor m = repo.findById(id).orElseThrow();         // context closes here
    return new MonitorResponse(m.getOrganization().getName());   // 💥
}
```
```
LazyInitializationException: could not initialize proxy [Organization#...] - no Session
```

**The cause:** the proxy needs an open persistence context to run its query, and the context closed when the transaction ended. It's not a bug — it's Hibernate telling you the truth: *"you asked for data outside the boundary where I could fetch it."*

### The fixes, ranked

| Fix | Verdict |
|---|---|
| **Fetch what you need in the query** — `JOIN FETCH` or `@EntityGraph` | ✅ **Correct.** Explicit, one query, no surprises. |
| **DTO projection** — select exactly the fields into a record | ✅ **Best for read paths.** No entities, no lazy loading possible. |
| `@Transactional` on the service method that also builds the DTO | ✅ Fine, as long as mapping happens inside the boundary |
| `Hibernate.initialize(m.getOrganization())` | ⚠️ Works; a code smell — you're patching per call site |
| `fetch = EAGER` | ❌ **Fixes one call site, degrades every other.** Now every query over-fetches. |
| `spring.jpa.open-in-view=true` (the **default**) | ❌ **See below.** |

### `open-in-view` — turn it off, and know why

Spring Boot's `spring.jpa.open-in-view` defaults to **`true`**. It keeps the persistence context open for the whole HTTP request, including view rendering and JSON serialization. So lazy loading "just works" from a controller.

**Why it's harmful:**

1. **It holds a database connection for the entire request**, including JSON serialization and the time spent writing bytes to a slow client. **This is a top cause of connection pool exhaustion** (Lesson 19).
2. **It hides N+1 problems.** Jackson walking a lazy collection fires a query per element, in the *view* layer, invisible to your service tests. It works in development and collapses in production.
3. **It makes queries happen in the presentation layer**, where you have no transaction control and no way to reason about them.
4. **It masks design errors.** If a service returns an entity that a controller must lazily navigate, the service's contract is incomplete.

```yaml
spring:
  jpa:
    open-in-view: false           # ★ set this in every project, day one
```

**Boot even logs a warning about it at startup**, which most people ignore. When you turn it off you'll get `LazyInitializationException`s — **that's the point.** Each one is a place where you were doing an unplanned query. Fix them with fetch joins or projections.

**This is one of the highest-value things you can say in a JPA interview**, because it's specific, it's a real production issue, and it demonstrates you've looked past the defaults.

### `readOnly = true` — the other default worth adopting

```java
@Transactional(readOnly = true)
public MonitorResponse get(UUID id) { ... }
```

Buys you:
- **Hibernate skips dirty checking** and doesn't take entity snapshots → less memory and CPU on read paths
- **Accidental mutations aren't flushed** (the §2 trap becomes harmless)
- The JDBC connection is marked read-only, so some databases/proxies can route it to a **read replica**

**Convention worth adopting:** `@Transactional(readOnly = true)` at the *class* level on services, overridden with plain `@Transactional` on write methods. Cheap, and it prevents a whole bug class.

---

## 7. Build & break

**1. Set up JPA.**
```xml
<dependency><groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-jpa</artifactId></dependency>
```
```yaml
spring:
  jpa:
    open-in-view: false                    # ★
    hibernate:
      ddl-auto: validate                   # your hand-written schema from Lesson 19
    properties:
      hibernate:
        format_sql: true
        jdbc.batch_size: 50
        order_inserts: true
        order_updates: true
logging:
  level:
    org.hibernate.SQL: DEBUG                          # ★ see every statement
    org.hibernate.orm.jdbc.bind: TRACE                # ★ see bound parameters
```
**Turn on that SQL logging now and leave it on for all of Module 6.** You cannot learn JPA without seeing the SQL it generates. This is non-negotiable.

**2. Map the `Monitor` entity** against your existing table. Set `fetch = LAZY` on every to-one association.

**3. ⭐ Prove dirty checking.**
```java
@Transactional
public void renameWithoutSave(UUID id, String name) {
    Monitor m = repo.findById(id).orElseThrow();
    m.setName(name);
    log.info("no save() called");
}
```
Run it and **watch the UPDATE appear in the SQL log.** Then remove `@Transactional` and run again — **no UPDATE, no error, silent data loss.** These two runs, side by side, teach you managed vs detached permanently.

**4. Prove identity.**
```java
@Transactional
public void identityDemo(UUID id) {
    Monitor a = repo.findById(id).orElseThrow();
    Monitor b = repo.findById(id).orElseThrow();
    log.info("same object: {}", a == b);              // true
    log.info("queries in log: count them");           // ONE select
}
```

**5. Observe flush timing.**
```java
@Transactional
public void flushDemo() {
    repo.save(new Monitor(...));
    log.info("--- after save, before count ---");
    long n = repo.count();                            // watch INSERT appear HERE
    log.info("--- after count ---");
}                                                     // watch COMMIT here
```
Match the log markers to the SQL. **Then call `repo.saveAndFlush(...)` instead and note the INSERT moves earlier.**

**6. Reproduce the flush-ordering constraint violation** from §3: delete a monitor and insert a new one with the same unique name in one transaction. Read the exception, then fix it with an explicit `flush()`.

**7. `LazyInitializationException`, then three fixes.** With `open-in-view: false`, return a DTO built from `monitor.getOrganization().getName()` in a non-transactional method. Read the exception. Then fix it:
- `@Transactional(readOnly = true)` with mapping inside
- `@EntityGraph(attributePaths = "organization")` on the repository method
- a JPQL DTO projection

**Compare the SQL each produces.** The projection issues the smallest query — that's the lesson.

**8. Prove `open-in-view` is dangerous.** Set it back to `true`, return an entity with a lazy collection from a controller, and count the queries in the log. Then set it to `false` and watch the same endpoint fail. **The failure is the framework protecting you.**

**9. Prove the `IDENTITY` batching problem.** Create an entity with `GenerationType.IDENTITY`, save 100 in one transaction with `batch_size=50`, and count the INSERT statements: 100. Switch to `SEQUENCE` with `allocationSize=50` and count again: batched. **This is the experiment that makes you the person who knows why batching "isn't working."**

**10. Prove the `save()` extra SELECT.** With an assigned UUID id, save a new entity and watch for a `SELECT` before the `INSERT` — that's `merge()`. Then implement `Persistable<UUID>` with an `isNew` transient flag and watch the SELECT disappear.

### Explain out loud (60s)

1. What the persistence context is, mechanically, in one sentence.
2. How dirty checking works, and why `save()` isn't needed.
3. The detached-entity trap, and why it fails silently.
4. Why `@Transactional(readOnly = true)` on reads is a correctness measure, not an optimization.
5. One reason `open-in-view: true` is a trap, in terms of where the queries end up running.

---

## 8. Production rules

| Do / Never | Why |
|---|---|
| **Put `@Transactional(readOnly = true)` on every read path** | Changes to a managed entity are written at flush whether you meant them or not — read-only makes an accidental mutation harmless instead of silent |
| **Never mutate an entity you only intended to read** | `m.setName(m.getName().trim())` inside a GET is a database write; nothing in the code says so |
| **Never load an entity outside a transaction and expect changes to persist** | The context closes with the transaction, so the entity is detached and your setter writes to nothing — no UPDATE, no error |
| **Don't call `save()` on an already-managed entity** | Dirty checking already handles it; the habit hides the fact that you don't know whether your entity is managed |
| **Set `open-in-view: false` explicitly** | The default is `true`, which keeps the context open through view rendering and turns every lazy getter in your serializer into a query |
| **Keep every to-one association `LAZY`** | `EAGER` can't be overridden per query, so one convenient default degrades every other query that touches the entity |
| **Know what your ID generator costs** | `IDENTITY` forces an immediate INSERT on `save()`, which disables batching entirely — a surprise only visible in the SQL log |
| **Never serialize an entity outside its transaction** | Jackson calls every getter, so a lazy association either explodes with `LazyInitializationException` or quietly fetches a million rows |
| **Never rely on the order of your `save()` calls** | Hibernate defers and reorders statements at flush to respect foreign keys and enable batching; your call order is not the SQL order |
| **Turn on `org.hibernate.SQL` logging and leave it on while learning** | JPA's whole difficulty is the distance between what you wrote and what it ran; the log closes that distance |

---

## 9. Interview traps

**Q1. What is the persistence context?**
A transaction-scoped first-level cache and change tracker for managed entities — effectively a map from entity key to instance, plus a snapshot of each entity's loaded state. It provides identity guarantees (one row = one object per context), automatic dirty checking, and write-behind (deferred, batchable SQL).

**Q2. Why does an entity update without calling `save()`?**
Because it's *managed*. At flush, Hibernate compares each managed entity against its loaded snapshot and issues UPDATEs for differences. `save()` is only needed for new entities or to re-attach detached ones.

**Q3. Name the four entity states and how you move between them.**
Transient (new, untracked) → `persist()` → Managed (tracked) → `detach()`/`clear()`/context close → Detached (has an ID, untracked); or Managed → `remove()` → Removed (DELETE at flush). `merge()` takes a detached entity and returns a *managed copy*.

**Q4. My update didn't persist and there's no error. Why?**
The entity was detached — most often because the method had no `@Transactional`, so the persistence context from the read closed immediately and the mutation was never tracked. Fix by opening a transaction around read+modify, or by explicitly calling `save()` (which merges).

**Q5. `persist()` vs `merge()`, and what does Spring Data's `save()` do?**
`persist()` makes the *given* transient instance managed. `merge()` copies state from a detached instance into a managed one and returns **that copy** — the argument stays detached. `save()` checks `isNew()` and calls `persist()` or `merge()` accordingly, which is why saving entities with assigned IDs issues an extra SELECT per entity.

**Q6. When does Hibernate flush?**
At commit; before a JPQL/Criteria query whose results could be affected by pending changes (with default `AUTO` flush mode); on explicit `flush()`; and immediately on insert when using `IDENTITY` ID generation. Flush ≠ commit — flushed changes are still rolled back on failure.

**Q7. What is `LazyInitializationException` and how do you fix it properly?**
Accessing an uninitialized proxy or lazy collection after the persistence context closed. Proper fixes: fetch what you need in the query (`JOIN FETCH`/`@EntityGraph`) or use a DTO projection. Bad fixes: switching to `EAGER` (degrades all other queries and can't be undone per-query) or relying on `open-in-view`.

**Q8. What is `open-in-view`, what's the default, and why should you disable it?**
It keeps the persistence context open for the whole HTTP request; Spring Boot defaults it to **true**. Disable it because it holds a database connection through view rendering and serialization (a leading cause of pool exhaustion), hides N+1 problems in the presentation layer, and masks incomplete service contracts.

**Q9. What are JPA's default fetch types and what would you change?**
`@ManyToOne` and `@OneToOne` default to **EAGER**; collections default to LAZY. Set all to-one associations `LAZY` explicitly — EAGER can't be overridden per query, whereas LAZY can be upgraded with a fetch join, so LAZY is strictly more flexible.

**Q10. Why does `GenerationType.IDENTITY` prevent batch inserts?**
Hibernate must execute each INSERT immediately to obtain the generated key, so it cannot defer or group statements. Use `SEQUENCE` with an `allocationSize` (matching the DB sequence's increment) to get batching.

**Q11. UUID or auto-increment primary keys?**
Sequential IDs give the best index locality and smallest storage but are enumerable, which is an information-disclosure and data-scraping risk in a multi-tenant API. Random UUID v4 is safe but fragments the B-tree on insert. **UUID v7** is time-ordered, giving unguessability with good insert locality — the best default for high-volume tables in a multi-tenant system.

**Q12. What does `@Transactional(readOnly = true)` actually do?**
Hibernate skips dirty-check snapshots (less memory/CPU and no accidental flushes), the JDBC connection is flagged read-only, and some infrastructure can route it to a read replica. A good class-level default on services, overridden on write methods.

---

## 10. Retention check

1. Define the persistence context in one sentence, then name the three things it gives you.
2. Explain, mechanically, why an entity updates without `save()`.
3. Draw the four entity states and the transitions between them.
4. Give three reasons `open-in-view=true` is harmful.
5. Which fetch types default to EAGER, and why is LAZY strictly more flexible?
6. Why does `IDENTITY` break batch inserts, and what do you use instead?

---

**Next:** [Lesson 21 — Entity mapping & relationships](21-entity-mapping-and-relationships.md) — mapping any schema correctly, including the bidirectional and `equals`/`hashCode` traps that catch nearly everyone.

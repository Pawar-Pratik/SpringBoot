# Lesson 22 — Spring Data JPA repositories

> **Why this lesson exists:** Spring Data makes queries so easy that people stop thinking about the SQL they generate. That's how you end up with a 40-character method name, a query that loads entire entities to read one column, and an N+1 problem in a dashboard. This lesson covers every querying tool and — more importantly — when each is the right one.

**Time:** ~80 minutes · **Prereq:** Lessons 20–21

---

## 1. How a repository interface becomes a working bean

```java
public interface MonitorRepository extends JpaRepository<Monitor, Long> {
    Optional<Monitor> findByOrganizationIdAndName(Long orgId, String name);
}
```

You wrote an interface with no implementation. It works. **How?**

```
1. @EnableJpaRepositories (auto-configured by Boot) registers a
   JpaRepositoryConfigExtension

2. It scans for interfaces extending Repository (directly or transitively)

3. For each, it registers a JpaRepositoryFactoryBean  ← a FactoryBean! (Lesson 06)

4. At creation, the factory:
   a. Instantiates SimpleJpaRepository — the real implementation of all the
      standard methods (findById, save, findAll, delete...)
   b. PARSES each custom method name into a query (see §2)
   c. Creates a JDK dynamic PROXY implementing your interface,
      routing each call to either SimpleJpaRepository or the derived query

5. The proxy is registered as a bean. That's what gets injected.
```

**Nothing here is new to you** — it's `FactoryBean` + reflection + proxies (Lessons 04, 06, 09). Being able to explain this is a strong interview moment, because most people treat Spring Data as pure magic.

**Proof you can run in 10 seconds:**
```java
log.info("{}", monitorRepository.getClass().getName());
// → com.sun.proxy.$Proxy142   (or a similar generated name)
```

### The repository hierarchy

```
Repository<T, ID>                          marker interface only
   ▲
CrudRepository<T, ID>                      save, findById, findAll, delete, count, existsById
   ▲
ListCrudRepository<T, ID>                  ← Spring Data 3+: returns List instead of Iterable
   ▲
PagingAndSortingRepository<T, ID>          findAll(Pageable), findAll(Sort)
   ▲
JpaRepository<T, ID>                       + flush, saveAndFlush, deleteAllInBatch, getReferenceById
```

**Which to extend?**

| Interface | When |
|---|---|
| `Repository<T,ID>` | You want to expose *only* the methods you declare — **the most disciplined choice** |
| `CrudRepository` | Basic CRUD, no paging |
| `JpaRepository` | The pragmatic default; JPA-specific extras |

**The disciplined argument, worth making in a review:** extending `JpaRepository` exposes ~20 methods including `deleteAll()` and `findAll()`. On a table with 50 million rows, an innocent `findAll()` is an outage. Extending bare `Repository` and declaring only what you need makes the repository's contract intentional:

```java
public interface CheckResultRepository extends Repository<CheckResult, Long> {
    // deliberately NO findAll() — this table has 50M rows
    List<CheckResult> findTop100ByMonitorIdOrderByCheckedAtDesc(Long monitorId);
    void save(CheckResult result);
}
```

**Also: `getReferenceById` vs `findById`.** `getReferenceById` (formerly `getOne`) returns a **lazy proxy without querying** — perfect for setting a foreign key without loading the row:
```java
monitor.setOrganization(orgRepository.getReferenceById(orgId));   // no SELECT ✅
```
The catch: it throws `EntityNotFoundException` *later*, on first access, not at call time. Great for FK assignment; wrong when you need to know the row exists.

---

## 2. Derived query methods

Spring Data parses the method name into a query:

```java
public interface MonitorRepository extends JpaRepository<Monitor, Long> {

    List<Monitor> findByStatus(MonitorStatus status);
    List<Monitor> findByStatusAndOrganizationId(MonitorStatus status, Long orgId);
    List<Monitor> findByStatusOrStatus(MonitorStatus a, MonitorStatus b);
    List<Monitor> findByStatusIn(Collection<MonitorStatus> statuses);
    List<Monitor> findByNameContainingIgnoreCase(String fragment);   // LIKE %x% — see the warning
    List<Monitor> findByLastCheckedAtBefore(Instant cutoff);
    List<Monitor> findByLastCheckedAtBetween(Instant from, Instant to);
    List<Monitor> findByOrganizationIdOrderByNameAsc(Long orgId);
    List<Monitor> findTop10ByOrganizationIdOrderByCreatedAtDesc(Long orgId);
    Optional<Monitor> findFirstByUrlOrderByCreatedAtAsc(String url);
    List<Monitor> findByLastErrorIsNotNull();
    boolean existsByOrganizationIdAndName(Long orgId, String name);
    long countByStatus(MonitorStatus status);
    void deleteByOrganizationId(Long orgId);

    // Navigate relationships with underscores or camelCase
    List<Monitor> findByOrganization_Plan(Plan plan);
    List<Monitor> findByOrganizationPlan(Plan plan);        // works if unambiguous
}
```

**Keywords:** `And Or Is Equals Between LessThan GreaterThan After Before IsNull IsNotNull Like NotLike StartingWith EndingWith Containing IgnoreCase OrderBy Not In NotIn True False Distinct`

**Prefixes:** `find… get… read… query… search… stream… count… exists… delete… remove…`

### When derived queries stop being a good idea

```java
// ❌ You have created a write-only method name.
List<Monitor> findByOrganizationIdAndStatusInAndLastCheckedAtBeforeAndNameContainingIgnoreCaseOrderByCreatedAtDesc(
        Long orgId, Collection<MonitorStatus> statuses, Instant cutoff, String name);
```

**The rule: 2–3 conditions maximum.** Beyond that, use `@Query` (readable) or a `Specification` (dynamic). A method name is documentation; when it stops being readable it has stopped doing its job.

**Two more traps:**

**`findBy…Containing` generates `LIKE '%value%'`** — a leading wildcard, which **cannot use a normal B-tree index**. On a large table that's a full scan. For real text search use Postgres full-text search (`tsvector` + GIN index) or a trigram index (`pg_trgm`). Naming this unprompted is a strong signal.

**`deleteByX` loads then deletes one-by-one.** Derived delete methods fetch matching entities into the persistence context and issue an individual DELETE for each (so lifecycle callbacks and cascades fire). For bulk deletion use `@Modifying @Query`. **A derived delete over 100,000 rows is 100,001 statements.**

---

## 3. `@Query` — JPQL and native SQL

```java
public interface MonitorRepository extends JpaRepository<Monitor, Long> {

    // JPQL — operates on ENTITIES and FIELDS, not tables and columns
    @Query("""
           SELECT m FROM Monitor m
           WHERE m.organization.id = :orgId
             AND m.status IN :statuses
           ORDER BY m.createdAt DESC
           """)
    List<Monitor> search(@Param("orgId") Long orgId,
                         @Param("statuses") Collection<MonitorStatus> statuses);

    // JOIN FETCH — solve N+1 by loading the association in ONE query   [Lesson 24]
    @Query("SELECT m FROM Monitor m JOIN FETCH m.organization WHERE m.status = :status")
    List<Monitor> findByStatusWithOrganization(@Param("status") MonitorStatus status);

    // DTO projection — build the record directly in the query
    @Query("""
           SELECT new com.vigil.monitor.dto.MonitorSummary(m.id, m.name, m.status, o.name)
           FROM Monitor m JOIN m.organization o
           WHERE o.id = :orgId
           """)
    List<MonitorSummary> summariesForOrg(@Param("orgId") Long orgId);

    // Native SQL — for things JPQL can't express
    @Query(value = """
           SELECT m.id, m.name,
                  count(*) FILTER (WHERE cr.status = 'UP')::float / count(*) * 100 AS uptime_pct,
                  percentile_cont(0.95) WITHIN GROUP (ORDER BY cr.response_time_ms) AS p95_ms
           FROM monitor m
           JOIN check_result cr ON cr.monitor_id = m.id
           WHERE m.org_id = :orgId AND cr.checked_at >= :since
           GROUP BY m.id, m.name
           """, nativeQuery = true)
    List<UptimeRow> uptimeReport(@Param("orgId") UUID orgId, @Param("since") Instant since);

    // Modifying query — bulk update/delete
    @Modifying(clearAutomatically = true, flushAutomatically = true)
    @Query("UPDATE Monitor m SET m.status = :status WHERE m.lastCheckedAt < :cutoff")
    int markStale(@Param("status") MonitorStatus status, @Param("cutoff") Instant cutoff);
}
```

### JPQL vs native SQL

| | JPQL | Native SQL |
|---|---|---|
| Operates on | Entities, fields | Tables, columns |
| Portable across databases | ✅ | ❌ |
| Returns managed entities | ✅ | ⚠️ needs `resultSetMapping` or a projection |
| Window functions, CTEs, `FILTER`, percentiles | ❌ | ✅ |
| Type-checked at startup | ✅ (Hibernate validates JPQL at boot) | ❌ (fails at execution) |

**The JPQL startup validation is a real, underrated benefit:** a typo in a JPQL query fails your application at boot, not at 2am when that endpoint is first hit. Native queries have no such check — which is a good reason to keep them tested.

### `@Modifying` — and the two flags that prevent silent bugs

```java
@Modifying(clearAutomatically = true, flushAutomatically = true)
@Query("UPDATE Monitor m SET m.status = 'DOWN' WHERE m.id = :id")
int markDown(@Param("id") Long id);
```

**Bulk JPQL updates bypass the persistence context entirely.** They go straight to SQL. Consequences:

1. **Entities already loaded in the context become stale** — they still hold the old value, and a subsequent read returns the *stale cached* object, not the database's new value. `clearAutomatically = true` evicts the context to prevent this.
2. **Pending changes might not be written yet**, so your bulk update could operate on rows that don't reflect in-memory modifications. `flushAutomatically = true` flushes first.
3. **`@Version` is not incremented** and optimistic locking is skipped.
4. **No lifecycle callbacks or cascades** — no `@PreUpdate`, no auditing fields updated.

**Set both flags unless you have a reason not to.** And know the trade-off: `clearAutomatically` detaches *everything*, so any entity you were holding becomes detached (Lesson 20's silent-update trap). Being able to state both the fix and its cost is the complete answer.

`@Modifying` queries also **require an active transaction**, or you get `TransactionRequiredException`.

### SQL injection in `@Query`

```java
// ✅ SAFE — parameterized
@Query("SELECT m FROM Monitor m WHERE m.name = :name")

// ❌ INJECTABLE — this compiles and ships
@Query(value = "SELECT * FROM monitor WHERE name = '" + "#{#name}" + "'", nativeQuery = true)
```
**Never concatenate or SpEL-interpolate user input into a query string.** Parameters are safe because they're bound separately (Lesson 19). And **you cannot parameterize identifiers** — a dynamic `ORDER BY` column name must be validated against an allowlist:

```java
private static final Set<String> SORTABLE = Set.of("name", "created_at", "status");
if (!SORTABLE.contains(sortColumn)) throw new BadRequestException("invalid sort");
```

`Sort` and `Pageable` from Spring Data are safe (validated against entity properties), but hand-built dynamic SQL is not. This is a genuinely common real vulnerability.

---

## 4. Projections — read only what you need

**This is one of the biggest, easiest performance wins in JPA, and most people don't use it.**

Loading a full entity to read three columns means: selecting every column, allocating the entity, taking a dirty-check snapshot, and registering it in the persistence context. For a 1,000-row dashboard query that's all waste.

### Interface projections (closed) — the cleanest option

```java
public interface MonitorSummaryView {
    Long getId();
    String getName();
    MonitorStatus getStatus();
}
```
```java
List<MonitorSummaryView> findByOrganizationId(Long orgId);
```
**Hibernate generates `SELECT m.id, m.name, m.status` — only those columns.** Spring Data returns proxies implementing the interface. Nothing is managed, nothing is dirty-checked.

### Nested projections

```java
public interface MonitorWithOrgView {
    Long getId();
    String getName();
    OrganizationView getOrganization();

    interface OrganizationView { String getName(); }
}
```
Generates an efficient join selecting only those columns. **No N+1**, because there's no lazy association to traverse.

### Record/DTO projections

```java
public record MonitorSummary(Long id, String name, MonitorStatus status, String orgName) {}
```
```java
@Query("""
       SELECT new com.vigil.monitor.dto.MonitorSummary(m.id, m.name, m.status, o.name)
       FROM Monitor m JOIN m.organization o WHERE o.id = :orgId
       """)
List<MonitorSummary> summaries(@Param("orgId") Long orgId);
```
The fully-qualified class name in JPQL is ugly, but it's typed and immutable. **Spring Data can also do it without the constructor expression** if the record's component names match the interface-projection rules — try both.

### Open (SpEL) projections — know the cost

```java
public interface MonitorLabelView {
    String getName();

    @Value("#{target.name + ' (' + target.status + ')'}")
    String getLabel();
}
```
**⚠️ Open projections defeat the optimization** — because SpEL can reference anything, Hibernate loads the **whole entity**. If your goal was fewer columns, an open projection silently undoes it. Good trap question.

### Dynamic projections

```java
<T> List<T> findByOrganizationId(Long orgId, Class<T> type);
```
```java
List<MonitorSummaryView> light = repo.findByOrganizationId(orgId, MonitorSummaryView.class);
List<Monitor> full          = repo.findByOrganizationId(orgId, Monitor.class);
```
One method, caller chooses the shape. Genuinely elegant.

**The habit to build: for every read-only endpoint, ask "do I need entities here?"** Usually the answer is no, and a projection is faster, uses less memory, and can't cause a lazy-loading problem. **Vigil's dashboard and status page use projections exclusively.**

---

## 5. Pagination and sorting

```java
Page<Monitor> findByOrganizationId(Long orgId, Pageable pageable);
Slice<Monitor> findByStatus(MonitorStatus status, Pageable pageable);
List<Monitor>  findByStatus(MonitorStatus status, Pageable pageable);
```
```java
Pageable p = PageRequest.of(0, 20, Sort.by(Sort.Direction.DESC, "createdAt"));
Page<Monitor> page = repo.findByOrganizationId(orgId, p);

page.getContent();          page.getTotalElements();     // ← requires a COUNT query
page.getTotalPages();       page.hasNext();
```

**`Page` vs `Slice` vs `List`:**

| Return type | Queries issued | Gives you |
|---|---|---|
| `Page<T>` | **2** (data + `count(*)`) | Total elements and total pages |
| `Slice<T>` | **1** (fetches size+1 rows) | `hasNext()` only |
| `List<T>` | 1 | Just the rows |

**Use `Slice` for infinite scroll.** A `count(*)` over a large filtered table is expensive and — for an endless-scroll UI — completely unused. Dropping `Page` for `Slice` on high-volume endpoints is a free 50% query reduction. **Very few candidates mention this; it's a nice differentiator.**

### The `JOIN FETCH` + pagination trap

```java
// ⚠️ Generates a WARNING and paginates IN MEMORY
@Query("SELECT m FROM Monitor m JOIN FETCH m.incidents")
Page<Monitor> findAllWithIncidents(Pageable pageable);
```
```
HHH90003004: firstResult/maxResults specified with collection fetch; applying in memory
```

**Why:** joining a collection multiplies rows (one monitor with 5 incidents = 5 rows), so `LIMIT 20` would return the wrong number of *monitors*. Hibernate's only correct option is to fetch **everything** and paginate in memory — which can OOM your application.

**Fixes:**
1. Use `@EntityGraph` with a **to-one** association only (no multiplication, so pagination is safe)
2. Two-query approach: page the IDs first, then fetch the collections for that page
3. `@BatchSize` on the collection to reduce the N+1 to N/batch queries (Lesson 24)

**This warning appearing in your logs is a live production risk, not a nitpick.** Recognizing it is genuinely valuable.

---

## 6. Specifications — type-safe dynamic queries

For search screens with optional filters, string concatenation is unacceptable and 2^n derived methods is absurd.

```java
public interface MonitorRepository extends JpaRepository<Monitor, Long>,
                                           JpaSpecificationExecutor<Monitor> { }
```

```java
public final class MonitorSpecs {

    public static Specification<Monitor> inOrganization(Long orgId) {
        return (root, query, cb) -> cb.equal(root.get("organization").get("id"), orgId);
    }

    public static Specification<Monitor> hasStatus(MonitorStatus status) {
        return status == null ? null                                  // ★ null = no-op
                : (root, query, cb) -> cb.equal(root.get("status"), status);
    }

    public static Specification<Monitor> nameContains(String fragment) {
        return !StringUtils.hasText(fragment) ? null
                : (root, query, cb) -> cb.like(cb.lower(root.get("name")),
                                               "%" + fragment.toLowerCase() + "%");
    }

    public static Specification<Monitor> checkedBefore(Instant cutoff) {
        return cutoff == null ? null
                : (root, query, cb) -> cb.lessThan(root.get("lastCheckedAt"), cutoff);
    }
}
```

```java
public Page<MonitorSummary> search(Long orgId, MonitorSearchCriteria c, Pageable pageable) {
    Specification<Monitor> spec = Specification.allOf(          // Spring Data 3.x
            MonitorSpecs.inOrganization(orgId),                  // always applied
            MonitorSpecs.hasStatus(c.status()),                  // applied only if non-null
            MonitorSpecs.nameContains(c.q()),
            MonitorSpecs.checkedBefore(c.staleBefore()));
    return repo.findAll(spec, pageable).map(mapper::toSummary);
}
```

**Why this is the right tool:** each filter is an independently testable unit, `null` means "skip," composition is trivial, and it's type-safe against field *values* (though `root.get("status")` is still a string — use the JPA metamodel, `Monitor_.status`, generated by the Hibernate JPA Metamodel Generator, for full compile-time safety).

**The `inOrganization` line is a security pattern worth noting:** in a multi-tenant system, the tenant predicate must be **unconditional**, applied by the service, never derived from client input. A `Specification` composed from request parameters plus a mandatory tenant clause is a clean way to guarantee that (Lesson 28).

**Alternatives:**

| Tool | Verdict |
|---|---|
| **Specification** (Criteria API) | ✅ Built in, no extra dependency. Verbose lambdas. **The default.** |
| **Querydsl** | Nicer fluent API (`monitor.status.eq(DOWN)`), fully type-safe via generated Q-classes; needs an annotation processor and adds build complexity |
| **jOOQ** | Best-in-class typed SQL, superb for complex reporting. Commercial licence for some databases. Not an ORM. |
| **Query by Example** | `findAll(Example.of(probe))`. Trivially simple, but no ranges, no OR, no LIKE control. Rarely enough. |

If asked: *"Specifications by default since they're built in; Querydsl if the team wants a nicer API and accepts the codegen; jOOQ when reporting SQL is complex enough to deserve a real query DSL."*

---

## 7. Custom repository implementations

When you need `EntityManager` or `JdbcClient` directly, but want it on the same repository interface:

```java
// 1. Declare the extra methods
public interface MonitorRepositoryCustom {
    List<UptimeRow> uptimeReport(UUID orgId, Instant since);
    void bulkInsertCheckResults(List<CheckResult> results);
}

// 2. Implement — the class name MUST be <InterfaceName>Impl
@RequiredArgsConstructor
class MonitorRepositoryCustomImpl implements MonitorRepositoryCustom {
    private final EntityManager em;
    private final JdbcClient jdbc;

    @Override
    public List<UptimeRow> uptimeReport(UUID orgId, Instant since) {
        return jdbc.sql("""...""").param("orgId", orgId).query(UptimeRow.class).list();
    }
}

// 3. Extend both — callers see one repository
public interface MonitorRepository extends JpaRepository<Monitor, Long>,
                                           MonitorRepositoryCustom { }
```

**The `Impl` suffix is a hard convention** — get it wrong and Spring Data won't find your class, failing with a confusing "no property found for..." error as it tries to parse the method name as a derived query. That error message is a well-known head-scratcher, and knowing its cause is a nice piece of practical knowledge.

**This is the pattern for mixing JPA and JDBC cleanly** — exactly what Vigil needs (Lesson 19 §4).

---

## 8. Build this — Vigil's repository layer

**1. `MonitorRepository`:**
- `existsByOrganizationIdAndName` — for the duplicate-name check
- `findByOrganizationId(Long, Pageable)` returning a **projection**, not entities
- `@Query` with `JOIN FETCH m.organization` for the detail view
- `@Modifying` bulk status update for stale monitors, with **both flags set**
- `findByNextCheckAtBeforeAndStatusNot(...)` — the scheduler's query (Lesson 32)

**2. `CheckResultRepository extends Repository<CheckResult, Long>`** — deliberately *not* `JpaRepository`. Write a comment explaining that `findAll()` on this table would be an outage. Add:
- `findTop100ByMonitorIdOrderByCheckedAtDesc`
- a cursor query: `findByMonitorIdAndCheckedAtLessThanOrderByCheckedAtDesc(..., Limit.of(50))`
- a native aggregate for uptime % and p95 latency

**3. Projections, three ways.** Implement `MonitorSummaryView` (interface), `MonitorSummary` (record with a constructor expression), and a nested projection including the organization name. **Compare the generated SQL for each** in your logs — this is the exercise, not the code.

**4. Prove projections reduce the query.** Query with `Monitor` and again with `MonitorSummaryView`. Count the columns in the SQL log. Then make it an **open projection** with `@Value("#{target...}")` and watch it go back to selecting everything. **That's the trap, observed.**

**5. `Page` vs `Slice`.** Same query, both return types. **Count the queries.** Note the `count(*)` that `Page` adds, and think about that on a 50-million-row table.

**6. Reproduce the pagination + collection-fetch warning.** `@Query("SELECT m FROM Monitor m JOIN FETCH m.incidents")` returning `Page`. Find `HHH90003004` in your logs. Then fix it with `@EntityGraph` on a to-one association and note the warning disappears.

**7. Specifications.** Build `MonitorSpecs` with four filters, all null-tolerant. Wire a `/api/v1/monitors/search` endpoint taking optional query params. Test with 0, 1, and 4 filters and **read the generated SQL each time** — watch the WHERE clause grow.

**8. Custom implementation.** Add `MonitorRepositoryCustom` + `Impl` using `JdbcClient` for the uptime report. Then **deliberately misname the class** (`MonitorRepositoryCustomImplementation`) and read the startup error, so you recognize it forever.

**9. `@Modifying` staleness, demonstrated:**
```java
@Transactional
public void demo(Long id) {
    Monitor m = repo.findById(id).orElseThrow();
    log.info("before: {}", m.getStatus());
    repo.markDown(id);                          // bulk update, no clearAutomatically
    log.info("after:  {}", m.getStatus());      // ← STILL THE OLD VALUE
    Monitor again = repo.findById(id).orElseThrow();
    log.info("reread: {}", again.getStatus());  // ← ALSO OLD (first-level cache!)
}
```
Then add `clearAutomatically = true` and re-run. **Those two runs side by side are the clearest possible demonstration of the persistence context as a cache.**

---

## 9. Interview traps

**Q1. How does Spring Data create an implementation for your interface?**
`@EnableJpaRepositories` scans for `Repository` sub-interfaces and registers a `JpaRepositoryFactoryBean` for each. The factory instantiates `SimpleJpaRepository` for the standard methods, parses custom method names into queries, and builds a **JDK dynamic proxy** implementing your interface that routes calls appropriately. It's `FactoryBean` + reflection + proxies.

**Q2. `CrudRepository` vs `JpaRepository` vs `Repository` — which would you extend?**
`JpaRepository` is the pragmatic default. But extending bare `Repository` and declaring only the needed methods is more disciplined: it prevents an accidental `findAll()` or `deleteAll()` on a 50-million-row table. I'd use `Repository` for high-volume tables.

**Q3. When do you stop using derived query methods?**
Past 2–3 conditions the method name becomes unreadable, which defeats its purpose. Then: `@Query` for a fixed complex query, `Specification` for dynamic optional filters.

**Q4. JPQL vs native SQL?**
JPQL works on entities/fields, is database-portable, returns managed entities, and is **validated at application startup**. Native SQL is required for window functions, CTEs, `FILTER`, percentiles and vendor features, but isn't portable and only fails at execution time.

**Q5. What does `@Modifying` do, and what are `clearAutomatically`/`flushAutomatically` for?**
It marks a query as an update/delete rather than a select. Bulk JPQL updates bypass the persistence context, so loaded entities go stale and pending changes may not be written; `flushAutomatically` writes pending changes first, `clearAutomatically` evicts the context afterwards so subsequent reads see the new values. Also: `@Version` isn't incremented and no lifecycle callbacks fire. Requires a transaction.

**Q6. What are projections and why use them?**
Interfaces or records selecting a subset of columns. Hibernate generates a narrower SELECT, and nothing is managed or dirty-checked — less I/O, less memory, and no lazy-loading risk. Use them for every read-only endpoint. **Caveat:** open (SpEL) projections load the whole entity, defeating the purpose.

**Q7. `Page` vs `Slice`?**
`Page` runs a second `count(*)` query to provide total elements/pages. `Slice` fetches size+1 rows and only tells you whether there's a next page. For infinite scroll, `Slice` halves the queries and the count is unused anyway.

**Q8. Why does `JOIN FETCH` with `Pageable` produce a warning?**
Joining a collection multiplies rows, so `LIMIT` would return the wrong number of root entities. Hibernate therefore fetches all rows and paginates **in memory** — a potential OOM. Fix with `@EntityGraph` on to-one associations, a two-query ID-then-fetch approach, or `@BatchSize`.

**Q9. What's a `Specification` and when do you reach for it?**
A composable `Predicate` factory over the JPA Criteria API. Ideal for search endpoints with optional filters: each filter is independently testable, returns `null` to be skipped, and composes via `Specification.allOf`. It's also a clean way to guarantee an unconditional tenant predicate in a multi-tenant system.

**Q10. How do you add a hand-written query to a Spring Data repository?**
A custom fragment interface plus an implementation class named `<Fragment>Impl` (the suffix is mandatory), then have your repository extend both. This is how you mix `EntityManager`/`JdbcClient` code into the same repository — e.g. JPA for CRUD, `JdbcClient` for reporting.

**Q11. Why is `findByNameContaining` potentially a performance problem?**
It generates `LIKE '%value%'`. A leading wildcard can't use a standard B-tree index, so it's a full scan on large tables. Use Postgres full-text search (`tsvector` + GIN) or a trigram index (`pg_trgm`) for real search.

**Q12. `getReferenceById` vs `findById`?**
`getReferenceById` returns a lazy proxy **without querying** — ideal for setting a foreign key without loading the row. It throws `EntityNotFoundException` on first property access rather than at call time, so it's wrong when you need to confirm existence.

---

## 10. Retention check

1. Explain how `MonitorRepository` gets an implementation, naming three mechanisms from earlier lessons.
2. Why might you deliberately extend `Repository` instead of `JpaRepository`?
3. Give the two flags on `@Modifying` and the specific bug each prevents.
4. What do projections save, and which kind silently doesn't work?
5. Why does `Page` cost two queries, and when would you switch to `Slice`?
6. Explain `HHH90003004` and give two fixes.

---

**Next:** [Lesson 23 — Transactions deep dive](23-transactions-deep-dive.md) — propagation, isolation, rollback rules and locking. The highest-value interview topic in this entire course.

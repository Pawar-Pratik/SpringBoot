# Lesson 24 — Performance, N+1 & migrations

> **Why this lesson exists:** The N+1 problem is the most common performance bug in ORM-based applications, and it's asked in almost every interview involving JPA. Migrations are how you change a production schema without downtime or data loss — a topic tutorials skip entirely and production punishes immediately. Both are things you'll be measured on.

**Time:** ~85 minutes · **Prereq:** Lessons 20–23

---

## Part 1 — The idea in one sentence

> **Both halves of this lesson are about work that is invisible at the point you write it: `m.getOrganization().getName()` looks like a field access and is a network round trip, and `ALTER TABLE` looks instant and can freeze a table — so the discipline is making the invisible measurable, with query counts in tests and lock timeouts in migrations.**

That is also why both fail in production rather than in review. N+1 scales with your data, and a migration's lock contention scales with your traffic; neither shows up against five rows on a laptop.

---

## Part 2 — The N+1 problem

### What it is

```java
@Transactional(readOnly = true)
public List<MonitorSummary> listMonitors(Long orgId) {
    List<Monitor> monitors = repo.findByOrganizationId(orgId);      // 1 query → 100 monitors
    return monitors.stream()
            .map(m -> new MonitorSummary(m.getName(),
                                         m.getOrganization().getName()))  // 💥 1 query EACH
            .toList();
}
```

```sql
SELECT * FROM monitor WHERE org_id = ?;                 -- 1
SELECT * FROM organization WHERE id = ?;                -- 2
SELECT * FROM organization WHERE id = ?;                -- 3
...                                                      -- 101
```

**101 queries where 1 would do.** Each is fast (~1ms), so it looks fine in development with 5 rows. With 100 rows and a 2ms network round trip, you've added 200ms. With 1,000 rows across a nested association, you've added seconds.

**Why it's so dangerous:** it's *invisible in the code*. `m.getOrganization().getName()` looks like a field access. Nothing in the Java hints at a network round trip. And it scales with your data, so it appears in production long after the code shipped.

### Every way it hides

```java
// 1. Lazy to-one in a loop — the classic
monitors.forEach(m -> log.info(m.getOrganization().getName()));

// 2. Lazy collection in a loop
monitors.forEach(m -> log.info("{}", m.getIncidents().size()));

// 3. Returning entities from a controller with open-in-view=true  ← queries in the VIEW layer
@GetMapping public List<Monitor> list() { return repo.findAll(); }

// 4. Stream/flatMap over a lazy collection — functional style hides it beautifully
orgs.stream().flatMap(o -> o.getMonitors().stream()).toList();

// 5. equals/hashCode touching an association (Lombok @Data on an entity!)
new HashSet<>(monitors);

// 6. toString() on an entity with associations, inside a log statement
log.debug("monitors: {}", monitors);       // ← the log statement causes 100 queries

// 7. @ManyToOne EAGER — fires N queries at load time, before you touch anything
```

**#6 deserves a highlight:** a `log.debug` can be the *cause* of an N+1. And if debug logging is off in production, the queries don't happen — so the bug appears only when someone enables debug logging to investigate a different problem. That's a genuinely nasty story worth telling.

### Detection — how you *find* it, which matters more than the theory

**1. Log the SQL. Always, in development.**
```properties
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.orm.jdbc.bind=TRACE
```

**2. Count the queries automatically.** This is the professional approach:
```xml
<dependency>
    <groupId>net.ttddyy.observation</groupId>
    <artifactId>datasource-micrometer-spring-boot</artifactId>
    <version>1.0.5</version>
</dependency>
```
Or the classic `datasource-proxy` / `p6spy`. Then **assert query counts in your tests:**

```java
@Test
void listMonitorsUsesOneQuery() {
    var stats = entityManagerFactory.unwrap(SessionFactory.class).getStatistics();
    stats.clear();

    monitorService.listMonitors(orgId);

    assertThat(stats.getPrepareStatementCount())
            .as("N+1 regression check")
            .isEqualTo(1);
}
```

**This test is the single most valuable thing in this lesson.** N+1 problems are *regressions* — someone adds an innocent `m.getOrganization().getName()` to a mapper six months later and it's back. A query-count assertion catches it in CI. **Very few candidates mention testing for N+1; it's a standout answer.**

**3. Hibernate statistics in development:**
```properties
spring.jpa.properties.hibernate.generate_statistics=true
```

**4. Fail fast on lazy loading outside a transaction.** Hibernate's `enable_lazy_load_no_trans` should be **`false`** (the default) — some people enable it to "fix" `LazyInitializationException`, which converts an error into a silent N+1 in an unpredictable place. Never enable it.

**5. In production:** APM tools (Datadog, New Relic) show "100 identical queries in one trace" clearly. Or watch your own `spring.data.repository.invocations` and datasource query metrics (Lesson 35).

### The fixes, with trade-offs

#### Fix 1 — `JOIN FETCH` (explicit, per query)

```java
@Query("SELECT m FROM Monitor m JOIN FETCH m.organization WHERE m.organization.id = :orgId")
List<Monitor> findByOrgWithOrganization(@Param("orgId") Long orgId);
```
```sql
SELECT m.*, o.* FROM monitor m JOIN organization o ON m.org_id = o.id WHERE m.org_id = ?
```
**One query.** ✅

**Multiple collections is the trap:**
```java
// ❌ MultipleBagFetchException, or a Cartesian product
@Query("SELECT m FROM Monitor m JOIN FETCH m.incidents JOIN FETCH m.checkResults")
```
Fetching two collections multiplies rows: 10 incidents × 20 results = 200 rows for one monitor. Hibernate refuses (`MultipleBagFetchException`) when both are `List`. Using `Set` avoids the exception but **gives you the Cartesian product anyway** — you've traded an error for a silent performance disaster.

**Fixes:** fetch one collection per query (two queries, `DISTINCT`-free and correct), or use `@BatchSize`/`SUBSELECT`, or use `Set` plus `hibernate.query.passDistinctThrough`. **Knowing that `Set` "fixes" the exception without fixing the problem is a strong detail.**

#### Fix 2 — `@EntityGraph` (declarative, reusable)

```java
@EntityGraph(attributePaths = {"organization", "notificationChannels"})
List<Monitor> findByOrganizationId(Long orgId);

@EntityGraph(attributePaths = {"organization"})
@Override
Optional<Monitor> findById(Long id);            // override the inherited method!
```

Or named on the entity:
```java
@NamedEntityGraph(name = "Monitor.withOrgAndIncidents",
    attributeNodes = {
        @NamedAttributeNode("organization"),
        @NamedAttributeNode(value = "incidents", subgraph = "incidentDetail")
    },
    subgraphs = @NamedSubgraph(name = "incidentDetail",
                               attributeNodes = @NamedAttributeNode("acknowledgedBy")))
@Entity
public class Monitor { }
```
```java
@EntityGraph("Monitor.withOrgAndIncidents")
Optional<Monitor> findById(Long id);
```

**`@EntityGraph` is usually the better tool than `JOIN FETCH`** because it composes with derived queries, `Pageable`, and `Specification` — and it keeps the fetch strategy separate from the query logic. It's also the answer to "how do you fetch different graphs for different use cases from the same entity?"

#### Fix 3 — `@BatchSize` (the safety net)

```java
@Entity
public class Monitor {
    @ManyToOne(fetch = LAZY)
    @BatchSize(size = 25)
    private Organization organization;
}
```
Or globally:
```properties
spring.jpa.properties.hibernate.default_batch_fetch_size=25
```
```sql
-- Instead of 100 queries:
SELECT * FROM organization WHERE id IN (?,?,?,...,?)   -- 4 queries for 100 monitors
```

**Set `default_batch_fetch_size` in every project.** It doesn't *solve* N+1, but it turns 100 queries into 4 — a global safety net for the N+1s you haven't found yet. **This one property is one of the highest-value JPA settings that almost nobody sets**, and recommending it is a memorable answer.

#### Fix 4 — Projections (best for read paths)

```java
public interface MonitorSummaryView {
    String getName();
    OrgView getOrganization();
    interface OrgView { String getName(); }
}
```
One query, only the needed columns, no entities, **N+1 is structurally impossible** because there's no lazy association to traverse. **For read-only endpoints this is the correct answer, not a workaround** (Lesson 22 §4).

#### Fix 5 — Manual two-query fetch (for complex cases)

```java
List<Monitor> monitors = repo.findByOrganizationId(orgId);
Set<Long> orgIds = monitors.stream().map(Monitor::getOrgId).collect(toSet());
Map<Long, Organization> orgs = orgRepo.findAllById(orgIds).stream()
        .collect(toMap(Organization::getId, identity()));
// join in memory
```
Two queries, total control, no ORM surprises. Sometimes the clearest option — especially across aggregate boundaries or when the data comes from different services.

### The fix NOT to use

```java
@ManyToOne(fetch = FetchType.EAGER)     // ❌
```
It "fixes" one call site by degrading every other query on that entity — and EAGER cannot be overridden per query, while LAZY can be upgraded with a fetch join. **Strictly worse in every case.** (Lesson 20 §5.)

---

## Part 3 — Query and index performance

### Read the query plan

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM check_result
WHERE monitor_id = '...' AND checked_at >= now() - interval '7 days'
ORDER BY checked_at DESC LIMIT 100;
```

**What to look for:**

| Sign | Meaning |
|---|---|
| `Seq Scan` on a large table | **No usable index.** The main thing you're hunting. |
| `Index Scan` / `Index Only Scan` | ✅ Good. `Index Only` is best — it never touches the heap. |
| `Bitmap Heap Scan` | Index used, many rows matched — often fine |
| `rows=1000` estimate vs `actual rows=500000` | **Stale statistics.** Run `ANALYZE`. Bad estimates cause bad plans. |
| `Nested Loop` with a large outer | Often a bad join order caused by bad estimates |
| `Sort` with `external merge Disk` | Sorting spilled to disk — raise `work_mem` or add an index that provides the order |

**`EXPLAIN ANALYZE` actually runs the query** (wrap it in a transaction and roll back if it writes). `EXPLAIN` alone only plans.

### Indexing rules that matter

```sql
-- 1. Index your foreign keys. Postgres does NOT do this automatically.
CREATE INDEX idx_check_result_monitor ON check_result (monitor_id);
```
**Postgres creates an index for a PRIMARY KEY and UNIQUE constraint, but not for a FOREIGN KEY.** Missing FK indexes make joins and — crucially — **cascading deletes** catastrophically slow. This is one of the most common real-world schema mistakes.

```sql
-- 2. Composite index column order: equality first, then range/sort
CREATE INDEX idx_cr_monitor_time ON check_result (monitor_id, checked_at DESC);
```
This serves `WHERE monitor_id = ? ORDER BY checked_at DESC` perfectly — equality narrows, then the index already provides the order, so **no sort step at all**. Reverse the columns and it's nearly useless for this query. **Column order in a composite index is the thing people get wrong most.**

The **leftmost prefix rule:** an index on `(a, b, c)` serves queries on `(a)`, `(a,b)`, `(a,b,c)` — but **not** `(b)` or `(b,c)` alone.

```sql
-- 3. Covering index — an INDEX ONLY SCAN, never touching the table
CREATE INDEX idx_cr_covering ON check_result (monitor_id, checked_at DESC)
    INCLUDE (status, response_time_ms);

-- 4. Partial index — smaller and faster when you only query a subset
CREATE INDEX idx_monitor_active ON monitor (next_check_at)
    WHERE status <> 'PAUSED';
```
**The partial index is exactly right for Vigil's scheduler**, which only ever queries non-paused monitors due for a check. It's a fraction of the size of a full index and stays hot in cache.

```sql
-- 5. Build indexes without locking writes in production
CREATE INDEX CONCURRENTLY idx_name ON table (col);
```
**Non-negotiable in production.** A plain `CREATE INDEX` takes a lock that blocks writes for the duration — minutes on a large table. `CONCURRENTLY` doesn't (it's slower and can't run inside a transaction, and can leave an invalid index if it fails, which you then drop and retry). **Flyway needs `-- @flyway:executeInTransaction=false` for these.** Knowing this specific detail is a strong production signal.

**The cost of indexes** — state it, because people forget: every index slows INSERT/UPDATE/DELETE and consumes storage and cache. **Unused indexes are pure cost.** Find them:
```sql
SELECT relname, indexrelname, idx_scan FROM pg_stat_user_indexes WHERE idx_scan = 0;
```

### JPA-specific performance settings

```yaml
spring:
  jpa:
    open-in-view: false                          # ★ Lesson 20
    properties:
      hibernate:
        default_batch_fetch_size: 25             # ★ global N+1 safety net
        jdbc:
          batch_size: 50                         # batch INSERTs/UPDATEs
          fetch_size: 100                        # JDBC rows per round trip
        order_inserts: true                      # ★ required for batching to be effective
        order_updates: true
        batch_versioned_data: true               # batch updates on @Version entities
        query:
          in_clause_parameter_padding: true      # ★ pads IN lists to powers of 2 → plan cache reuse
        generate_statistics: false               # true in dev only
```

**`in_clause_parameter_padding` is a lovely obscure win:** without it, `IN (?,?,?)` and `IN (?,?,?,?)` are different SQL strings, so each distinct list length gets its own cached plan and you thrash the plan cache. Padding to powers of two (2, 4, 8, 16) collapses that to a handful of shapes.

**Bulk operations — the pattern:**
```java
@Transactional
public void importMonitors(List<CreateMonitorRequest> requests) {
    for (int i = 0; i < requests.size(); i++) {
        em.persist(toEntity(requests.get(i)));
        if (i % 50 == 0) {
            em.flush();      // write the batch
            em.clear();      // ★ CRITICAL — free the persistence context or you OOM
        }
    }
}
```
**Without `em.clear()`, the persistence context grows to hold every entity plus a dirty-check snapshot of each — and dirty checking becomes O(n²).** For 100,000 rows you'll OOM. **Honestly though: for real bulk loads, use `JdbcClient.batchUpdate` (Lesson 19) or Postgres `COPY`.** JPA is the wrong tool for bulk insertion, and saying so is the right answer.

### Caching layers (preview of Lesson 33)

| Layer | Scope | Notes |
|---|---|---|
| **First-level cache** | Persistence context / transaction | Always on, not optional |
| **Second-level cache** | Across transactions, per JVM | Opt-in (`@Cache` + a provider). **Beware in multi-instance deployments** — each JVM has its own, so they diverge |
| **Query cache** | Query results | Easy to get wrong; invalidated by any write to the involved tables. Usually not worth it |
| **Application cache** | Redis, shared | ✅ **Usually the right choice** — explicit, shared, TTL-controlled |

**The senior position: skip Hibernate's second-level cache in a multi-instance deployment and use an explicit Redis cache instead.** Reason: L2 is per-JVM, so with 5 pods you have 5 caches that silently disagree, and invalidation across them requires a distributed cache provider that adds more complexity than it saves. Explicit `@Cacheable` on a shared Redis gives you one source of truth and a visible TTL. That's a well-reasoned answer, not a dismissal.

---

## Part 4 — Database migrations

### Why `ddl-auto` is not a migration strategy

```yaml
spring.jpa.hibernate.ddl-auto: none | validate | update | create | create-drop
```

| Value | Behaviour | Use |
|---|---|---|
| `none` | Nothing | ✅ Production (with Flyway) |
| **`validate`** | Verify entities match the schema; **fail startup on mismatch** | ✅ **Production — best choice** |
| `update` | Add missing tables/columns | ❌ **Never in production** |
| `create` | Drop and recreate | Local scratch only |
| `create-drop` | Create, then drop at shutdown | Tests |

**Why `update` is dangerous, specifically:**
- It **never drops or renames** anything — removing a field leaves an orphan column forever
- It can't do data migrations, and a type change becomes an incompatible column or a silent no-op
- The result depends on the *current* schema state, so it's **not reproducible** — two environments diverge
- **No version history, no review, no rollback plan.** You cannot tell what it will do before it does it.
- Multiple instances starting simultaneously race to alter the schema

**`validate` is the production setting**, and it's genuinely valuable: it fails startup if your entities and schema have drifted, catching a forgotten migration at deploy time instead of at first query.

### Flyway

```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-database-postgresql</artifactId>
</dependency>
```

```
src/main/resources/db/migration/
├── V1__create_organization_and_user.sql
├── V2__create_monitor.sql
├── V3__create_check_result.sql
├── V4__add_monitor_next_check_index.sql
├── V5__add_incident_tables.sql
└── R__refresh_uptime_view.sql          ← Repeatable: re-runs when its checksum changes
```

**Naming: `V<version>__<description>.sql` — two underscores.** One underscore is a silent no-op that leaves you confused.

```yaml
spring:
  flyway:
    enabled: true
    baseline-on-migrate: true          # for adopting Flyway on an existing database
    validate-on-migrate: true
    out-of-order: false                # ★ keep false — enforces linear history
  jpa:
    hibernate:
      ddl-auto: validate               # ★ Flyway owns the schema; Hibernate only checks
```

**How it works:** Flyway creates a `flyway_schema_history` table recording each applied version and a **checksum** of the file. On startup it applies pending migrations in order, inside a transaction where the database supports transactional DDL (Postgres does; MySQL largely doesn't).

**The checksum is the important part: editing an already-applied migration fails validation.** That's deliberate — it guarantees every environment ran identical SQL. **Never edit an applied migration; write a new one.** This is the #1 Flyway rule and a guaranteed interview question.

### Flyway vs Liquibase

| | Flyway | Liquibase |
|---|---|---|
| Format | Plain SQL (or Java) | XML/YAML/JSON/SQL changelogs |
| Learning curve | Minimal — it's just SQL | Higher |
| Database portability | You write DB-specific SQL | Abstracted DDL, generates per-dialect |
| Rollback | Paid feature / write your own down script | Built-in rollback support |
| Diff generation | ❌ | ✅ can diff schemas |
| Popularity in Spring | More common | Common in enterprise |

**Choose Flyway** unless you need genuine multi-database support or built-in rollback. *"Flyway, because plain SQL means anyone can review the migration and there's no abstraction hiding what runs against production"* is a solid answer.

### Zero-downtime migrations — the senior skill

During a rolling deploy, **old and new application code run simultaneously** against **one database**. Every migration must therefore be compatible with both.

**The rule: never make a breaking schema change in a single deploy.**

#### Renaming a column — the expand/contract pattern

```
❌ WRONG:  ALTER TABLE monitor RENAME COLUMN url TO target_url;
           → old pods still SELECT url → immediate 500s for every request
```

```
✅ RIGHT — four deploys:

Deploy 1 (expand):    ALTER TABLE monitor ADD COLUMN target_url text;
                      Code writes BOTH columns, reads `url`.

Deploy 2 (backfill):  UPDATE monitor SET target_url = url WHERE target_url IS NULL;
                      (batched, so you don't lock the table)
                      Code writes both, reads `target_url`.

Deploy 3:             Code writes and reads only `target_url`.

Deploy 4 (contract):  ALTER TABLE monitor DROP COLUMN url;
```

Tedious, and it is genuinely how it's done. **Being able to describe expand/contract from memory is one of the clearest senior signals in a database interview.**

#### Adding a NOT NULL column

```sql
-- ❌ Fails if the table has rows; and on older Postgres, rewrites the whole table
ALTER TABLE monitor ADD COLUMN region text NOT NULL;

-- ✅ Postgres 11+: a non-volatile DEFAULT is metadata-only — instant, no rewrite
ALTER TABLE monitor ADD COLUMN region text NOT NULL DEFAULT 'us-east-1';

-- ✅ Portable three-step for large tables:
-- 1. ADD COLUMN region text;               (nullable, instant)
-- 2. backfill in batches
-- 3. ALTER COLUMN region SET NOT NULL;     (a validating scan, but no rewrite)
```

#### The lock-duration rules

| Operation | Postgres lock | Safe? |
|---|---|---|
| `ADD COLUMN` (nullable, or with a constant default, PG11+) | brief `ACCESS EXCLUSIVE` | ✅ |
| `ADD COLUMN ... NOT NULL` without default | table rewrite | ❌ on large tables |
| `CREATE INDEX` | blocks writes for the duration | ❌ → use `CONCURRENTLY` |
| `ALTER COLUMN TYPE` | full rewrite | ❌ → new column + backfill + swap |
| `ADD CONSTRAINT ... CHECK` | validating scan, blocks writes | → `NOT VALID`, then `VALIDATE CONSTRAINT` |
| `ADD FOREIGN KEY` | locks **both** tables | → `NOT VALID`, then validate |
| `DROP COLUMN` | metadata only | ✅ instant |

**And the one that catches people:** *any* `ALTER TABLE` needs an `ACCESS EXCLUSIVE` lock, and if a long-running query holds a conflicting lock, your ALTER waits — **and every subsequent query queues behind your waiting ALTER.** A "quick" migration can freeze the table. **Always set a `lock_timeout` so it fails fast instead of causing a pile-up:**

```sql
SET lock_timeout = '3s';
ALTER TABLE monitor ADD COLUMN region text;
```

**That `lock_timeout` line is one of the most valuable single pieces of advice in this lesson**, and mentioning it unprompted marks you as someone who has actually done this.

### Migration discipline

**✅ Do:**
- One logical change per migration
- Make them idempotent where possible (`IF NOT EXISTS`)
- Review migrations like code — they're irreversible in production
- **Test on a copy of production-sized data**, not an empty dev database
- Set `lock_timeout` and `statement_timeout` in migration scripts
- Separate schema changes from data backfills (backfills batched, often as a job, not a migration)
- Keep migrations **forward-only**; recover by rolling forward

**❌ Don't:**
- Edit an applied migration (checksum failure, and environments diverge)
- Put a multi-million-row `UPDATE` in a startup migration — it delays or blocks the deploy
- Use `ddl-auto: update`
- Rely on rollback scripts as your safety net — a `DROP COLUMN` rollback loses data. **Backups plus forward fixes are the real strategy.**

---

## Part 5 — Multi-tenancy (Vigil is multi-tenant)

Three approaches, and you should be able to compare them:

| Strategy | Isolation | Complexity | Scale |
|---|---|---|---|
| **Discriminator column** (`org_id` on every table) | Weakest — a code bug leaks data | Lowest | ✅ Best; one connection pool, one schema |
| **Schema per tenant** | Better | Medium — N schemas to migrate | Hundreds of tenants |
| **Database per tenant** | Strongest | Highest — N databases, N pools, N migrations | Few, large, regulated tenants |

**Vigil uses a discriminator column,** which is right for a SaaS with many small tenants — and it means **every query must filter by `org_id`, and a single missed filter is a cross-tenant data breach.**

**Defence in depth, and this is the part worth getting right:**

```java
// Layer 1 — Hibernate filter, applied automatically per session
@Entity
@FilterDef(name = "tenantFilter",
           parameters = @ParamDef(name = "orgId", type = UUID.class))
@Filter(name = "tenantFilter", condition = "org_id = :orgId")
public class Monitor { }
```
```java
// enable it per request/transaction
em.unwrap(Session.class).enableFilter("tenantFilter").setParameter("orgId", currentOrgId());
```

```java
// Layer 2 — an unconditional tenant predicate in every Specification (Lesson 22)
Specification.allOf(MonitorSpecs.inOrganization(currentOrgId()), ...userFilters);
```

```sql
-- Layer 3 — Postgres Row-Level Security: the database itself enforces it
ALTER TABLE monitor ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON monitor
    USING (org_id = current_setting('app.current_org')::uuid);
```

```java
// Layer 4 — an ArchUnit or integration test that FAILS if any repository method
// can be called without a tenant filter
```

**Row-Level Security is the strongest option** because it holds even if your application code has a bug — the database refuses to return other tenants' rows. It costs a `SET app.current_org` per connection (which interacts awkwardly with connection pooling, so you set it per transaction) and makes some queries harder to plan. **Discussing RLS in a multi-tenancy answer is a strong differentiator** — most candidates stop at "add a WHERE clause."

---

## Part 6 — Build & break

**1. ⭐ Create, observe, and fix an N+1.** Seed 100 monitors across 5 organizations. Write the naive `listMonitors` from §1. **Count the SQL statements in your log — you should see 101.** Then fix it five ways and count each time:
- `JOIN FETCH`
- `@EntityGraph`
- `@BatchSize(25)` — note it's 5 queries, not 1, but 20× better than 101
- an interface projection with a nested `OrgView`
- manual two-query with an in-memory join

**Write the query counts in a table.** That table is a great thing to put in Vigil's README, and it's a great thing to describe in an interview.

**2. Write the N+1 regression test** from §1 using Hibernate `Statistics`. Then break the fix (add a `getOrganization().getName()` somewhere) and watch the test fail. **This is the exercise that makes you the person who prevents N+1 rather than fixing it.**

**3. Set the global safety net.** Add `default_batch_fetch_size: 25` and re-run your worst N+1 case without any other fix. Note the improvement from doing nothing but setting a property.

**4. Reproduce `MultipleBagFetchException`.** Two `JOIN FETCH`es on `List` collections. Read the error. Then change both to `Set` and **count the rows returned** — confirm you now have a Cartesian product instead of an exception. **Feel that this is not a fix.**

**5. Indexes and `EXPLAIN`.** Insert 500,000 `check_result` rows (use `JdbcClient.batchUpdate` — you'll appreciate Lesson 19 now):
```sql
EXPLAIN ANALYZE SELECT * FROM check_result
WHERE monitor_id = '...' ORDER BY checked_at DESC LIMIT 100;
```
- Run it with **no index** → note `Seq Scan` and the timing
- Add `(checked_at)` only → still poor
- Add `(monitor_id, checked_at DESC)` → **`Index Scan`, no sort step.** Compare the numbers.
- Add `INCLUDE (status, response_time_ms)` → look for `Index Only Scan`
- Create a **partial index** for the scheduler query and compare index sizes with `\di+`

**Record the before/after timings.** Concrete numbers are far more persuasive than "I know about indexes."

**6. Set up Flyway properly.** Convert your hand-written schema into `V1__…` through `V5__…` migrations. Set `ddl-auto: validate`. Drop the database, start the app, and watch Flyway build everything from scratch. Then inspect `flyway_schema_history`.

**7. Prove the checksum rule.** Edit an applied migration file and restart. **Read the validation failure.** Then revert it and add a new `V6__` instead.

**8. Prove `validate` catches drift.** Add a field to an entity with no matching migration. Read the startup failure. **That failure is your friend** — it caught a forgotten migration before production did.

**9. ⭐ Do an expand/contract rename.** Rename `monitor.url` to `monitor.target_url` across four migrations plus the corresponding code changes, exactly as in §3. **Actually do all four steps.** It's tedious, and doing it once means you can describe it fluently forever.

**10. Feel a lock.** In one psql session: `BEGIN; SELECT * FROM monitor;` (leave it open). In another: `ALTER TABLE monitor ADD COLUMN x text;`. **Watch it block.** Then in a third session try `SELECT * FROM monitor` and watch *that* block too — behind your waiting ALTER. **That's the pile-up.** Now add `SET lock_timeout = '2s';` before the ALTER and watch it fail fast instead.

**11. Bulk insert three ways** (revisit Lesson 19): JPA `save()` in a loop, JPA with `flush`+`clear` every 50, and `JdbcClient.batchUpdate`. Time 50,000 rows each and watch the memory in a profiler. Then try JPA *without* `clear()` and watch the heap climb.

**12. Multi-tenancy.** Add `org_id` to every table. Implement the tenant `Specification` predicate. Then write an integration test that creates data for two organizations and **asserts that org A's queries never return org B's rows.** Then deliberately remove the tenant filter from one repository method and confirm the test fails. **That test is a security control.**

### Explain out loud (60s)

1. What N+1 is, and the three places it hides that aren't a loop you wrote.
2. How you'd *detect* it, and how you'd stop it coming back.
3. Why `EAGER` is not a fix.
4. The expand/contract sequence for renaming a column, deploy by deploy.
5. Why `lock_timeout` matters more than the duration of the ALTER itself.

---

## Part 7 — Production rules

| Do / Never | Why |
|---|---|
| **Log Hibernate's SQL in development and never turn it off** | N+1 is invisible in Java and obvious in the log — you cannot find what you never look at |
| **Assert query counts in tests for every list endpoint** | It turns a performance bug into a failing build; without it, the regression returns the first time someone adds a field to a DTO |
| **Fetch associations explicitly with a join fetch or an entity graph** | The fix belongs in the query, where it is visible and per-call, not in a mapping that affects every other query |
| **Never "fix" N+1 by switching an association to `EAGER`** | It can't be overridden per query, so it repairs one call site by degrading every other query that touches the entity |
| **Watch for N+1 hiding in a `log.debug`** | With debug off the queries don't happen, so the bug appears only when someone enables logging to investigate something else |
| **Set `open-in-view: false`** | Otherwise lazy getters fire during serialization, which puts your queries in the view layer where no profiler is looking |
| **Never use `ddl-auto: update` as a migration strategy** | It never drops or renames, applies silently in whatever order Hibernate chooses, and leaves environments quietly divergent |
| **Name Flyway migrations `V<version>__<description>.sql` with two underscores** | One underscore is a silent no-op, so the migration simply never runs and nothing tells you |
| **Never edit an applied migration; write a new one** | Flyway checksums each file precisely so every environment is guaranteed to have run identical SQL |
| **Never make a breaking schema change in one deploy** | Old and new code run simultaneously during a rolling deploy — expand, migrate, then contract across releases |
| **Set `lock_timeout` before any `ALTER TABLE`** | An ALTER waits on a conflicting lock, and every later query queues behind the waiting ALTER — a "quick" migration freezes the table |
| **Never put a multi-million-row `UPDATE` in a startup migration** | The app can't start until it finishes, so your deploy becomes an outage; backfill in batches from a job instead |

---

## Part 8 — Interview traps

**Q1. What is the N+1 problem?**
One query loads N parents, then accessing a lazy association on each fires one query per parent — N+1 total. It's dangerous because `entity.getX()` looks like a field access, so it's invisible in the code and scales with data, appearing in production long after the code shipped.

**Q2. How do you detect it?**
SQL logging in development; Hibernate `Statistics` or a datasource proxy to **count queries in tests** (the real answer — N+1 is a regression, so assert query counts in CI); `generate_statistics` in dev; and APM traces showing repeated identical queries in production.

**Q3. How do you fix it, and what's wrong with `EAGER`?**
`JOIN FETCH`, `@EntityGraph`, `@BatchSize`/`default_batch_fetch_size`, projections, or a manual two-query in-memory join. `EAGER` is the wrong fix: it degrades every other query on the entity and **can't be overridden per query**, whereas LAZY can be upgraded with a fetch join.

**Q4. What is `MultipleBagFetchException`, and does using `Set` fix it?**
It's thrown when you `JOIN FETCH` two `List` collections, because the join multiplies rows. Switching to `Set` **removes the exception but not the Cartesian product** — you've hidden a performance disaster. Real fixes: one collection per query, or `@BatchSize`/subselect fetching.

**Q5. `JOIN FETCH` vs `@EntityGraph`?**
Same effect; `@EntityGraph` is declarative and composes with derived queries, `Pageable`, and `Specification`, keeping fetch strategy separate from query logic. `JOIN FETCH` is inline in JPQL. Prefer `@EntityGraph` for reusable fetch plans.

**Q6. Why is `ddl-auto: update` unacceptable in production?**
It never drops or renames, can't migrate data, produces schema that depends on current state (so environments diverge), has no version history/review/rollback, and multiple instances race on startup. Use Flyway with `ddl-auto: validate` so drift fails the deploy.

**Q7. What's the cardinal rule of Flyway?**
Never edit an applied migration — Flyway checksums each file and validation will fail, and worse, environments would have run different SQL. Always add a new migration.

**Q8. How do you rename a column with zero downtime?**
Expand/contract over four deploys: add the new column and write both; backfill in batches and switch reads; stop writing the old column; drop it. Required because during a rolling deploy old and new code run against the same database simultaneously.

**Q9. Why must you use `CREATE INDEX CONCURRENTLY` in production?**
A plain `CREATE INDEX` holds a lock blocking writes for its duration — minutes on a large table. `CONCURRENTLY` avoids that (slower, can't run in a transaction, may leave an invalid index to drop and retry). In Flyway, mark the migration `executeInTransaction=false`.

**Q10. Why set `lock_timeout` in migrations?**
Any `ALTER TABLE` needs `ACCESS EXCLUSIVE`. If a long query holds a conflicting lock, the ALTER waits — and **every subsequent query queues behind it**, freezing the table. A short `lock_timeout` makes the migration fail fast and retry instead of causing an outage.

**Q11. Composite index column order — how do you choose?**
Equality predicates first, then range/sort columns: `(monitor_id, checked_at DESC)` serves `WHERE monitor_id = ? ORDER BY checked_at DESC` with no sort step. Remember the leftmost-prefix rule: `(a,b,c)` doesn't help a query filtering only on `b`.

**Q12. Does Postgres index foreign keys automatically?**
**No.** It indexes primary keys and unique constraints only. Missing FK indexes make joins and cascading deletes extremely slow — one of the most common real schema mistakes.

**Q13. Would you use Hibernate's second-level cache?**
Usually not in a multi-instance deployment: it's per-JVM, so N pods hold N caches that silently diverge, and cross-instance invalidation needs a distributed provider that costs more complexity than it saves. I'd use an explicit shared cache (Redis via `@Cacheable`) with a visible TTL and deliberate invalidation.

**Q14. How would you implement multi-tenancy?**
Discriminator column for many small tenants (best scale, weakest isolation), schema-per-tenant for hundreds, database-per-tenant for few large regulated tenants. With a discriminator, defend in depth: a Hibernate `@Filter` or an unconditional `Specification` predicate, **Postgres Row-Level Security** so the database enforces it even if app code has a bug, and integration tests asserting cross-tenant queries return nothing.

**Q15. Why does bulk `save()` in JPA cause an OutOfMemoryError?**
Every persisted entity stays in the persistence context with a dirty-check snapshot, so memory grows linearly and dirty checking becomes O(n²). Fix with periodic `flush()` + `clear()` — or better, use `JdbcClient.batchUpdate` or `COPY`, because JPA is the wrong tool for bulk loading.

---

## Part 9 — Retention check

1. Give five ways an N+1 can hide, including the one involving a log statement.
2. Why is a query-count test more valuable than knowing how to fix an N+1?
3. Why is `Set` not a real fix for `MultipleBagFetchException`?
4. Explain expand/contract in four steps.
5. Why does a waiting `ALTER TABLE` freeze reads, and what's the one-line mitigation?
6. Give the composite-index column-order rule and an example.

---

## Module 6 complete ✅

**This was the hardest module in the course.** You now understand persistence from the connection pool up: pooling, the persistence context, mapping, querying, transactions, and performance. This is the area where most Spring developers are weakest, so it's where your depth will be most visible.

**Next:** [Lesson 25 — Spring Security architecture](../07-security/25-security-architecture.md) — the filter chain, and how to debug a 401 vs a 403 in under a minute.

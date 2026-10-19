# Lesson 19 — JDBC, DataSources & connection pooling

> **Why this lesson exists:** Everyone jumps straight to JPA and then can't explain why their app hangs under load. JPA sits on JDBC, and JDBC sits on a connection pool with a *finite* number of connections. Pool exhaustion is one of the most common production incidents in Java, and it's invisible unless you understand this layer. Also: `JdbcClient` is genuinely the right tool for a lot of work, and knowing when to skip the ORM is a senior signal.

**Time:** ~60 minutes · **Prereq:** Module 5

---

## 1. The idea in one sentence

> **Your service's real concurrency limit isn't the 200 Tomcat threads — it's the ten connections in the pool, and a queue in front of a fast database beats a mob inside a slow one.**

Queueing at the pool is admission control, not a bug. Once you see pool size as the number that keeps the database inside its efficient operating range, the counterintuitive sizing advice, the multi-pod arithmetic and most "the app hangs under load" incidents all explain themselves.

---

## 2. JDBC — the foundation

**JDBC (Java Database Connectivity)** is the standard Java API for relational databases. Every higher-level tool — Hibernate, Spring Data, MyBatis, jOOQ — ultimately calls JDBC.

Raw JDBC, so you know what's being hidden from you:

```java
String sql = "SELECT id, name, url FROM monitor WHERE org_id = ? AND status = ?";

try (Connection conn = dataSource.getConnection();                   // ← from the POOL
     PreparedStatement ps = conn.prepareStatement(sql)) {

    ps.setObject(1, orgId);
    ps.setString(2, status.name());

    try (ResultSet rs = ps.executeQuery()) {
        List<Monitor> monitors = new ArrayList<>();
        while (rs.next()) {
            monitors.add(new Monitor(
                rs.getObject("id", UUID.class),
                rs.getString("name"),
                rs.getString("url")));
        }
        return monitors;
    }
} catch (SQLException e) {
    throw new DataAccessException("query failed", e);      // checked exception, must handle
}
```

**The five problems this shows** (and each is something Spring solves):

1. **Resource management.** Forget to close a `Connection` and you leak a pool slot. Leak enough and the pool is dead. `try-with-resources` helps, but it's on you.
2. **Checked `SQLException` everywhere.** Every caller must handle or declare it, for a condition it usually can't fix (Lesson 02).
3. **Manual mapping.** Column-to-field code, by hand, for every query.
4. **Vendor-specific error codes.** A unique-constraint violation is SQLState `23505` on Postgres, error `1062` on MySQL. Portable error handling is impossible without a translation layer.
5. **Enormous boilerplate.** Nine lines of ceremony around one line of intent.

### `PreparedStatement` and SQL injection

```java
// ❌ SQL INJECTION. This is not theoretical — it's the #1 web vulnerability historically.
String sql = "SELECT * FROM monitor WHERE name = '" + userInput + "'";
// userInput = "x'; DROP TABLE monitor; --"

// ✅ Parameterized. The driver sends SQL and values SEPARATELY.
PreparedStatement ps = conn.prepareStatement("SELECT * FROM monitor WHERE name = ?");
ps.setString(1, userInput);
```

**Why parameterization actually works:** the SQL text is parsed and planned *before* values are supplied, so a value can never be re-interpreted as syntax. It's not escaping — escaping can be got wrong. It's a structural separation.

Bonus: the database can cache the execution plan for a repeated `PreparedStatement`, so parameterized queries are usually *faster* too.

**In Spring you get this for free** — `JdbcClient` `?`/named parameters and JPA/JPQL parameters are all prepared. **The danger is string concatenation into a `@Query(nativeQuery = true)` or a dynamically built `String`.** That's the one place to be vigilant, and Lesson 22 covers it again.

---

## 3. Connection pooling — the operationally critical part

### Why pool at all

Opening a database connection is expensive: a TCP handshake, TLS negotiation, authentication, and server-side session/process setup. **5–50ms**, and on Postgres it forks a process. Doing that per query would dominate your latency.

A **connection pool** keeps N connections open and lends them out:

```
        ┌─────────────────────── HikariPool ───────────────────────┐
        │  [conn1: IDLE]  [conn2: IN USE]  [conn3: IDLE] ...       │
        └──────────────────────────────────────────────────────────┘
                    ▲                              │
   getConnection()  │                              ▼  close() → RETURNED, not closed
              ┌─────────────┐
              │ your thread │
              └─────────────┘
```

**Critically: `connection.close()` on a pooled connection does not close it — it returns it to the pool.** The pool hands you a proxy (Lesson 09's mechanism, again) that intercepts `close()`. If you *didn't* call close, that connection is gone forever from the pool's perspective. That's a connection leak.

### HikariCP

Spring Boot's default since 2.0. It's small, fast, and mostly correct by default — its author's philosophy is that a pool should have few knobs because most tuning is people papering over bugs.

> **Fun fact:** "Hikari" (光) is Japanese for "light" — the name is about both speed and small size. Its performance comes from unglamorous engineering: a custom lock-free `ConcurrentBag`, avoiding `ArrayList` bounds checks, and bytecode-level micro-optimization of the proxy layer. It benchmarks several times faster than the pools it replaced (C3P0, DBCP, Tomcat JDBC), which is why the ecosystem standardized on it.

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/vigil
    username: vigil
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 10           # ★ the one that matters most
      minimum-idle: 10                # = max for steady-state servers (avoid churn)
      connection-timeout: 3000        # ms to WAIT FOR A POOL SLOT before failing
      idle-timeout: 600000            # 10m — only relevant if minIdle < maxPoolSize
      max-lifetime: 1800000           # 30m — ★ MUST be less than the DB/proxy idle killer
      keepalive-time: 120000          # 2m — probe idle connections
      leak-detection-threshold: 20000 # ★ log a stack trace if held > 20s
      pool-name: VigilHikariPool
      auto-commit: false              # Spring manages transactions
```

### Pool sizing — the counterintuitive part

**Everyone's instinct is wrong here, which makes it a great interview question.**

The naive assumption: "200 Tomcat threads, so I need 200 connections." **No.** A larger pool is usually *slower*.

**HikariCP's formula** (from their well-known wiki page, originally derived from PostgreSQL research):

```
connections = ((core_count * 2) + effective_spindle_count)
```

For a modern 4-core server with SSD storage: **~8–10 connections.** For most services, **10 is right and 20 is generous.**

**Why more is worse:**

- A database executes queries with a limited number of CPUs and disks. 200 concurrent queries on 4 cores means each is time-sliced — the *same total work*, but every query's latency multiplied, plus context-switch overhead.
- Postgres allocates memory (`work_mem`) **per connection per sort/hash operation**. 200 connections × 4MB × multiple operations can OOM the database.
- Postgres uses a **process per connection**. Hundreds of processes is real overhead.
- More concurrency means more lock contention and more deadlock opportunities.

**The mental model to keep:** a queue in front of a fast database beats a mob inside a slow one. Queueing at the pool is *good* — it's admission control, and it keeps the database in its efficient operating range.

**The multi-instance trap that catches people:**

```
10 pods × maximum-pool-size: 20 = 200 connections
Postgres default max_connections = 100
→ FATAL: sorry, too many clients already
```

**Your pool size is per-instance; the database limit is global.** Autoscaling makes this worse — scaling out to handle load can take the database down. This is a genuine production incident pattern and a strong thing to raise unprompted. Mitigations: size pools with the replica count in mind, raise `max_connections` (with the memory cost), or put **PgBouncer** in front (transaction-mode pooling multiplexes many client connections onto few server ones — the standard answer at scale).

### `max-lifetime` — the setting that prevents mystery errors

```
max-lifetime (30m)  MUST be shorter than:
  · the database's idle connection timeout
  · any load balancer / NAT / proxy idle timeout (AWS NLB defaults to 350s!)
  · Postgres idle_in_transaction_session_timeout
```

If the network silently drops an idle connection and the pool doesn't know, you get **intermittent "connection reset by peer" or "closed connection"** on the first query after a quiet period. Classic symptom: errors only after periods of low traffic, or only on the first request each morning. **`max-lifetime` below every intermediary's timeout makes the pool retire connections before anything else kills them.** This one setting has resolved a lot of baffling incidents.

### Leak detection — turn it on

```yaml
leak-detection-threshold: 20000
```

If a connection is held longer than 20s without being returned, Hikari logs a **stack trace of where it was acquired**. That's the fastest possible route from "the app hangs after an hour" to the exact offending line. Small cost, enormous diagnostic value — enable it in non-production always, and consider it in production too.

### Diagnosing pool exhaustion

The error:
```
java.sql.SQLTransientConnectionException: VigilHikariPool - Connection is not available,
request timed out after 3000ms (total=10, active=10, idle=0, waiting=8)
```

**Read that carefully — it tells you everything:** all 10 connections active, 8 threads waiting. Now the question is *why* connections are held so long. The usual causes, in order of frequency:

1. **A slow query** holding a connection for seconds. Fix the query or add an index.
2. **A transaction spanning an HTTP call.** `@Transactional` around a `RestClient` call to Slack means the connection is pinned for the entire network round trip. **This is the classic mistake** (Lesson 23).
3. **`open-in-view=true`** keeping the connection open through view rendering and JSON serialization (Lesson 20).
4. **A genuine leak** — manual `getConnection()` without `close()`.
5. **Pool too small for legitimate throughput** — the *least* common cause, but the first thing people change.

**The instinct to build:** exhaustion is nearly always a *duration* problem, not a *size* problem. Increasing the pool masks it briefly and moves the bottleneck to the database. **Look at how long connections are held before you look at how many there are.**

Monitor it (Lesson 35):
```
hikaricp_connections_active
hikaricp_connections_pending      ← ★ alert if > 0 for any sustained period
hikaricp_connections_usage_seconds
```

---

## 4. `JdbcTemplate` and `JdbcClient`

Spring's JDBC abstraction removes the boilerplate while keeping you in control of the SQL.

### `JdbcClient` (Spring 6.1+) — the modern API, use this

```java
@Repository
@RequiredArgsConstructor
class CheckResultRepository {

    private final JdbcClient jdbc;

    // Query a list
    List<CheckResultRow> findRecent(UUID monitorId, int limit) {
        return jdbc.sql("""
                SELECT id, monitor_id, status, response_time_ms, checked_at
                FROM check_result
                WHERE monitor_id = :monitorId
                ORDER BY checked_at DESC
                LIMIT :limit
                """)
                .param("monitorId", monitorId)
                .param("limit", limit)
                .query(CheckResultRow.class)          // maps to a record by column name
                .list();
    }

    // Single optional result
    Optional<UptimeSummary> uptimeFor(UUID monitorId, Instant since) {
        return jdbc.sql("""
                SELECT count(*) AS total,
                       count(*) FILTER (WHERE status = 'UP') AS up,
                       percentile_cont(0.95) WITHIN GROUP (ORDER BY response_time_ms) AS p95
                FROM check_result
                WHERE monitor_id = :monitorId AND checked_at >= :since
                """)
                .param("monitorId", monitorId)
                .param("since", since)
                .query(UptimeSummary.class)
                .optional();
    }

    // Scalar
    long countFailing(UUID orgId) {
        return jdbc.sql("SELECT count(*) FROM monitor WHERE org_id = :orgId AND status = 'DOWN'")
                .param("orgId", orgId)
                .query(Long.class)
                .single();
    }

    // Update
    int markStale(Duration olderThan) {
        return jdbc.sql("UPDATE monitor SET status = 'UNKNOWN' WHERE last_checked_at < :cutoff")
                .param("cutoff", Instant.now().minus(olderThan))
                .update();
    }
}
```

**`JdbcClient` is a genuinely nice API**: fluent, named parameters, automatic record mapping, and `.list()`/`.single()`/`.optional()`/`.stream()` terminal operations. It wraps `JdbcTemplate`, so it's not a new engine — just a much better front door.

### `JdbcTemplate` — what you'll see in existing code

```java
List<Monitor> monitors = jdbcTemplate.query(
        "SELECT id, name, url FROM monitor WHERE org_id = ?",
        (rs, rowNum) -> new Monitor(rs.getObject("id", UUID.class),
                                    rs.getString("name"), rs.getString("url")),
        orgId);

Long count = jdbcTemplate.queryForObject(
        "SELECT count(*) FROM monitor WHERE org_id = ?", Long.class, orgId);

jdbcTemplate.update("UPDATE monitor SET status = ? WHERE id = ?", status.name(), id);
```

`NamedParameterJdbcTemplate` is the named-parameter variant. **`queryForObject` throws `EmptyResultDataAccessException` when there's no row** — a classic gotcha; prefer `JdbcClient`'s `.optional()`.

### Batch operations — the performance win

Inserting 10,000 check results one at a time is 10,000 network round trips.

```java
jdbcTemplate.batchUpdate(
        "INSERT INTO check_result (id, monitor_id, status, response_time_ms, checked_at) " +
        "VALUES (?, ?, ?, ?, ?)",
        results, 500,                                     // batch size
        (ps, r) -> {
            ps.setObject(1, r.id());
            ps.setObject(2, r.monitorId());
            ps.setString(3, r.status().name());
            ps.setInt(4, r.responseTimeMs());
            ps.setObject(5, r.checkedAt());
        });
```

**Postgres-specific note:** add `reWriteBatchedInserts=true` to your JDBC URL and the driver rewrites batched single-row inserts into multi-row `VALUES (...),(...),(...)` statements. **This can be a 2–3× improvement on bulk inserts** and it's a single URL parameter. Very few people know it, and Vigil's check-result writer needs it.

```
jdbc:postgresql://localhost:5432/vigil?reWriteBatchedInserts=true
```

---

## 5. When to use JDBC instead of JPA

**This is a senior judgment call, and having an opinion is worth points.**

### Use `JdbcClient` when:

| Case | Why |
|---|---|
| **Reporting / analytics queries** | Aggregations, window functions, CTEs, `percentile_cont` — JPQL can't express them, and you don't want entities |
| **Bulk inserts/updates** | JPA's per-entity overhead and persistence-context growth make it slow and memory-hungry |
| **Reads that don't need entities** | A dashboard row isn't a domain object; loading an entity to build a DTO is wasted work |
| **You need exact control of the SQL** | Query plans, index hints, database-specific features |
| **Very hot paths** | No persistence context, no dirty checking, no proxies — measurably less overhead |

### Use JPA when:

| Case | Why |
|---|---|
| **CRUD on a rich domain model** | Relationships, cascades, and dirty checking genuinely save code |
| **A unit of work touching several entities** | The persistence context tracks and flushes changes for you |
| **Optimistic locking** | `@Version` is nearly free |
| **You want DB portability** | Dialect abstraction |

**Vigil uses both deliberately, and this is a genuinely strong architecture story:**
- **JPA** for `Monitor`, `Incident`, `Organization`, `User` — real domain objects with relationships and lifecycles
- **`JdbcClient`** for `check_result` writes (batched, millions of rows, append-only) and all reporting/uptime queries (aggregations JPQL can't express)

**"I use JPA for the domain and JdbcClient for reporting and bulk writes"** is the answer that signals you've hit ORM limits and know where they are — rather than either worshipping or dismissing ORMs. Both extremes are junior positions.

---

## 6. Spring's exception translation

```java
try {
    jdbc.sql("INSERT INTO monitor (org_id, name) VALUES (:org, :name)")
        .param("org", orgId).param("name", name).update();
} catch (DuplicateKeyException e) {          // ← Spring's PORTABLE exception
    throw new MonitorNameTakenException(name);
}
```

Spring maps vendor error codes/SQLStates into a portable hierarchy of **unchecked** exceptions:

```
DataAccessException
├── DataIntegrityViolationException
│   └── DuplicateKeyException                 unique constraint
├── DataAccessResourceFailureException        can't reach the DB
├── QueryTimeoutException
├── CannotAcquireLockException                lock timeout / deadlock
├── OptimisticLockingFailureException         @Version mismatch
├── PessimisticLockingFailureException
├── EmptyResultDataAccessException            queryForObject found nothing
└── IncorrectResultSizeDataAccessException
```

**Why this matters:** without it, catching "duplicate key" means checking `SQLState 23505` on Postgres and error `1062` on MySQL — vendor-specific code in your business logic. With it, you catch `DuplicateKeyException` and your code is portable.

**`@Repository` enables translation** for classes that throw native exceptions (Lesson 06) — a `BeanPostProcessor` wraps the bean in a proxy that translates. `JdbcClient` and Spring Data repositories do it automatically. **This is the concrete answer to "what does `@Repository` actually do?"**

**One catch to be aware of:** `DataIntegrityViolationException` can mean unique violation, foreign key violation, or not-null violation. If you need to distinguish, inspect the cause or the constraint name. Better still, check preconditions explicitly *and* catch the exception — the check gives a good error message, the catch handles the race.

---

## 7. Build & break

**1. Get a real database running.** Postgres in Docker — you'll use this for the rest of the course:
```bash
docker run --name vigil-pg -e POSTGRES_PASSWORD=vigil -e POSTGRES_USER=vigil -e POSTGRES_DB=vigil -p 5432:5432 -d postgres:16
```
```xml
<dependency><groupId>org.postgresql</groupId><artifactId>postgresql</artifactId>
  <scope>runtime</scope></dependency>
<dependency><groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-jdbc</artifactId></dependency>
```
Note the driver's **`runtime` scope** — Lesson 01's point, now load-bearing.

**2. Create the schema by hand** (Flyway comes in Lesson 24):
```sql
CREATE TABLE monitor (
    id UUID PRIMARY KEY,
    org_id UUID NOT NULL,
    name TEXT NOT NULL,
    url TEXT NOT NULL,
    status TEXT NOT NULL,
    interval_seconds INT NOT NULL,
    last_checked_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT uq_monitor_org_name UNIQUE (org_id, name)
);

CREATE TABLE check_result (
    id UUID PRIMARY KEY,
    monitor_id UUID NOT NULL REFERENCES monitor(id) ON DELETE CASCADE,
    status TEXT NOT NULL,
    response_time_ms INT,
    status_code INT,
    error TEXT,
    checked_at TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_check_result_monitor_time ON check_result (monitor_id, checked_at DESC);
```
**That index is not decoration** — every Vigil query filters by monitor and orders by time. Lesson 24 explains why the column order matters.

**3. `JdbcClient` repository.** Implement `CheckResultRepository` from §3 with `findRecent`, `uptimeFor` (using `percentile_cont` for p95), and `countFailing`. **Note that the p95 query is impossible in JPQL** — that's the justification for using JDBC here, felt directly.

**4. ⭐ Cause and diagnose pool exhaustion.** This is the most valuable exercise in the lesson.
```yaml
spring.datasource.hikari.maximum-pool-size: 3
spring.datasource.hikari.connection-timeout: 2000
spring.datasource.hikari.leak-detection-threshold: 5000
```
```java
@GetMapping("/hold")
public String hold() throws Exception {
    try (Connection c = dataSource.getConnection()) {
        Thread.sleep(10_000);                     // simulate a slow query / external call
        return "held";
    }
}
```
Fire 6 concurrent requests. Requests 4–6 fail with `SQLTransientConnectionException` — **read the message: `total=3, active=3, idle=0, waiting=N`.** Then remove the `try-with-resources` (leak it deliberately) and watch Hikari print a **leak stack trace pointing at your line.** Then restore it.

**Now the real lesson:** raise the pool to 20 and note the requests succeed. Then reason: *what happens with 10 replicas?* 200 connections against a default `max_connections=100`. **Write down why raising the pool was the wrong fix.**

**5. Batch insert benchmark.** Insert 10,000 check results three ways and time each:
- one `update()` per row in a loop
- `batchUpdate` with batch size 500
- `batchUpdate` + `reWriteBatchedInserts=true` in the JDBC URL

Record the numbers. The gap between #1 and #3 is usually startling (often 10×+), and it's a great concrete data point to cite in an interview.

**6. Exception translation.** Insert a duplicate `(org_id, name)` and catch `DuplicateKeyException`. Then log `e.getCause()` and find the raw Postgres SQLState `23505`. **You've just seen the translation layer working** — and you now know exactly what `@Repository` buys you.

**7. SQL injection, safely.** In a throwaway method, build a query by concatenation and pass `x' OR '1'='1`. Watch it return all rows. Then parameterize it and watch the same input return nothing. **Do this once. It makes the abstract concrete.**

**8. Watch the pool metrics.** Add Actuator + Micrometer, hit `/actuator/metrics/hikaricp.connections.active`, and observe it move under load. This is the metric you'll alert on in production.

### Explain out loud (60s)

1. Why parameterized queries are structurally safe, not just escaped.
2. What `close()` does to a pooled connection, and what a leak costs you.
3. The pool-sizing formula, and three reasons a bigger pool is slower.
4. The multi-pod arithmetic, and what you'd check before scaling out.
5. Two kinds of work where you'd skip JPA and use `JdbcClient`.

---

## 8. Production rules

| Do / Never | Why |
|---|---|
| **Never build SQL by string concatenation** | Parameterization isn't escaping — the statement is parsed before values arrive, so a value structurally cannot become syntax |
| **Watch native `@Query` strings and dynamically built SQL especially** | Everything else in Spring is parameterized for you; those two are the only places injection can still get in |
| **Size the pool from cores, not from your thread count** | Roughly `cores × 2` — around ten for most services; 200 concurrent queries on four cores is the same work with every latency multiplied |
| **Set `minimum-idle` equal to `maximum-pool-size` on a steady-state server** | Otherwise the pool churns connections, paying the 5–50ms setup cost repeatedly for no benefit |
| **Multiply pool size by pod count and check it against `max_connections`** | Ten pods at twenty connections is 200 against a Postgres default of 100 — and you need headroom for migrations and an admin session |
| **Always set `connection-timeout`** | A starved pool should fail fast with a clear error instead of hanging request threads until everything times out at once |
| **Enable `leak-detection-threshold` outside production** | A connection that is never returned is gone from the pool permanently, and the leak is invisible until the pool is empty |
| **Let Spring own connection acquisition and release** | `close()` on a pooled connection returns it rather than closing it; hand-managed connections are where leaks come from |
| **Reach for `JdbcClient` for bulk, append-only and reporting work** | Entity semantics cost you nothing useful there, and skipping the ORM avoids loading a million rows to write or count them |
| **Catch `DuplicateKeyException`, never a vendor error code** | `@Repository`'s translation makes the same handling work on Postgres and Oracle; `23505` is a string that will outlive your memory of it |

---

## 9. Interview traps

**Q1. Why do we need connection pooling?**
Opening a connection costs 5–50ms (TCP + TLS + auth + server-side session, and a process fork on Postgres). Doing it per query would dominate latency. A pool keeps connections open and lends them out; `close()` on a pooled connection returns it rather than closing it.

**Q2. What pool size should you use, and why isn't bigger better?**
Roughly `(cores × 2) + spindles` — often 10 for a typical service. Bigger is usually slower: the database has finite CPU/disk, so more concurrent queries just time-slice the same work while adding context switching, per-connection memory (Postgres `work_mem`), process overhead, and lock contention. Queueing at the pool is healthy admission control.

**Q3. Ten replicas each with a 20-connection pool. What's the problem?**
200 connections against a database whose `max_connections` is often 100 → `too many clients already`, and autoscaling makes it worse. Pool size is per-instance; the limit is global. Fix: size with replica count in mind, raise `max_connections` (accepting memory cost), or front the DB with PgBouncer in transaction pooling mode.

**Q4. You see `Connection is not available, request timed out`. How do you diagnose it?**
Read the counts in the message (`total/active/idle/waiting`), then investigate **duration**, not size: slow queries, `@Transactional` methods spanning HTTP calls, `open-in-view=true`, or a genuine leak (enable `leak-detection-threshold` to get an acquisition stack trace). Raising the pool masks it and pushes the bottleneck into the database.

**Q5. What is `max-lifetime` for?**
To retire connections before something else silently kills them — the database's idle timeout, or a load balancer/NAT idle timeout (AWS NLB defaults to 350s). Without it you get intermittent "connection reset" errors, typically after quiet periods. Set it below every intermediary's timeout.

**Q6. How does `PreparedStatement` prevent SQL injection?**
SQL text is parsed and planned before parameter values are sent, so a value can never be reinterpreted as syntax. It's structural separation, not escaping. Bonus: the plan can be cached, so it's often faster.

**Q7. What does `@Repository` actually do?**
Beyond being a `@Component`, it enables **exception translation**: a `BeanPostProcessor` proxies the bean so vendor-specific `SQLException`s/Hibernate exceptions become Spring's portable unchecked `DataAccessException` hierarchy — letting you catch `DuplicateKeyException` regardless of database vendor.

**Q8. When would you use `JdbcClient` instead of JPA?**
Reporting and analytics (window functions, CTEs, percentiles that JPQL can't express), bulk inserts/updates (JPA's per-entity overhead and growing persistence context), read-only projections where entities are wasted work, and hot paths needing exact SQL control. JPA for CRUD on a rich domain model with relationships and optimistic locking.

**Q9. How do you make bulk inserts fast in Spring?**
`batchUpdate` with a batch size (e.g. 500) instead of per-row updates, plus `reWriteBatchedInserts=true` on the Postgres JDBC URL so the driver rewrites them into multi-row `VALUES` statements — often a 2–3× further gain. For JPA: `hibernate.jdbc.batch_size`, ordered inserts, and clearing the persistence context periodically (Lesson 24).

**Q10. Why is HikariCP the default?**
Measurably faster than C3P0/DBCP/Tomcat JDBC through lock-free data structures and a heavily optimized proxy layer, with a deliberately small configuration surface — its design position is that most pool tuning is compensating for application bugs.

---

## 10. Retention check

1. What does `connection.close()` do on a pooled connection, and what mechanism makes that possible?
2. Why is a 200-connection pool usually slower than a 10-connection pool?
3. Give the four most common causes of pool exhaustion, ordered by likelihood.
4. Why must `max-lifetime` be shorter than your load balancer's idle timeout?
5. Name two queries you'd write with `JdbcClient` rather than JPA, and why.
6. What single JDBC URL parameter meaningfully speeds up Postgres batch inserts?

---

**Next:** [Lesson 20 — JPA & Hibernate fundamentals](20-jpa-hibernate-fundamentals.md) — the persistence context, which is the concept most self-described "JPA developers" cannot explain. This is a big one.

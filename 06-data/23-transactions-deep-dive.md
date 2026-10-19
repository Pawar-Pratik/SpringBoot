# Lesson 23 — Transactions deep dive

> **Why this lesson exists:** this is the highest-value interview topic in the entire course. `@Transactional` is one annotation with roughly eight ways to be subtly wrong, and every one of them is a favourite question. Propagation, isolation, rollback rules, self-invocation, transaction scope, locking. Master this lesson and you will out-answer nearly every candidate you're compared against — and you'll write systems that don't corrupt data.

**Time:** ~95 minutes · **Prereq:** Lessons 09, 20

---

## 1. The idea in one sentence

> **`@Transactional` answers three independent questions — where the transaction starts (propagation), what undoes it (rollback rules), and who else may touch the rows meanwhile (isolation and locking) — and every classic bug here is you assuming one answer while the proxy applied another.**

Hold the three apart and the hard cases become ordinary. "Marked rollback-only" is propagation. A committed `IOException` is rollback rules. A lost update is locking. Nothing about them is mysterious once you know which question you got wrong.

---

## 2. ACID, and what your database actually gives you

| Property | Meaning | Who provides it |
|---|---|---|
| **Atomicity** | All operations commit or none do | Database (via undo logs / WAL) |
| **Consistency** | Constraints hold before and after | Database (constraints) + your code |
| **Isolation** | Concurrent transactions don't corrupt each other | Database (locks / MVCC) — **and it's configurable** |
| **Durability** | Committed data survives a crash | Database (WAL fsync) |

**Isolation is the one you have to think about**, because it's a dial with a performance cost, and the default is not the strongest setting.

### The four anomalies isolation prevents

**1. Dirty read** — reading uncommitted data from another transaction.
```
T1: UPDATE monitor SET status='DOWN'      (not committed)
T2: SELECT status → 'DOWN'                ← reads a value that may never exist
T1: ROLLBACK                              ← T2 acted on fiction
```

**2. Non-repeatable read** — reading the same row twice within one transaction and getting different values.
```
T1: SELECT status FROM monitor WHERE id=1 → 'UP'
T2: UPDATE monitor SET status='DOWN' WHERE id=1; COMMIT
T1: SELECT status FROM monitor WHERE id=1 → 'DOWN'    ← changed mid-transaction
```

**3. Phantom read** — the same *query* returns different **rows**.
```
T1: SELECT count(*) FROM monitor WHERE org_id=1 → 5
T2: INSERT INTO monitor (org_id) VALUES (1); COMMIT
T1: SELECT count(*) FROM monitor WHERE org_id=1 → 6   ← a new row appeared
```

**4. Lost update** — two transactions read, both modify, the second overwrites the first.
```
T1: read balance=100 ─┐
T2: read balance=100 ─┤
T1: write 100-10=90  ─┤
T2: write 100-20=80  ─┘  ← T1's deduction vanished
```
**Lost update is the one that hurts in real applications** and it's *not* prevented by the default isolation level. It's what `@Version` (optimistic locking) exists for. §6.

### Isolation levels

| Level | Dirty read | Non-repeatable | Phantom | Cost |
|---|---|---|---|---|
| `READ_UNCOMMITTED` | ✅ possible | ✅ | ✅ | lowest |
| **`READ_COMMITTED`** | ❌ prevented | ✅ possible | ✅ possible | **Postgres/Oracle/SQL Server default** |
| `REPEATABLE_READ` | ❌ | ❌ | ✅ possible* | **MySQL/InnoDB default** |
| `SERIALIZABLE` | ❌ | ❌ | ❌ | highest — real throughput cost |

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
```

**Facts worth knowing precisely:**
- **Postgres has no `READ_UNCOMMITTED`** — asking for it silently gives you `READ_COMMITTED`. Postgres's MVCC design simply never exposes uncommitted rows.
- **Postgres's `REPEATABLE_READ` also prevents phantom reads** (it's snapshot isolation), which is stronger than the SQL standard requires. So the standard's table understates what Postgres gives you.
- **MySQL's default is `REPEATABLE_READ`, Postgres's is `READ_COMMITTED`.** Code that works on one can misbehave on the other. Genuinely good trivia.
- **`SERIALIZABLE` in Postgres doesn't block — it aborts.** You get `could not serialize access due to concurrent update`, and **your application must retry.** People forget the retry and ship a system that randomly fails under load.

**Practical guidance:** leave the default. Use `@Version` (optimistic locking) for lost updates and `SELECT FOR UPDATE` (pessimistic) for genuine contention. Raising the isolation level is rarely the right first move, and "I'd use optimistic locking rather than raising isolation" is the answer that shows judgment.

---

## 3. `@Transactional` — how it actually works

You know this from Lesson 09: **`@Transactional` is `@Around` advice on a proxy.**

```java
// What the proxy does, conceptually:
public void createMonitor(Request req) {
    TransactionStatus tx = txManager.getTransaction(definition);   // BEGIN (or join)
    try {
        target.createMonitor(req);                                 // your code
        txManager.commit(tx);                                      // COMMIT
    } catch (RuntimeException | Error e) {
        txManager.rollback(tx);                                    // ROLLBACK
        throw e;
    } catch (Exception checked) {
        txManager.commit(tx);                                      // ⚠️ COMMIT! see §5
        throw checked;
    }
}
```

The transaction is bound to the **thread** via `TransactionSynchronizationManager` (a `ThreadLocal`) — which is why transactions don't propagate across `@Async`, `@Scheduled`, or reactive thread hops.

**Where to put it:** the **service layer** (Lesson 13). Not the controller (the transaction would span serialization), not the repository (each method becomes its own transaction, so you can't compose two writes atomically).

**The class-level convention worth adopting:**
```java
@Service
@Transactional(readOnly = true)                  // ★ default for the whole class
public class MonitorService {

    public MonitorResponse get(UUID id) { ... }             // inherits readOnly ✅

    @Transactional                                          // override for writes
    public MonitorResponse create(CreateMonitorRequest req) { ... }
}
```
Reads get the read-only benefits (Lesson 20 §6) and accidental mutations can't be flushed. Writes opt in explicitly.

---

## 4. Propagation — the most-asked sub-topic

**Propagation answers: "a transaction already exists; what should I do?"**

```java
@Transactional(propagation = Propagation.REQUIRED)      // default
```

| Propagation | Existing transaction | No transaction |
|---|---|---|
| **`REQUIRED`** (default) | **Join it** | Create one |
| **`REQUIRES_NEW`** | **Suspend it, create a new independent one** | Create one |
| `SUPPORTS` | Join it | Run **without** a transaction |
| `NOT_SUPPORTED` | Suspend it, run without | Run without |
| `MANDATORY` | Join it | **Throw** `IllegalTransactionStateException` |
| `NEVER` | **Throw** | Run without |
| `NESTED` | Create a **savepoint** | Create one |

### `REQUIRED` — and the consequence people miss

```java
@Service
public class OrderService {
    @Transactional                                    // REQUIRED
    public void placeOrder(Order o) {
        orderRepo.save(o);
        inventoryService.reserve(o);                  // also @Transactional REQUIRED
        auditService.log("order placed");             // also @Transactional REQUIRED
    }
}
```

**All three run in ONE physical transaction.** The inner methods *join* rather than starting their own. So:
- One commit at the end of the outer method
- **If `auditService.log` throws, the order is rolled back too.** Audit logging shouldn't be able to fail your order — that's what `REQUIRES_NEW` is for.

**And the trap that surprises everyone — "marked rollback-only":**

```java
@Transactional
public void outer() {
    try {
        inner();                       // @Transactional REQUIRED — same physical transaction
    } catch (Exception e) {
        log.warn("inner failed, continuing");     // ← you "handled" it
    }
    repo.save(somethingElse);
}   // 💥 UnexpectedRollbackException: Transaction silently rolled back
    //    because it has been marked as rollback-only
```

**What happened:** when `inner()` threw, its proxy couldn't roll back (it doesn't own the transaction), so it **set the rollback-only flag** on the shared transaction. Your `catch` swallowed the exception, so `outer()` proceeded and tried to commit — and the transaction manager refused, throwing `UnexpectedRollbackException`.

**This is a top-tier interview question because the mental model that produces the bug is so reasonable.** Fixes:
- If the inner failure is genuinely tolerable, make it `REQUIRES_NEW` so its rollback is isolated
- Or don't catch it — let it propagate and roll back everything
- Or use `noRollbackFor` on the inner method if the exception isn't really a failure

### `REQUIRES_NEW` — independent commit

```java
@Service
public class AuditService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void log(String action) {
        auditRepo.save(new AuditEntry(action));
    }   // commits INDEPENDENTLY — survives the caller's rollback
}
```

**Use it for:** audit logs that must persist even if the business operation fails, per-item processing in a batch (one bad item shouldn't roll back 999 good ones), and anything that must commit regardless of the caller's outcome.

**The costs, which you must state:**

1. **It consumes a second connection from the pool.** The outer transaction's connection is suspended but still held. **Nested `REQUIRES_NEW` calls in a loop can exhaust the pool** — a real incident pattern (Lesson 19).
2. **It's not visible to the outer transaction** until it commits, and vice versa. A `REQUIRES_NEW` method can't see the outer transaction's uncommitted work — which can look like missing data.
3. **`REQUIRES_NEW` through self-invocation does nothing** (Lesson 09). It silently joins the existing transaction. Genuinely dangerous, because you *believe* you have isolation.

### `NESTED` — savepoints

Creates a savepoint rather than a new transaction: an inner failure rolls back to the savepoint while the outer transaction continues; but if the outer rolls back, the inner is lost too.

**Requires JDBC savepoint support** and works with `DataSourceTransactionManager`; **JPA/`JpaTransactionManager` does not support `NESTED`** and throws `NestedTransactionNotSupportedException`. Knowing that limitation is a good detail.

### `MANDATORY` — an underused safety net

```java
@Transactional(propagation = Propagation.MANDATORY)
public void appendCheckResult(CheckResult r) { ... }   // MUST be called inside a transaction
```

A method that only makes sense as part of a larger unit of work declares it. Called without a transaction, it fails immediately and loudly instead of silently auto-committing. **Cheap, self-documenting protection.**

---

## 5. Rollback rules — the classic gotcha

```java
@Transactional
public void createMonitor(Request req) throws IOException {
    repo.save(new Monitor(req));
    throw new IOException("checked exception");
}
// 💀 THE MONITOR IS COMMITTED. No rollback.
```

**Default rule: roll back on `RuntimeException` and `Error`. Commit on checked exceptions.**

**Why?** Rod Johnson's design rationale: a checked exception is part of a method's declared contract — an *anticipated* condition the caller is expected to handle — so it may well represent a normal outcome that should persist. An unchecked exception is an unexpected failure. It's defensible reasoning, and it's **completely counter-intuitive to almost everyone.**

```java
// Fixes:
@Transactional(rollbackFor = Exception.class)                 // roll back on everything
@Transactional(rollbackFor = {IOException.class, SQLException.class})
@Transactional(noRollbackFor = InsufficientStockException.class)  // commit despite this one
```

**The practical answer, and the one to give in interviews:** use unchecked exceptions for your domain errors (which is Spring's convention anyway, Lesson 02), and add `rollbackFor = Exception.class` when you must work with checked exceptions from a library. **Some teams put `@Transactional(rollbackFor = Exception.class)` everywhere as a policy** so nobody has to remember the rule — a defensible position worth mentioning.

**Programmatic rollback** without throwing:
```java
TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
```
Rarely the right design (an exception is clearer), but it exists.

---

## 6. Locking

### Optimistic locking — your default

```java
@Entity
public class Monitor {
    @Version private long version;
}
```

```sql
UPDATE monitor SET name = ?, version = 6 WHERE id = ? AND version = 5
```
Zero rows affected → someone else won → `OptimisticLockingFailureException`.

```java
// Handling it — a retry, or a 409 to the client
@Retryable(retryFor = OptimisticLockingFailureException.class,
           maxAttempts = 3, backoff = @Backoff(delay = 50, multiplier = 2))
@Transactional
public void updateMonitor(UUID id, UpdateRequest req) { ... }
```

**⚠️ The retry must be *outside* the transaction.** Retrying inside a rolled-back transaction retries in a doomed transaction. `@Retryable` on the `@Transactional` method works because the retry advice is ordered outside the transaction advice by default — but **verify it** (Lesson 10 §ordering). This ordering subtlety is a genuinely advanced point.

**Why optimistic is the default choice:** no locks are held, so no blocking, no deadlocks, no lock timeouts. The cost is only paid on actual conflict — and in most applications conflicts are rare. **For a user-facing edit form, optimistic locking plus a 409 response ("this was changed by someone else, please review") is both correct and better UX than silently overwriting.**

**Forcing a version bump** when you modify a child but want the parent's version to change (so concurrent edits to *any* part of the aggregate conflict):
```java
@Lock(LockModeType.OPTIMISTIC_FORCE_INCREMENT)
```

### Pessimistic locking — for real contention

```java
public interface MonitorRepository extends JpaRepository<Monitor, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)                    // SELECT ... FOR UPDATE
    @Query("SELECT m FROM Monitor m WHERE m.id = :id")
    Optional<Monitor> findByIdForUpdate(@Param("id") Long id);

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @QueryHints(@QueryHint(name = "jakarta.persistence.lock.timeout", value = "3000"))
    Optional<Monitor> findAndLock(Long id);
}
```

| Lock mode | SQL | Meaning |
|---|---|---|
| `PESSIMISTIC_READ` | `SELECT ... FOR SHARE` | Others may read, not write |
| `PESSIMISTIC_WRITE` | `SELECT ... FOR UPDATE` | Exclusive |
| `PESSIMISTIC_FORCE_INCREMENT` | `FOR UPDATE` + version bump | Exclusive and versioned |

**Use pessimistic when:** contention is high enough that optimistic retries would thrash, and the critical section is short. Classic cases: decrementing inventory, allocating a seat, claiming a job from a queue.

**Always set a lock timeout.** Without one, a blocked transaction waits indefinitely, holding its own connection — and you have a cascade. **`SELECT FOR UPDATE SKIP LOCKED` is the elegant pattern for job claiming:**

```sql
SELECT * FROM check_queue
WHERE status = 'PENDING'
ORDER BY scheduled_at
LIMIT 10
FOR UPDATE SKIP LOCKED          -- ← other workers skip these rows instead of waiting
```

**`SKIP LOCKED` turns a table into a concurrent work queue with no external broker.** Multiple Vigil check-workers can pull disjoint batches with zero coordination. It's a genuinely elegant technique, it's underused, and mentioning it in a design interview lands well.

### Deadlocks

```
T1: lock A, then wants B
T2: lock B, then wants A          → deadlock; the DB kills one victim
```
```
ERROR: deadlock detected
DETAIL: Process 123 waits for ShareLock on transaction 456...
```

**Prevention:** always acquire locks in a **consistent order** (e.g. sort IDs before locking a batch), keep transactions short, use lower isolation where sufficient, and be prepared to retry — deadlock victims are retryable by definition. That "sort your IDs" trick is concrete, practical, and a good thing to know.

---

## 7. The mistakes that cause real incidents

### ❌ #1 — A transaction spanning a network call

```java
@Transactional
public void createMonitor(Request req) {
    Monitor m = repo.save(new Monitor(req));
    slackClient.notify("Monitor created: " + m.getName());   // 💥 HTTP call INSIDE the transaction
    repo.save(auditEntry(m));
}
```

**Why this is the worst one:**
- The database connection is held for the entire HTTP round trip. Slack takes 2s → your connection is pinned for 2s → 10 concurrent requests exhaust a 10-connection pool → **total outage caused by Slack being slow** (Lesson 19).
- If Slack times out and you throw, the monitor is rolled back — but Slack may have *already delivered* the message. Now your database and the outside world disagree.
- Distributed state and a database transaction cannot be made atomic. Trying is a category error.

**The fix — commit first, then publish:**
```java
@Transactional
public MonitorResponse createMonitor(Request req) {
    Monitor m = repo.save(new Monitor(req));
    events.publishEvent(new MonitorCreatedEvent(m.getId()));      // in-transaction, no I/O
    return mapper.toResponse(m);
}

@Component
class MonitorCreatedListener {
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)   // ★
    @Async
    void onCreated(MonitorCreatedEvent e) {
        slackClient.notify(...);        // runs AFTER commit, on another thread, no DB connection
    }
}
```

**`@TransactionalEventListener(AFTER_COMMIT)` is the single most useful annotation in this lesson.** It means "do this only if the transaction actually committed." No I/O inside transactions, and no notifications for work that rolled back. (Lesson 32 covers the outbox pattern for when you need at-least-once delivery guarantees.)

**The rule to internalize: nothing but database work belongs inside a transaction.** No HTTP, no message publishing, no file I/O, no `Thread.sleep`, no cache calls to a remote Redis.

### ❌ #2 — `@Transactional` on a private/final method, or self-invoked

Lesson 09. Silent no-op. **Diagnose with `TransactionSynchronizationManager.isActualTransactionActive()`.**

### ❌ #3 — A transaction wrapping a long read

```java
@Transactional
public void generateMonthlyReport() {
    List<CheckResult> all = repo.findAllByCheckedAtAfter(monthAgo);   // 5,000,000 rows
    // ... 60 seconds of processing, connection held the whole time
}
```
Fixes: stream with a cursor (`Stream<T>` + `@QueryHints(HINT_FETCH_SIZE)`), page through in separate transactions, or push the aggregation into SQL (Lesson 19).

### ❌ #4 — `readOnly = true` doesn't guarantee read-only

```java
@Transactional(readOnly = true)
public void oops(UUID id) {
    Monitor m = repo.findById(id).orElseThrow();
    m.setName("changed");
    repo.saveAndFlush(m);           // ⚠️ may still write, depending on the database
}
```
`readOnly` is primarily a **hint**: Hibernate skips dirty checking, and the JDBC connection is flagged read-only. Postgres will actually reject writes on a read-only transaction; some databases won't. **Treat it as an optimization and a safety net, not a hard guarantee.** Precise, and the kind of nuance that distinguishes a real answer.

### ❌ #5 — Assuming a transaction crosses threads

```java
@Transactional
public void parent() {
    childService.doAsync();       // @Async → different thread → NO TRANSACTION
}
```
Transactions are thread-bound (`ThreadLocal`). `@Async`, `@Scheduled`, executors, and parallel streams all start with no transaction. The async method needs its own `@Transactional` — and it will be a *separate* transaction, so it can't see the parent's uncommitted work. (Lesson 32.)

### ❌ #6 — Interleaving reads and writes across a long service method

Keep the transaction as small as the atomic unit requires. Load data, close the transaction, do the CPU work, open a transaction to write. A transaction is a lock on database resources — hold it for the minimum time.

---

## 8. Programmatic transactions

```java
@Service
@RequiredArgsConstructor
public class CheckBatchService {
    private final TransactionTemplate tx;                    // auto-configured bean

    public void processBatch(List<Monitor> monitors) {
        for (Monitor m : monitors) {
            try {
                tx.executeWithoutResult(status -> checkAndStore(m));   // one tx PER ITEM
            } catch (Exception e) {
                log.error("check failed for {}, continuing", m.getId(), e);
            }
        }
    }
}
```

**Why `TransactionTemplate` is genuinely better here:**
- The transaction boundary is **visible in the code** — it can't be silently defeated by self-invocation
- Per-item transactions mean one failure doesn't roll back the batch
- Fine-grained control: you can commit, mark rollback-only, or set a savepoint mid-loop

```java
var template = new TransactionTemplate(txManager);
template.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW);
template.setTimeout(5);
template.setIsolationLevel(TransactionDefinition.ISOLATION_READ_COMMITTED);
```

**Some very experienced teams prefer `TransactionTemplate` over `@Transactional` entirely**, precisely because proxy-based transactions have invisible failure modes. Saying *"I'd use `TransactionTemplate` where the boundary is subtle or per-item, because it can't be defeated by self-invocation and the scope is explicit"* is a strong, non-obvious answer.

---

## 9. Build & break

**Turn on the evidence first — you can't learn this without it:**
```properties
logging.level.org.springframework.transaction.interceptor=TRACE
logging.level.org.springframework.orm.jpa.JpaTransactionManager=DEBUG
logging.level.org.hibernate.SQL=DEBUG
```

**1. Prove propagation.** Two services, printing `TransactionSynchronizationManager.isActualTransactionActive()` and `getCurrentTransactionName()`. Test `outer()` calling `inner()` with `inner` as `REQUIRED`, then `REQUIRES_NEW`, then `MANDATORY`, then `NEVER`. **Read the TRACE log each time and note how many transactions begin.**

**2. ⭐ Reproduce `UnexpectedRollbackException`.** Outer `@Transactional` calling inner `@Transactional` (REQUIRED); inner throws; outer catches and continues. **Watch the commit fail with "Transaction silently rolled back because it has been marked as rollback-only."** Then fix it by making inner `REQUIRES_NEW`. **This experiment alone is worth the lesson.**

**3. Prove the checked-exception rule.**
```java
@Transactional
public void checkedNoRollback() throws Exception {
    repo.save(new Monitor("committed-anyway"));
    throw new Exception("boom");
}
@Transactional
public void uncheckedRollsBack() {
    repo.save(new Monitor("gone"));
    throw new RuntimeException("boom");
}
```
Query the table afterwards. **One row exists, one doesn't.** Then add `rollbackFor = Exception.class` and confirm the first also rolls back. **This is the single most surprising Spring behaviour; prove it to yourself.**

**4. Optimistic locking, end to end.** `@Version` on `Monitor`, then simulate two concurrent editors:
```java
Monitor a = repo.findById(id).orElseThrow();     // in transaction 1
Monitor b = repo.findById(id).orElseThrow();     // in transaction 2 (use two threads)
// both set the name, both save
```
Second commit throws `OptimisticLockingFailureException`. **Look at the SQL — see `WHERE id = ? AND version = ?`.** Then add `@Retryable` and confirm the retry succeeds. Then map it to **409 Conflict** in your `@ControllerAdvice`.

**5. Pessimistic locking and `SKIP LOCKED`.** Write a `claimBatch` query using `FOR UPDATE SKIP LOCKED`. Run two threads claiming simultaneously and confirm they get **disjoint** rows. Then remove `SKIP LOCKED` and watch the second thread **block**. **This is the exercise that makes you the person who knows how to build a job queue on Postgres.**

**6. Cause a deadlock deliberately.** Two transactions locking rows A then B, and B then A. Read the Postgres deadlock error. Then fix it by sorting IDs before locking. **You now understand deadlock prevention from experience.**

**7. ⭐ Fix the "transaction spanning a network call" antipattern.** Write the bad version with a `Thread.sleep(3000)` standing in for Slack, set `maximum-pool-size: 3`, and fire 5 concurrent requests. **Watch pool exhaustion.** Then refactor to `@TransactionalEventListener(AFTER_COMMIT)` + `@Async` and confirm the pool stays healthy and the notification still fires. **This is the most production-relevant exercise in Module 6.**

**8. Prove AFTER_COMMIT semantics.** Make the transaction roll back after publishing the event, and confirm the listener **does not fire**. Then use plain `@EventListener` and confirm it *does* — firing a notification for work that never persisted. That contrast is the whole point.

**9. `TransactionTemplate` per-item batch.** Process 10 monitors where #5 throws. Confirm 9 succeeded. Then do the same with a single `@Transactional` around the loop and confirm **all 10 rolled back.** Two different correct behaviours for two different requirements — know which you're choosing.

**10. Timeout.**
```java
@Transactional(timeout = 2)
public void slow() throws InterruptedException { Thread.sleep(5000); repo.save(...); }
```
Read the `TransactionTimedOutException`. Note that the timeout is checked **at the next database operation**, not by interrupting your thread — so a pure `Thread.sleep` isn't cut short. That detail matters and surprises people.

### Explain out loud (60s)

1. The three questions `@Transactional` answers, and one classic bug from each.
2. The "marked rollback-only" trace: who set the flag, and why your `catch` made it worse.
3. Why a checked exception commits by default, and the fix.
4. What `REQUIRES_NEW` costs, in connections and in visibility.
5. Why no network call belongs inside a transaction, and what to use instead.

---

## 10. Production rules

| Do / Never | Why |
|---|---|
| **Nothing but database work goes inside a transaction** | An HTTP call, a message publish or a sleep holds a pooled connection and a row lock for the duration of someone else's outage |
| **Use `@TransactionalEventListener(AFTER_COMMIT)` to trigger side effects** | It is the clean way to say "only if this actually committed" — no I/O inside the transaction, and no notifications for work that rolled back |
| **Put `@Transactional` on public service methods, and never self-invoke them** | It's proxy-based, so a `private`, `final` or `this.`-called method is silently un-advised — and a `REQUIRES_NEW` that silently joins is worse than none |
| **Add `rollbackFor = Exception.class` wherever checked exceptions can escape** | The default rolls back only on unchecked exceptions, so a checked `IOException` commits your half-finished work |
| **Never catch an exception from an inner `REQUIRED` method and carry on** | The inner proxy set rollback-only on the shared transaction, so your commit fails with `UnexpectedRollbackException` far from the cause |
| **Isolate a tolerable failure with `REQUIRES_NEW`, not with a `try/catch`** | That gives the inner work its own transaction to roll back, which is what you meant when you decided the failure was survivable |
| **Never call `REQUIRES_NEW` in a loop** | Each one suspends the caller's connection and takes a second from the pool, which is a documented way to exhaust it |
| **Put `readOnly = true` on read paths, knowing it isn't a guarantee** | It skips dirty-check snapshots and blocks accidental flushes, but it does not make the database refuse writes |
| **Use `@Version` plus a 409 as your default concurrency strategy** | No locks held means no blocking, no deadlocks and no lock timeouts; you only pay on an actual conflict, which is rare |
| **Put retry advice outside the transaction** | Retrying inside a transaction that is already doomed just repeats the failure; verify the advice ordering rather than assuming it |
| **Use `MANDATORY` on methods that only make sense inside a larger unit of work** | Called standalone they fail loudly instead of quietly auto-committing a fragment |
| **Don't reach for `NESTED` under JPA** | `JpaTransactionManager` doesn't support savepoint-based nesting and throws `NestedTransactionNotSupportedException` |
| **Never assume a transaction crosses a thread** | It is bound to the thread via `ThreadLocal`, so `@Async` work runs in no transaction at all unless it starts its own |

---

## 11. Interview traps

**Q1. How does `@Transactional` work?**
It's `@Around` advice on a Spring AOP proxy. The proxy asks the `PlatformTransactionManager` to begin (or join) a transaction bound to the current thread via a `ThreadLocal`, invokes the target, then commits — or rolls back on `RuntimeException`/`Error`. Because it's proxy-based, all Lesson 09 limitations apply: no effect on `private`/`final`/`static` methods or self-invoked calls.

**Q2. Where should `@Transactional` go and why not the other layers?**
The service layer. Not the controller (the transaction would span serialization and view rendering, holding a connection far too long). Not the repository (each call becomes its own transaction, so you can't make two writes atomic).

**Q3. Explain propagation, and give a real use for `REQUIRES_NEW`.**
`REQUIRED` (default) joins an existing transaction or creates one. `REQUIRES_NEW` suspends the current one and creates an independent transaction — used for audit logs that must persist even if the business operation rolls back, and for per-item processing so one bad item doesn't roll back the batch. Costs: a second pooled connection while the outer is suspended, no visibility of the outer's uncommitted work, and it's a silent no-op when self-invoked.

**Q4. What is `UnexpectedRollbackException` and how do you get it?** *(a favourite)*
An inner `REQUIRED` method throws; it can't roll back (it doesn't own the transaction) so it marks the shared transaction **rollback-only**. The caller catches the exception and continues, then commit is refused. Fix: make the inner method `REQUIRES_NEW` if its failure is tolerable, or let the exception propagate, or use `noRollbackFor`.

**Q5. Does `@Transactional` roll back on a checked exception?**
**No.** Default is rollback on `RuntimeException` and `Error` only; checked exceptions **commit**. The rationale is that a checked exception is a declared, anticipated outcome. Fix with `rollbackFor = Exception.class`, or use unchecked domain exceptions (Spring's own convention).

**Q6. Optimistic vs pessimistic locking?**
Optimistic (`@Version`): no locks; the UPDATE includes the version, and zero affected rows means a conflict → `OptimisticLockingFailureException`, which you retry or surface as 409. Best when conflicts are rare — no blocking, no deadlocks. Pessimistic (`SELECT FOR UPDATE`): acquires a real lock, blocking others. Best for genuine high contention with short critical sections, and it **must** have a lock timeout.

**Q7. What's `SELECT FOR UPDATE SKIP LOCKED` for?**
Concurrent work queues. Each worker claims a batch of rows and locks them; `SKIP LOCKED` makes other workers pass over locked rows instead of blocking, so N workers pull disjoint batches with no coordination — a job queue on plain Postgres, no broker required.

**Q8. Why is calling an HTTP API inside a `@Transactional` method dangerous?**
It holds a pooled database connection for the whole network round trip, so a slow third party causes pool exhaustion and a full outage. And it's not atomic: the call may succeed while the transaction rolls back, leaving your database and the outside world disagreeing. Fix: publish a domain event and act on it in `@TransactionalEventListener(AFTER_COMMIT)`, or use a transactional outbox for delivery guarantees.

**Q9. What does `@TransactionalEventListener(AFTER_COMMIT)` give you?**
The listener runs only if the transaction actually committed — so notifications, cache invalidations and message publishes never fire for rolled-back work, and no I/O happens inside the transaction.

**Q10. Do transactions propagate to `@Async` methods?**
No. Transactions are thread-bound via `ThreadLocal`, so an async method starts with none. It needs its own `@Transactional`, and that will be a separate transaction that cannot see the caller's uncommitted changes.

**Q11. What isolation level would you use, and what's the default?**
Leave the default: `READ_COMMITTED` on Postgres/Oracle/SQL Server, `REPEATABLE_READ` on MySQL. For lost updates use `@Version`, not a higher isolation level. Note Postgres has no `READ_UNCOMMITTED`, its `REPEATABLE_READ` also prevents phantoms, and its `SERIALIZABLE` **aborts** conflicting transactions rather than blocking — so the application must retry.

**Q12. Does `readOnly = true` prevent writes?**
It's mainly a hint: Hibernate skips dirty-check snapshots (so accidental mutations aren't flushed) and the JDBC connection is flagged read-only. Postgres will reject writes; not every database does. Treat it as an optimization plus a safety net, not a guarantee.

**Q13. How do you handle a transaction that must process 1,000 items where individual failures are acceptable?**
`TransactionTemplate` (or a `REQUIRES_NEW` method on a separate bean) per item, catching and logging per-item failures. A single transaction around the loop would roll back all 1,000 on one bad item, and would hold a connection and locks for the whole run.

**Q14. How do you prevent deadlocks?**
Acquire locks in a consistent order (e.g. sort IDs before locking a batch), keep transactions short, avoid unnecessary lock escalation, set lock timeouts, and retry — deadlock victims are retryable by definition.

---

## 12. Retention check

1. Explain `UnexpectedRollbackException` in 30 seconds, including why the mental model that causes it is reasonable.
2. Why doesn't a checked exception roll back, and what's the design argument for it?
3. Give three costs of `REQUIRES_NEW`.
4. Why is an HTTP call inside a transaction an outage risk *and* a correctness risk?
5. When do you choose pessimistic over optimistic locking, and what must you always add?
6. What does `SKIP LOCKED` enable, and why is that elegant?

---

**Next:** [Lesson 24 — Performance, N+1 & migrations](24-performance-n1-and-migrations.md) — finding and killing the N+1 problem, indexing, and running schema changes safely in production.

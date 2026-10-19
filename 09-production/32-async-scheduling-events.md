# Lesson 32 — Async, scheduling & events

> **Why this lesson exists:** Vigil's core is background work — thousands of scheduled checks, async notifications, incident escalation. Getting this right is what separates a demo from a system: `@Scheduled` on three instances runs everything three times, `@Async` with the default executor is a production hazard, and events published before a commit fire for work that never happened. Every one of these is a real incident pattern.

**Time:** ~80 minutes · **Prereq:** Lessons 08, 09, 23

---

## 1. The idea in one sentence

> **Everything in this lesson works perfectly on one instance and breaks on two: `@Scheduled` has no cluster awareness, `@Async` inherits no context, and an event published before commit announces work that may never have happened.**

So the question to ask of every piece of background work is the same one: what happens when there are three of these running, on different threads, and one of the transactions rolls back?

---

## 2. `@Async`

```java
@Configuration
@EnableAsync                             // ★ required, or @Async is silently ignored
public class AsyncConfig implements AsyncConfigurer { }
```

```java
@Service
public class NotificationService {

    @Async("notificationExecutor")                     // ★ ALWAYS name the executor
    public void sendAsync(Incident incident) { }        // fire and forget

    @Async("notificationExecutor")
    public CompletableFuture<DeliveryResult> sendTracked(Incident incident) {
        return CompletableFuture.completedFuture(deliver(incident));   // ★ note this
    }
}
```

**The rules, and each one has bitten people:**

1. **`@EnableAsync` is required.** Without it, `@Async` does nothing — no error, method runs synchronously.
2. **It's proxy-based** (Lesson 09). Self-invocation → runs synchronously, silently. `private`/`final` → silently ignored.
3. **Return `void` or `CompletableFuture<T>`.** Any other return type gets `null` immediately, because the proxy returns before the method finishes. Returning `String` from an `@Async` method is a bug that compiles.
4. **Exceptions in `void` async methods vanish** unless you install an `AsyncUncaughtExceptionHandler`. §2.2.
5. **`ThreadLocal` context doesn't propagate** — `SecurityContextHolder`, MDC, request scope, transactions. Fourth appearance of this lesson.
6. **Always name the executor.** §2.1.

### 2.1 The default executor is a production hazard

Without configuration, Boot's `@Async` uses a `SimpleAsyncTaskExecutor`, which **creates a new thread per invocation with no pooling and no bound.**

```
1,000 notifications → 1,000 platform threads → ~1GB of stack → OutOfMemoryError
```

> **Boot 3.2+ nuance:** with `spring.threads.virtual.enabled=true`, `SimpleAsyncTaskExecutor` uses **virtual** threads, so unbounded creation is far less catastrophic (Lesson 18). But "unbounded" is still wrong — you've simply moved the failure to whatever those threads contend for: connection pool, downstream rate limits, memory held by in-flight work. **Bound your concurrency deliberately.**

```java
@Configuration
@EnableAsync
@RequiredArgsConstructor
public class AsyncConfig implements AsyncConfigurer {

    @Bean("notificationExecutor")
    ThreadPoolTaskExecutor notificationExecutor() {
        var executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(4);
        executor.setMaxPoolSize(16);
        executor.setQueueCapacity(500);
        executor.setThreadNamePrefix("notify-");                  // ★ shows up in thread dumps
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.setWaitForTasksToCompleteOnShutdown(true);        // ★ graceful shutdown
        executor.setAwaitTerminationSeconds(30);
        executor.setTaskDecorator(new ContextPropagatingTaskDecorator());   // §2.3
        executor.initialize();
        return executor;
    }

    @Bean("checkExecutor")
    AsyncTaskExecutor checkExecutor() {
        // Vigil's HTTP checks are I/O-bound → virtual threads are ideal (Lesson 18)
        return new TaskExecutorAdapter(Executors.newVirtualThreadPerTaskExecutor());
    }

    @Override
    public Executor getAsyncExecutor() { return notificationExecutor(); }   // the default

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) ->
                log.error("Async method {} failed with params {}", method.getName(), params, ex);
    }
}
```

**Understand the `ThreadPoolTaskExecutor` growth semantics — this is a classic interview question, and the behaviour is counter-intuitive:**

```
1. Fewer than corePoolSize threads?     → create a new thread
2. Core threads busy?                    → QUEUE the task
3. Queue FULL?                           → create threads up to maxPoolSize
4. Queue full AND at maxPoolSize?        → RejectedExecutionHandler
```

**The trap: with a large queue, `maxPoolSize` is effectively never reached.** `core=4, max=100, queue=10000` behaves as a 4-thread pool until 10,000 tasks are backed up. People set `maxPoolSize=100` expecting 100 threads and get 4. **A large queue means high latency and invisible backpressure; a small queue means threads scale up but tasks get rejected sooner.** Choose deliberately.

**Rejection policies:**

| Policy | Behaviour | Use |
|---|---|---|
| `AbortPolicy` (default) | Throws `RejectedExecutionException` | When losing work must be loud |
| **`CallerRunsPolicy`** | The **submitting thread** runs the task | ✅ **Natural backpressure** — the producer slows down |
| `DiscardPolicy` | Silently drops | ❌ Almost never |
| `DiscardOldestPolicy` | Drops the oldest queued task | Rarely |

**`CallerRunsPolicy` is usually right** because it applies backpressure automatically: when the pool is saturated, the producer is forced to do the work itself and therefore stops producing. **But know the caveat:** if the caller is a Tomcat request thread, you've just made a request thread do background work — so it's excellent for internal producers and questionable at the HTTP boundary.

### 2.2 Exception handling

```java
@Async void fireAndForget() { throw new RuntimeException("gone forever"); }
```
Nobody is holding a `Future`, so nothing observes the exception. Without an `AsyncUncaughtExceptionHandler`, it's logged by the executor at best, and silently swallowed at worst. **Always configure the handler.**

```java
@Async CompletableFuture<Result> tracked() { ... }
tracked().exceptionally(ex -> { log.error("failed", ex); return Result.failed(); });
```
With a `CompletableFuture`, the exception is captured and surfaces on `get()`/`join()`/`exceptionally()`.

### 2.3 Context propagation — the ThreadLocal problem, solved

```java
public class ContextPropagatingTaskDecorator implements TaskDecorator {
    @Override
    public Runnable decorate(Runnable runnable) {
        // captured on the SUBMITTING thread
        Map<String, String> mdc = MDC.getCopyOfContextMap();
        SecurityContext security = SecurityContextHolder.getContext();
        UUID tenant = TenantContext.getOrNull();

        return () -> {
            try {
                if (mdc != null) MDC.setContextMap(mdc);
                SecurityContextHolder.setContext(security);
                if (tenant != null) TenantContext.set(tenant);
                runnable.run();
            } finally {
                MDC.clear();                       // ★ MANDATORY — pooled thread reuse
                SecurityContextHolder.clearContext();
                TenantContext.clear();
            }
        };
    }
}
```

**This one class solves the ThreadLocal problem for MDC (Lesson 14), security (Lesson 25), and tenancy (Lesson 28) simultaneously.** The `finally` block is not optional — a pooled thread retaining the previous task's tenant is a cross-tenant data leak.

> **Modern alternative:** Micrometer's **`ContextPropagation`** library (`ContextSnapshot`/`ContextRegistry`) standardizes this, and Boot auto-configures it for tracing. Worth naming: *"I'd use a `TaskDecorator`, or Micrometer's context-propagation library which generalizes it and integrates with tracing."*

---

## 3. `@Scheduled`

```java
@Configuration
@EnableScheduling                          // ★ required
public class SchedulingConfig { }
```

```java
@Component
@RequiredArgsConstructor
public class CheckScheduler {

    @Scheduled(fixedDelay = 10_000)                    // 10s AFTER the previous run FINISHES
    void dispatchDueChecks() { }

    @Scheduled(fixedRate = 10_000)                     // every 10s from the previous START
    void heartbeat() { }

    @Scheduled(cron = "0 0 2 * * *", zone = "UTC")     // 02:00 UTC daily
    void purgeOldCheckResults() { }

    @Scheduled(initialDelay = 30_000, fixedDelay = 60_000)   // wait 30s after startup
    void warmCaches() { }

    @Scheduled(fixedDelayString = "${vigil.checks.dispatch-interval:10s}")   // ★ configurable
    void configurableDispatch() { }
}
```

### `fixedDelay` vs `fixedRate` — a guaranteed interview question

```
fixedDelay = 10s:   [--run 15s--]..10s..[--run 15s--]..10s..
                    Gap is always 10s AFTER completion. Never overlaps. ✅ SAFE DEFAULT

fixedRate  = 10s:   [--run 15s--]
                              [--run 15s--]        ← the next run was due at t=10
                                        [--run..]  ← tasks pile up
                    If a run exceeds the period, executions QUEUE UP.
```

**Use `fixedDelay` unless you specifically need a fixed cadence.** With `fixedRate` and a task that occasionally runs long, invocations queue and — combined with the single-threaded default (§3.1) — the scheduler falls permanently behind. **This is a real production failure mode.**

**Cron format is Spring-specific: 6 fields, seconds first.**
```
second minute hour day-of-month month day-of-week
0      0      2    *            *     *              → 02:00:00 daily
0      */5    *    *            *     *              → every 5 minutes
0      0      9    *            *     MON-FRI        → 09:00 weekdays
```
**Standard Unix cron has 5 fields (no seconds), so copying a crontab line into `@Scheduled` shifts everything.** Also: **always set `zone`**, or you inherit the server's timezone and your "2am job" moves with DST, and differs between your laptop and production.

Spring also provides macros: `@Scheduled(cron = "@daily")`, `"@hourly"`, `"@midnight"`.

### 3.1 The single-thread default — a genuine trap

**By default, ALL `@Scheduled` methods in your application share ONE thread.**

```java
@Scheduled(fixedDelay = 1000) void quick() { }                  // needs to run every second
@Scheduled(fixedDelay = 1000) void slow()  { sleep(60_000); }   // takes a minute
// → quick() does not run for a minute. One slow task starves every schedule.
```

```java
@Configuration
@EnableScheduling
public class SchedulingConfig implements SchedulingConfigurer {

    @Override
    public void configureTasks(ScheduledTaskRegistrar registrar) {
        registrar.setTaskScheduler(taskScheduler());
    }

    @Bean
    ThreadPoolTaskScheduler taskScheduler() {
        var scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(5);                            // ★ not 1
        scheduler.setThreadNamePrefix("sched-");
        scheduler.setWaitForTasksToCompleteOnShutdown(true);
        scheduler.setAwaitTerminationSeconds(30);
        scheduler.setErrorHandler(t -> log.error("scheduled task failed", t));   // ★ see below
        return scheduler;
    }
}
```
Or simply:
```properties
spring.task.scheduling.pool.size=5
```

**And the other half of the trap: an uncaught exception in a `fixedDelay`/`fixedRate` task cancels that task permanently.** It won't run again until the application restarts, and there's no alert. **Always wrap the body in try/catch, or set an `ErrorHandler` on the scheduler.** This is one of the highest-value facts in the lesson — a silently dead scheduled job is exactly the kind of failure nobody notices for weeks.

```java
@Scheduled(fixedDelay = 10_000)
void dispatchDueChecks() {
    try {
        // ...
    } catch (Exception e) {
        log.error("dispatch failed, will retry next tick", e);   // ★ never let it escape
    }
}
```

### 3.2 THE MULTI-INSTANCE PROBLEM

**This is the most important thing in this lesson.**

```
3 replicas of Vigil, each running @Scheduled(cron = "0 0 2 * * *") sendDailyReport()

→ Every customer receives THREE identical daily reports.
→ The purge job runs three times concurrently → lock contention, possibly deadlock.
→ Every monitor is checked three times → 3× load on customers' servers,
  3× check_result rows, and uptime percentages computed from triplicated data.
```

**`@Scheduled` has no cluster awareness whatsoever.** Every instance runs every schedule. This works perfectly in development with one instance and breaks the moment you scale — which is exactly when nobody is looking for it.

### Solution 1 — ShedLock (the pragmatic standard)

```xml
<dependency><groupId>net.javacrumbs.shedlock</groupId>
  <artifactId>shedlock-spring</artifactId><version>5.16.0</version></dependency>
<dependency><groupId>net.javacrumbs.shedlock</groupId>
  <artifactId>shedlock-provider-jdbc-template</artifactId><version>5.16.0</version></dependency>
```

```sql
CREATE TABLE shedlock (
    name       VARCHAR(64)  NOT NULL PRIMARY KEY,
    lock_until TIMESTAMPTZ  NOT NULL,
    locked_at  TIMESTAMPTZ  NOT NULL,
    locked_by  VARCHAR(255) NOT NULL
);
```

```java
@Configuration
@EnableSchedulerLock(defaultLockAtMostFor = "10m")
public class ShedLockConfig {
    @Bean
    LockProvider lockProvider(DataSource dataSource) {
        return new JdbcTemplateLockProvider(JdbcTemplateLockProvider.Configuration.builder()
                .withJdbcTemplate(new JdbcTemplate(dataSource))
                .usingDbTime()                    // ★ use DB time, not server clocks
                .build());
    }
}
```

```java
@Scheduled(cron = "0 0 2 * * *", zone = "UTC")
@SchedulerLock(name = "dailyReport",
               lockAtMostFor = "30m",             // ★ safety net if the holder dies
               lockAtLeastFor = "1m")             // ★ prevents double-run on clock skew
void sendDailyReport() { }
```

**Exactly one instance acquires the lock and runs; the others skip immediately.**

**The two parameters that matter:**
- **`lockAtMostFor`** — if the holder crashes (SIGKILL, OOM) it never releases the lock, so this is the expiry that lets another instance take over. **Set it longer than the job's worst-case runtime, or a second instance will start while the first is still running.**
- **`lockAtLeastFor`** — holds the lock for a minimum even if the job finishes instantly, protecting against clock skew causing two instances to both run a fast job.

**`.usingDbTime()` is important:** it makes the database's clock authoritative rather than each application server's, eliminating clock-skew bugs entirely. Easy to miss, worth knowing.

**ShedLock is not a distributed scheduler** — it's a "don't run this twice" lock. It doesn't distribute work, retry failures, or give you a job history. That limitation is exactly the right thing to state when comparing it to Quartz.

### Solution 2 — Database work claiming (what Vigil actually needs)

For high-volume work, one instance doing everything is a bottleneck. **You want all instances working on *disjoint* subsets** — which is Lesson 23's `SKIP LOCKED`:

```java
@Repository
@RequiredArgsConstructor
public class CheckQueueRepository {

    private final JdbcClient jdbc;

    @Transactional
    public List<DueCheck> claimBatch(int limit, String workerId) {
        return jdbc.sql("""
                WITH claimed AS (
                    SELECT id FROM monitor
                    WHERE next_check_at <= now()
                      AND status <> 'PAUSED'
                    ORDER BY next_check_at
                    LIMIT :limit
                    FOR UPDATE SKIP LOCKED          -- ★ other workers skip these rows
                )
                UPDATE monitor m
                SET next_check_at = now() + (m.interval_seconds || ' seconds')::interval,
                    claimed_by = :workerId,
                    claimed_at = now()
                FROM claimed
                WHERE m.id = claimed.id
                RETURNING m.id, m.url, m.type, m.config
                """)
                .param("limit", limit).param("workerId", workerId)
                .query(DueCheck.class).list();
    }
}
```

```java
@Component
@RequiredArgsConstructor
public class CheckDispatcher {

    private final CheckQueueRepository queue;
    private final CheckExecutorService executors;

    @Scheduled(fixedDelay = 5_000)                 // ★ NO ShedLock — every instance participates
    void dispatch() {
        try {
            List<DueCheck> batch = queue.claimBatch(50, workerId);
            batch.forEach(executors::executeAsync);      // fan out to the virtual-thread executor
        } catch (Exception e) {
            log.error("dispatch failed, retrying next tick", e);
        }
    }
}
```

**This is genuinely elegant, and it's the design to be able to explain:**
- **All instances work in parallel** on disjoint batches — it scales horizontally by adding replicas
- **No broker, no coordinator, no leader election** — Postgres provides the mutual exclusion
- **Crash-safe:** `next_check_at` is advanced inside the claiming transaction, and the `claimed_at` timestamp lets a reaper reclaim work from a dead worker
- **Naturally rate-limited** by `LIMIT` and the dispatch interval

**Say this in an interview:** *"For deduplicating a singleton job I'd use ShedLock. For distributing high-volume work I'd use `SELECT ... FOR UPDATE SKIP LOCKED` so every instance claims a disjoint batch — it scales by adding replicas and needs no external coordinator."* That's a design answer, not a library answer.

### Solution 3 — Quartz, and when it's warranted

| | `@Scheduled` + ShedLock | Quartz |
|---|---|---|
| Setup | Trivial | Several tables, more config |
| Clustered | Via ShedLock | ✅ Built in |
| Dynamic jobs (runtime create/modify) | ❌ | ✅ |
| Misfire handling | ❌ | ✅ configurable policies |
| Persistent job state & history | ❌ | ✅ |
| Per-job retry | ❌ | ✅ |

**Use Quartz when jobs are data (user-defined schedules), when you need misfire policies, or when you need job history.** Otherwise `@Scheduled` + ShedLock is dramatically simpler. **Vigil's monitor intervals are user-defined, but they're modelled as `next_check_at` rows rather than Quartz triggers** — which is simpler and scales better. Being able to justify *not* using Quartz is as valuable as knowing it exists.

---

## 4. Application events

### Basics

```java
public record MonitorWentDownEvent(UUID monitorId, UUID orgId, String error, Instant at) {}
```
```java
@Service
@RequiredArgsConstructor
public class CheckResultProcessor {
    private final ApplicationEventPublisher events;

    @Transactional
    public void record(Monitor monitor, CheckOutcome outcome) {
        checkResultRepository.save(outcome.toEntity(monitor));
        if (outcome.isFailure() && monitor.getStatus() != DOWN) {
            monitor.setStatus(DOWN);
            events.publishEvent(new MonitorWentDownEvent(
                    monitor.getId(), monitor.getOrgId(), outcome.error(), outcome.at()));
        }
    }
}
```
```java
@Component
public class IncidentListener {
    @EventListener
    void onMonitorDown(MonitorWentDownEvent e) { incidentService.open(e); }

    @EventListener(condition = "#event.orgId != null")        // SpEL filtering
    void onlyForRealOrgs(MonitorWentDownEvent e) { }

    @EventListener
    @Order(1)                                                  // ordered listeners
    void first(MonitorWentDownEvent e) { }
}
```

**Events are SYNCHRONOUS by default** and run in the publisher's thread and transaction. That surprises people: `publishEvent` is a method call, not a queue.

### Why use events at all

**1. Breaking circular dependencies** (Lesson 07 §7). `MonitorService` no longer needs to know `IncidentService` exists.

**2. Open/closed extension.** Adding "also update the status page cache when a monitor goes down" means adding a listener — no existing class changes.

**But be honest about the cost:** events make control flow **invisible**. `Ctrl+click` on `publishEvent` doesn't show you the six listeners it triggers. **Overusing events produces a system nobody can trace.**

**The guideline:** use an event when the publisher genuinely shouldn't care who reacts (notifications, cache invalidation, audit, analytics). Use a direct method call when the reaction is *part of the operation* and its failure should fail the operation. **"Would this operation be wrong if no one handled it?" — if yes, call the method directly.** That test is a good thing to articulate.

### `@TransactionalEventListener` — the most useful annotation here

From Lesson 23 §7, and it deserves restating because it solves a real correctness problem:

```java
@Component
public class NotificationListener {

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)   // ★ the default
    @Async("notificationExecutor")
    void onMonitorDown(MonitorWentDownEvent e) {
        slackClient.notify(e);          // ✅ runs only if the transaction COMMITTED,
    }                                    //    on another thread, holding no DB connection
}
```

| Phase | Fires |
|---|---|
| `BEFORE_COMMIT` | Before commit — still in the transaction, can still fail it |
| **`AFTER_COMMIT`** (default) | **Only on successful commit** ✅ |
| `AFTER_ROLLBACK` | Only on rollback |
| `AFTER_COMPLETION` | Either way |

**What this prevents:** a plain `@EventListener` fires inside the transaction, so if the transaction later rolls back, **you've already sent a Slack message about an incident that doesn't exist.** With `AFTER_COMMIT`, the notification is impossible unless the data is durable.

**Two gotchas:**
1. **Writing to the database in an `AFTER_COMMIT` listener needs `REQUIRES_NEW`** — the original transaction is finished, so there's nothing to join. Without it, the write silently doesn't happen (or throws, depending on config).
2. **`@TransactionalEventListener` outside a transaction does nothing by default.** Set `fallbackExecution = true` if the listener should also run when there's no transaction.

### The transactional outbox pattern — for guaranteed delivery

`AFTER_COMMIT` still has a gap: the transaction commits, then the JVM is killed before the notification is sent. **The notification is lost, permanently, with no record that it should have happened.**

For anything that *must* be delivered (billing events, alerts, cross-service messages):

```java
// 1. Write the message to an outbox table IN THE SAME TRANSACTION as the business change.
//    Atomic: either both happen or neither does.
@Transactional
public void recordFailure(Monitor monitor, CheckOutcome outcome) {
    monitor.setStatus(DOWN);
    outboxRepository.save(new OutboxMessage(
            "MonitorWentDown", monitor.getId(), toJson(event), Instant.now()));
}
```
```java
// 2. A separate poller publishes and marks them sent. Crash-safe: unsent rows are retried.
@Component
@RequiredArgsConstructor
public class OutboxPublisher {

    @Scheduled(fixedDelay = 1000)
    void publish() {
        List<OutboxMessage> batch = outboxRepository.claimUnpublished(100);   // SKIP LOCKED
        for (OutboxMessage m : batch) {
            try {
                broker.send(m.topic(), m.payload());       // or an HTTP call
                outboxRepository.markPublished(m.id());
            } catch (Exception e) {
                outboxRepository.recordFailure(m.id(), e.getMessage());  // retry next tick
            }
        }
    }
}
```

**Why this is the correct pattern:** it makes "the business change happened" and "the message will be sent" a single atomic fact, because they're the same database transaction. **You cannot achieve that with a message broker and a database without either an outbox or a distributed transaction** — and distributed transactions (XA) are slow, complex, and unavailable with most modern brokers.

**The trade-off to state:** delivery becomes **at-least-once**, not exactly-once. The poller can send and then crash before marking it published, so the message is re-sent. **Therefore every consumer must be idempotent** (Lessons 34, 37). Naming that consequence is what makes the answer complete.

**Being able to explain the outbox pattern is a strong senior/architect signal** — it shows you understand that "publish an event after committing" has a failure mode, and you know the standard remedy.

---

## 5. Build & break — Vigil's check engine

**This is the heart of the project.** Everything so far has been infrastructure; this is the feature.

**1. The scheduler configuration.** `ThreadPoolTaskScheduler` with pool size 5, named threads, graceful shutdown, and an `ErrorHandler`.

**2. Executors.** A bounded `notificationExecutor` (`ThreadPoolTaskExecutor`, `CallerRunsPolicy`, `TaskDecorator`) and a virtual-thread `checkExecutor` for I/O-bound HTTP checks.

**3. The dispatcher** with `claimBatch` using `FOR UPDATE SKIP LOCKED`, dispatching to the check executor. **No ShedLock** — every instance should participate.

**4. ⭐ Prove multi-instance safety.** Run Vigil twice on different ports against the same database:
```bash
java -jar vigil.jar --server.port=8080 --vigil.worker-id=w1
```
```bash
java -jar vigil.jar --server.port=8081 --vigil.worker-id=w2
```
Seed 100 due monitors. **Confirm each monitor is checked exactly once**, and that both workers did roughly half:
```sql
SELECT claimed_by, count(*) FROM monitor GROUP BY claimed_by;
```
**Then remove `SKIP LOCKED` and watch worker 2 block on worker 1's rows.** Then remove `FOR UPDATE` entirely and watch **both workers check the same monitors** — duplicated results, doubled load on the target, corrupted uptime. **This experiment is the single most valuable one in the lesson.**

**5. ShedLock for the singleton jobs.** Add the table (via Flyway) and `@SchedulerLock` to `purgeOldCheckResults` and `sendDailyReport`. **With two instances running, confirm the purge job logs from only one.** Then remove `@SchedulerLock` and confirm it logs from both.

**6. Events end to end.** `MonitorWentDownEvent` → `IncidentListener` (opens an incident, plain `@EventListener`, inside the transaction so it's atomic with the status change) → `NotificationListener` (`@TransactionalEventListener(AFTER_COMMIT)` + `@Async`, so Slack is notified only after commit and off the request thread).

**7. ⭐ Prove AFTER_COMMIT matters.** Make the transaction throw *after* publishing the event.
- With plain `@EventListener`: **the Slack notification fires for an incident that was rolled back.**
- With `@TransactionalEventListener(AFTER_COMMIT)`: nothing fires.

**Log both. That contrast is the whole argument.**

**8. The outbox.** `outbox_message` table, write inside the business transaction, an `OutboxPublisher` polling with `SKIP LOCKED`. Then **kill the app between commit and publish** (`kill -9` right after the transaction) and confirm the message is published on restart. **You've just demonstrated at-least-once delivery across a crash** — a genuinely impressive thing to have working.

**9. Break these on purpose:**
- Remove `@EnableAsync` → `@Async` runs synchronously, **no error**. Confirm by logging the thread name.
- Self-invoke an `@Async` method → synchronous. Log the thread name.
- Return `String` from an `@Async` method → caller gets `null` immediately.
- Throw from a `void @Async` method with no handler → the exception disappears.
- Throw from a `fixedDelay` scheduled task → **confirm it never runs again.** This one is important; sit with it.
- Use `fixedRate = 1000` with a 5-second body → watch executions pile up.
- Set `core=2, max=50, queue=10000` and submit 1,000 tasks → **only 2 threads ever run.** Then reduce the queue to 10 and watch it scale to 50.
- Remove `MDC.clear()` from the `TaskDecorator` → watch correlation IDs bleed between tasks.

**10. Verify graceful shutdown.** With `waitForTasksToCompleteOnShutdown(true)`, submit a long task and send SIGTERM. Confirm it finishes. Then set it `false` and confirm it's abandoned mid-flight. **That's the difference between a clean deploy and lost work** (Lesson 08 §3).

### Explain out loud (60s)

1. Why the default `@Async` executor is a hazard, and what virtual threads do and don't fix.
2. `fixedDelay` vs `fixedRate`, and the failure mode of getting it wrong.
3. Two ways a `@Scheduled` job dies silently.
4. The multi-instance problem, and the difference between ShedLock and `SKIP LOCKED`.
5. The outbox pattern, and the consequence you must accept with it.

---

## 6. Production rules

| Do / Never | Why |
|---|---|
| **Never use the default `@Async` executor** | `SimpleAsyncTaskExecutor` creates an unbounded thread per task; with virtual threads it merely moves the failure to the connection pool or downstream rate limit |
| **Choose queue size deliberately, knowing `maxPoolSize` is often unreachable** | With a large queue the pool stays at core size until the queue fills, so `max=100` can behave as 4 threads with invisible latency |
| **Prefer `CallerRunsPolicy` for internal producers** | It applies natural backpressure by making the producer do the work, so it stops producing — but it's wrong at the HTTP boundary, where the caller is a request thread |
| **Install a `TaskDecorator` to carry MDC, security and tenant context** | They are all `ThreadLocal`-backed, and a pooled thread retaining the previous task's tenant is a cross-tenant leak |
| **Clear the propagated context in a `finally`** | Half the decorator is the copy; the other half is making sure it doesn't outlive the task on a reused thread |
| **Use `fixedDelay` unless you genuinely need a fixed cadence** | With `fixedRate`, a task that occasionally runs long queues invocations and — on the single-threaded default — falls permanently behind |
| **Give `@Scheduled` its own pool and an `ErrorHandler`** | All scheduled methods share one thread by default, and an uncaught exception cancels that task permanently with no alert until restart |
| **Always set `zone` on a cron expression, and remember Spring's cron has six fields** | Otherwise your 2am job moves with the server's DST, and a copied Unix crontab line shifts every field |
| **Never deploy `@Scheduled` to multiple instances without ShedLock or work claiming** | Every instance runs every schedule, and it only breaks when you scale — exactly when nobody is looking for it |
| **Use `SELECT … FOR UPDATE SKIP LOCKED` to distribute work, ShedLock to deduplicate a singleton job** | They solve different problems: one hands each replica a disjoint batch, the other just stops a job running twice |
| **Use `.usingDbTime()` with ShedLock** | It makes the database clock authoritative, which removes clock skew between application servers as a failure mode |
| **Use `@TransactionalEventListener(AFTER_COMMIT)` for side effects** | A plain listener fires inside the transaction, so a later rollback leaves you having announced an incident that doesn't exist |
| **Reach for an outbox when a message must not be lost** | Only writing the message in the same transaction makes "it happened" and "it will be sent" one atomic fact — and accept that delivery becomes at-least-once |
| **Ask "would this operation be wrong if nobody handled it?" before publishing an event** | If yes, call the method directly; events make control flow invisible, and overusing them produces a system nobody can trace |

---

## 7. Interview traps

**Q1. What's wrong with the default `@Async` executor?**
It's a `SimpleAsyncTaskExecutor` — a new thread per call, unbounded and unpooled, so a burst of work exhausts memory. With virtual threads enabled it's less catastrophic but still unbounded, moving the failure to the connection pool or downstream limits. Always define a named, bounded `ThreadPoolTaskExecutor` (or a virtual-thread executor for I/O-bound work) and reference it by name.

**Q2. Explain `ThreadPoolTaskExecutor`'s growth behaviour.**
Create threads up to `corePoolSize` → then **queue** → when the queue is full, create threads up to `maxPoolSize` → then reject. **The trap: a large queue means `maxPoolSize` is effectively never reached** — `core=4, max=100, queue=10000` behaves as a 4-thread pool. A large queue buys latency and hides backpressure; a small one scales threads but rejects sooner.

**Q3. Which rejection policy would you use and why?**
`CallerRunsPolicy` usually: the submitting thread executes the task, which forces the producer to slow down — natural backpressure. Caveat: if the caller is a Tomcat request thread, you've made a request thread do background work, so it's best for internal producers rather than at the HTTP boundary.

**Q4. `fixedDelay` vs `fixedRate`?**
`fixedDelay` measures the gap **after completion**, so runs never overlap. `fixedRate` measures from the previous **start**, so if a run exceeds the period, executions queue up and — with the default single scheduler thread — the schedule falls permanently behind. Default to `fixedDelay`.

**Q5. What happens if a `@Scheduled` task throws?**
For `fixedDelay`/`fixedRate`, an uncaught exception **cancels the task permanently** — it never runs again until restart, with no alert. Always catch inside the task or configure an `ErrorHandler` on the `TaskScheduler`.

**Q6. How many threads run `@Scheduled` tasks by default?**
**One**, shared by all scheduled methods — so one slow task starves every other schedule. Configure `spring.task.scheduling.pool.size` or a `ThreadPoolTaskScheduler`.

**Q7. ⭐ You deploy three replicas. What happens to `@Scheduled`?**
Every instance runs every schedule — triplicated reports, concurrent purges, monitors checked three times. `@Scheduled` has no cluster awareness. Fixes: **ShedLock** to make a job run on exactly one instance, or **`SELECT ... FOR UPDATE SKIP LOCKED`** work claiming so every instance processes a disjoint batch (better for high volume, since it scales with replicas), or Quartz in clustered mode.

**Q8. How does ShedLock work, and what are `lockAtMostFor`/`lockAtLeastFor`?**
A shared lock row (JDBC/Redis/Mongo) that one instance acquires before running; others skip. `lockAtMostFor` is the expiry that releases the lock if the holder dies — it must exceed the job's worst-case runtime or a second instance will start while the first still runs. `lockAtLeastFor` holds the lock for a minimum to prevent a double-run caused by clock skew on fast jobs. Use `.usingDbTime()` so the database clock is authoritative.

**Q9. Are Spring application events synchronous?**
Yes by default — they run on the publisher's thread, inside its transaction. Add `@Async` for asynchrony (and then exceptions no longer propagate to the publisher).

**Q10. What does `@TransactionalEventListener(AFTER_COMMIT)` solve?**
It runs the listener only if the transaction committed, so you never notify about work that rolled back, and no I/O happens inside the transaction (which would hold a connection). Gotchas: writing to the DB there needs `REQUIRES_NEW`, and outside a transaction the listener doesn't run unless `fallbackExecution = true`.

**Q11. What's the transactional outbox pattern and why do you need it?**
Write the outgoing message to an outbox table **in the same transaction** as the business change, then have a separate poller publish it. It makes "the change happened" and "the message will be sent" one atomic fact — which `AFTER_COMMIT` alone can't guarantee, since the process can die between commit and send. The consequence is at-least-once delivery, so consumers must be idempotent.

**Q12. Why doesn't the security context / MDC / tenant reach an `@Async` method?**
They're `ThreadLocal`-based and the async method runs on a different thread. Fix with a `TaskDecorator` that captures the context on the submitting thread and restores it on the executing thread — **clearing it in a `finally`**, since pooled threads are reused and a retained tenant is a cross-tenant leak. Micrometer's context-propagation library generalizes this.

**Q13. When would you use Quartz over `@Scheduled` + ShedLock?**
When schedules are runtime data (user-defined jobs), when you need misfire policies for missed executions, or when you need persistent job history and per-job retry. Otherwise `@Scheduled` + ShedLock is far simpler — and modelling schedules as `next_check_at` rows is often simpler still and scales better.

---

## 8. Retention check

1. What exactly happens with `core=4, max=100, queue=10000` under a burst of 1,000 tasks?
2. Why does a thrown exception permanently kill a `fixedDelay` task, and what's the fix?
3. You scale to three replicas. Name two different solutions and when each applies.
4. Explain what `lockAtMostFor` protects against and what happens if it's too short.
5. Give the failure mode that `AFTER_COMMIT` fixes, and the one it doesn't.
6. Why must the `TaskDecorator` clear context in a `finally`?

---

**Next:** [Lesson 33 — Caching done right](33-caching.md) — Redis, invalidation, and the stampede problem.

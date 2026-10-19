# Lesson 36 — Performance & JVM tuning

> **Why this lesson exists:** "The app is slow" is the vaguest and most common production complaint. This lesson gives you a *method* — measure, localize, fix, verify — plus the specific JVM and Spring knowledge to act on what you find. It also covers container memory (where most JVM production incidents actually come from) and the AOT/native-image story, which is a current interview topic.

**Time:** ~75 minutes · **Prereq:** Module 9 so far

---

## 1. The idea in one sentence

> **High latency with low CPU means you are waiting, not computing — and since the two most common causes of slowness both look exactly like that, no JVM flag will help until you've measured where the waiting happens.**

That is the whole method in one line: measure, localize, fix, re-measure. The tuning knowledge in this lesson is only useful after localization, and most of it turns out to be about memory limits in containers rather than about the garbage collector.

---

## 2. The method — this matters more than any tuning flag

```
1. DEFINE      What is "slow"? Which endpoint, which percentile, what target?
                  "p99 of GET /monitors is 2.4s, we need <300ms"
2. MEASURE     Reproduce it and get numbers. Don't guess.
3. LOCALIZE    Which layer? DB / app CPU / network / GC / lock contention / thread starvation
4. FIX         One change at a time.
5. VERIFY      Measure again. Did it help? By how much?
6. GUARD       Add a test or an alert so it can't silently regress.
```

**The rule: never optimize without a measurement, and never accept a fix without a re-measurement.** Most "performance work" is people changing things they believe are slow. Donald Knuth's actual quote is worth having precisely: *"Premature optimization is the root of all evil"* — but the sentence before it says you should absolutely optimize the critical 3% once you've **identified** it. **The sin isn't optimizing; it's optimizing without evidence.**

### Where the time actually is, ranked by frequency

In my experience of Spring applications, "slow" resolves to these in roughly this order:

| Rank | Cause | Symptom |
|---|---|---|
| 1 | **Database** — N+1, missing index, bad query plan | High latency, low CPU, connection pool busy |
| 2 | **Blocking I/O + thread starvation** | High latency, **low CPU**, pending pool requests |
| 3 | **Missing/ineffective caching** | Repeated identical work |
| 4 | **Chatty remote calls** — 20 sequential HTTP calls per request | Latency ≈ sum of round trips |
| 5 | **Serialization of huge payloads** | High CPU, large heap churn |
| 6 | GC pressure / memory | Periodic latency spikes |
| 7 | Lock contention | Latency rises non-linearly with concurrency |
| 8 | Actual algorithmic CPU cost | Sustained high CPU |

**Notice ranks 1 and 2: the two most common causes both present as high latency with LOW CPU.** If your CPU is at 15% and your p99 is 3 seconds, **you are waiting, not computing** — and no JVM flag will help. **That single diagnostic heuristic is worth more than any tuning knowledge**, and it's a great thing to say in an interview.

---

## 3. Localizing the problem

### Step 1 — the observability stack (Lesson 35)

Your trace waterfall answers "which layer?" in seconds:
```
POST /api/v1/monitors                      2400ms
├─ MonitorController.create                2395ms
│  ├─ SELECT organization                     8ms
│  ├─ ★ 47 × SELECT check_result           1850ms   ← N+1. Found it in 5 seconds.
│  └─ INSERT monitor                         12ms
```

**If you have tracing, start there. Always.** It's the cheapest possible localization.

### Step 2 — thread dumps (for hangs and starvation)

```bash
jcmd <pid> Thread.print > threads.txt
```
Or `/actuator/threaddump`. Read it for:
- **Many threads in the same stack frame** → they're all blocked on the same thing. That frame *is* your bottleneck.
- `BLOCKED` on a monitor → lock contention
- `WAITING` on `HikariPool.getConnection` → **pool exhaustion** (Lesson 19)
- `RUNNABLE` in `SocketRead` → waiting on a network response (which counts as blocked, not busy)

**Take three dumps 10 seconds apart.** Threads present in all three are genuinely stuck; threads that move are just working. **That technique — comparing successive dumps — is how you distinguish a hang from normal load**, and very few people know it.

### Step 3 — profiling (for CPU and allocation)

**Async-profiler** is the tool to know. It uses `AsyncGetCallTrace`, so it doesn't suffer from the safepoint bias that afflicts traditional samplers:

```bash
./profiler.sh -d 60 -e cpu -f cpu.html <pid>       # CPU flame graph
./profiler.sh -d 60 -e alloc -f alloc.html <pid>   # allocation flame graph
./profiler.sh -d 60 -e wall -f wall.html <pid>     # ★ WALL clock — includes blocked time
```

**`-e wall` is the one people miss.** CPU profiling shows where you *burn* CPU; wall-clock profiling shows where you *spend time*, including blocked on I/O. **For a latency problem, wall-clock is usually the right mode** — CPU profiling of an I/O-bound service shows you almost nothing.

**Reading a flame graph:** width = time spent, and you read **horizontally** for cost, not vertically for depth. A wide plateau is your hot spot. Deep-but-narrow stacks are irrelevant.

**JDK Flight Recorder** is built in, extremely low overhead (~1%), and safe to run continuously in production:
```bash
java -XX:StartFlightRecording=duration=120s,filename=rec.jfr -jar vigil.jar
jcmd <pid> JFR.start name=diag settings=profile duration=120s filename=rec.jfr
```
Open in **JDK Mission Control**. JFR gives you GC events, allocation profiles, lock contention, exceptions, JDBC statements, and thread states in one file. **"I'd take a JFR recording" is a strong answer** because it's the production-safe option — you can enable it on a live pod without meaningful impact.

### Step 4 — database-side

```sql
-- Postgres: your slowest queries by total time
SELECT calls, mean_exec_time, total_exec_time, query
FROM pg_stat_statements
ORDER BY total_exec_time DESC LIMIT 20;
```

**Order by `total_exec_time`, not `mean_exec_time`.** A 5ms query called 100,000 times costs far more than a 2-second query called once — and the fast-but-frequent one is invisible if you sort by mean. **That's the N+1 problem's signature in `pg_stat_statements`**, and knowing to look for it is a genuinely useful skill.

Then `EXPLAIN (ANALYZE, BUFFERS)` on the offenders (Lesson 24).

---

## 4. Memory and garbage collection

### The heap layout

```
┌─────────────────── JVM process memory ──────────────────────┐
│  HEAP                                                        │
│  ┌──────────── Young gen ────────────┐  ┌── Old gen ──────┐  │
│  │  Eden   │ Survivor0 │ Survivor1   │  │  long-lived     │  │
│  └─────────────────────────────────────┘  └────────────────┘  │
│  NON-HEAP                                                     │
│  Metaspace (class metadata) · Code cache (JIT) · Thread stacks │
│  Direct byte buffers (NIO) · GC structures                     │
└───────────────────────────────────────────────────────────────┘
```

**The critical insight for containers: heap is NOT total memory.** A container limit of 512MB with `-Xmx512m` **will** be OOM-killed, because metaspace, code cache, thread stacks (~1MB each), and direct buffers all live outside the heap. **Reserve 25–30% of the container limit for non-heap.**

**This is the #1 cause of "my Spring Boot pod keeps getting OOMKilled"** and it looks nothing like a Java `OutOfMemoryError` — the JVM is killed by the kernel with exit code 137 and **no stack trace, no heap dump, nothing in your logs.** That silence is why it's so confusing.

```bash
# ❌ Guaranteed OOMKill in a 512MB container
java -Xmx512m -jar vigil.jar

# ✅ Let the JVM read the container limit and take a percentage
java -XX:MaxRAMPercentage=75 -jar vigil.jar
```

**`-XX:MaxRAMPercentage` is the correct modern approach** — it's container-aware (the JVM reads cgroup limits) and survives someone changing the pod's memory limit without also editing the JVM flags. **Hardcoded `-Xmx` in a container is a latent bug waiting for a resource-limit change.**

### Garbage collectors

| GC | Characteristics | Use |
|---|---|---|
| **Serial** | One thread, small footprint | Tiny containers (<256MB), single CPU |
| **Parallel** | Multi-threaded, best **throughput**, longer pauses | Batch jobs where pauses don't matter |
| **G1** (default since 11) | Balanced, pause-time target, region-based | ✅ **Default — leave it alone** |
| **ZGC** | Sub-millisecond pauses, large heaps | Latency-critical, heaps >8GB |
| **Shenandoah** | Similar goals to ZGC | Same niche |

```bash
-XX:+UseG1GC -XX:MaxGCPauseMillis=200        # G1 with a pause target
-XX:+UseZGC -XX:+ZGenerational               # ZGC, generational mode (JDK 21+)
```

**The honest guidance: don't change the GC.** G1 is a good default and the overwhelming majority of "GC problems" are actually **allocation problems** — code creating far more garbage than necessary. Changing collectors treats the symptom.

**Diagnose before tuning:**
```bash
-Xlog:gc*:file=gc.log:time,uptime:filecount=5,filesize=10M
```
Then read it (or upload to GCEasy / gcplot). **What to look for:**
- **Frequent full GCs** → the heap is too small, or you have a leak
- **Long pauses (>1s)** → heap too large for the collector, or humongous allocations
- **High allocation rate** → the real problem is your code, not the GC
- **Old gen growing monotonically after each full GC** → **a memory leak.** This is the definitive signature.

**Metrics to alert on** (Lesson 35): `jvm_gc_pause_seconds`, `jvm_memory_used_bytes{area="heap"}`, `jvm_gc_memory_allocated_bytes_total`.

### Finding a memory leak

```bash
jcmd <pid> GC.heap_info                       # quick overview
jcmd <pid> GC.class_histogram | head -30      # ★ what's taking the space?
jcmd <pid> GC.heap_dump /tmp/heap.hprof       # full dump — pauses the JVM briefly
```

Then open in **Eclipse MAT** and run "Leak Suspects." MAT's **dominator tree** is the key view: it shows which object *retains* the most memory, not merely which class has the most instances.

**Always capture a heap dump on OOM in production:**
```bash
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/var/log/heapdump.hprof
-XX:+ExitOnOutOfMemoryError                   # ★ don't limp along in an undefined state
```
**`ExitOnOutOfMemoryError` matters:** after an `OutOfMemoryError`, some threads have died mid-operation and the application is in an unknown state. **Dying cleanly so the orchestrator restarts you is better than serving corrupted responses.**

**⚠️ And remember: a heap dump contains every password, token and PII value in memory** (Lesson 28 §5). Treat the file as a production secret.

**The common leaks in Spring applications:**
1. **Unbounded caches** — `ConcurrentMapCacheManager` with no eviction (Lesson 33)
2. **`ThreadLocal` not cleared** — with pooled threads, the value is retained forever (Lessons 14, 32)
3. **Growing static collections** — a `static List` used as a registry
4. **Unclosed resources** — streams, connections, `Closeable`s
5. **Listeners never deregistered**
6. **Large persistence contexts** — bulk `save()` without `clear()` (Lesson 24)

---

## 5. Startup time

Startup matters for autoscaling responsiveness, serverless cold starts, and rolling-deploy speed. A typical Spring Boot app starts in **2–6 seconds**.

**Diagnose first:**
```java
@Bean
ApplicationStartup applicationStartup() { return new BufferingApplicationStartup(2048); }
```
Then `/actuator/startup` gives a timed breakdown of every startup step. **Measure before optimizing** — the same rule as everything else.

### The options, cheapest first

| Technique | Improvement | Cost |
|---|---|---|
| **Trim dependencies** | Fewer auto-configurations to evaluate | Free. Do this first. |
| `spring.main.lazy-initialization=true` | 30–50% faster | **Errors move to first request** (Lesson 08) |
| Exclude unused auto-configurations | Modest | Must know what you need |
| **CDS / AOT cache** (JDK 12+/24+) | **20–40% faster** | ✅ **Best value — no code change** |
| Spring AOT processing | ~20–30% | A build step |
| **GraalVM native image** | **~50ms startup**, ~50MB RSS | Long builds, reflection config, no dynamic classloading |

### CDS — the underrated one

**Class Data Sharing** pre-parses class metadata into an archive the JVM memory-maps at startup instead of parsing class files again.

```bash
# Spring Boot 3.3+ makes this a documented, easy workflow
java -XX:ArchiveClassesAtExit=app.jsa -Dspring.context.exit=onRefresh -jar vigil.jar
java -XX:SharedArchiveFile=app.jsa -jar vigil.jar
```

**20–40% faster startup, no code changes, no reflection configuration, and identical runtime behaviour.** JDK 24 adds AOT caching (Project Leyden) which extends this further.

**CDS is the right answer for "how would you improve startup time?" in most cases** — it's nearly free, and far less disruptive than a native image. Mentioning it instead of jumping straight to GraalVM shows calibration.

### GraalVM native image — know the trade-offs

```bash
./mvnw -Pnative native:compile
```

**What you get:** ~50ms startup (vs ~3s), ~50MB memory (vs ~300MB), a single binary with no JVM.

**What you give up — and this is what interviews probe:**
- **No dynamic class loading**, so all reflection, proxies and resources must be known at build time. This is why **Spring Boot 3's AOT processing exists**: it computes bean definitions at build time and generates reflection hints (Lesson 04 §Part 2).
- **Long builds** — 3–10 minutes, and memory-hungry.
- **No JIT**, so *peak* throughput is often **lower** than a warmed-up JVM. **Native images start fast and can run slower.**
- Some libraries need manual hints or don't work at all.
- Different debugging and profiling tooling.

**When it's worth it:** serverless/FaaS where cold start dominates cost, CLI tools, very high pod density, scale-to-zero.
**When it isn't:** long-running services where a 3-second startup is irrelevant and peak throughput matters.

**The calibrated answer:** *"Native image is a startup-and-footprint optimization, not a throughput one — a warmed-up JVM often beats it on peak performance. For a long-running service I'd use CDS or AOT caching instead; I'd reach for native when cold start is the actual cost driver, like serverless."*

---

## 6. Application-level performance

### Database (usually the whole answer)

Re-read Lesson 24. The hit list, in order of impact:
```
1. Fix the N+1 (default_batch_fetch_size: 25 as a global safety net)
2. Add the right composite index (equality columns first, then range/sort)
3. Use projections — stop loading entities to read three columns
4. Slice instead of Page (drops the count(*) query)
5. Batch writes: hibernate.jdbc.batch_size + SEQUENCE ids + reWriteBatchedInserts=true
6. Push aggregation into SQL rather than into Java
7. Size the pool correctly — smaller is usually faster
```

### Connection pool sizing, restated

**Bigger is usually slower** (Lesson 19). `(cores × 2) + spindles` ≈ 10. **And the multi-instance trap: 10 pods × 20 connections = 200 against a `max_connections` of 100.**

### Serialization

```java
// Don't serialize what nobody reads.
@JsonInclude(NON_NULL)                        // omit nulls
projections instead of full entities           // ★ the biggest win (Lesson 22)
Slice instead of Page                          // drops count(*)
streaming for large exports                    // StreamingResponseBody, not a 500MB String
```

**Response compression is nearly free:**
```yaml
server:
  compression:
    enabled: true
    mime-types: application/json,text/html,text/css,application/javascript
    min-response-size: 1024
```
JSON compresses 5–10×. **One config block, and your bandwidth bill and mobile latency both drop.** Notably absent from most projects.

### Virtual threads (Lesson 18)

```properties
spring.threads.virtual.enabled=true
```
For an I/O-bound service, this is often the single largest throughput improvement available for one line of configuration. **But remember it relocates the bottleneck** — your connection pool, downstream rate limits, and memory held by in-flight work become the constraint. Verify with a load test rather than assuming.

### Reduce round trips

```java
// ❌ 20 sequential HTTP calls = 20 × latency
for (var m : monitors) results.add(client.check(m));

// ✅ Concurrent — total time ≈ the slowest single call
var futures = monitors.stream()
        .map(m -> CompletableFuture.supplyAsync(() -> client.check(m), checkExecutor))
        .toList();
var results = futures.stream().map(CompletableFuture::join).toList();
```
**Same principle in the database:** one query with `IN (...)` beats 20 queries by ID. **Latency is dominated by round trips far more often than by processing.**

---

## 7. Load testing

**You cannot claim a performance improvement without a load test.** And a load test you ran once is worth more in an interview than any amount of theory.

```javascript
// k6 — script.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '30s', target: 50 },     // ramp up
    { duration: '2m',  target: 50 },     // ★ SUSTAIN — where the interesting failures happen
    { duration: '30s', target: 200 },    // spike
    { duration: '1m',  target: 200 },
    { duration: '30s', target: 0 },
  ],
  thresholds: {
    http_req_duration: ['p(95)<300', 'p(99)<1000'],
    http_req_failed:   ['rate<0.01'],
  },
};

export default function () {
  const res = http.get('http://localhost:8080/api/v1/monitors',
      { headers: { Authorization: `Bearer ${__ENV.TOKEN}` } });
  check(res, { 'status 200': (r) => r.status === 200 });
  sleep(1);
}
```
```bash
k6 run -e TOKEN=$TOKEN script.js
```

**The rules that make a load test meaningful:**

1. **Sustain the load.** A 10-second burst tells you nothing about connection leaks, memory growth, or GC behaviour. **Run for minutes.** Most interesting failures are cumulative.
2. **Use realistic data volumes.** 100 rows and 10 million rows produce completely different query plans. Testing against an empty database is testing nothing.
3. **Test against production-like infrastructure.** Your laptop's SSD and zero network latency flatter you enormously.
4. **Watch the *system* while it runs**, not just the client's numbers: GC log, thread count, pool saturation, CPU, and — crucially — **the database's** CPU and lock waits. The bottleneck is often outside the JVM.
5. **Find the knee.** Ramp until latency degrades non-linearly. That inflection is your real capacity, and it's the number that matters for capacity planning.

**Tools:** **k6** (JS, great DX, CI-friendly — the modern default), **Gatling** (Scala/Java DSL, excellent reports), **JMeter** (GUI, ubiquitous, dated), **wrk/hey** (quick single-endpoint checks).

**Little's Law is worth knowing** because it lets you sanity-check results arithmetically:
```
concurrency = throughput × latency
→ 200 concurrent requests at 500ms latency = 400 req/s maximum
```
If your measured throughput doesn't match that, one of your numbers is wrong. **Being able to reason about capacity with a formula rather than a benchmark is a nice differentiator.**

---

## 8. Build & break — measure and improve Vigil

**1. Establish a baseline.** Seed realistic data: 5,000 monitors, **2,000,000 check results** (use `JdbcClient.batchUpdate` with `reWriteBatchedInserts=true` — Lesson 19). Then k6-test your three main endpoints and **record p50/p95/p99 and throughput in a table.** That table is your reference for everything below.

**2. ⭐ Find and fix the biggest problem, properly.** Run the load test with tracing on. Find the slowest span. **Predict the cause before looking.** Then:
- Fix it (index / fetch join / projection / cache)
- **Re-run the same load test**
- Record the improvement

**Repeat three times.** You now have a documented performance-optimization narrative — *"p99 went from 2.4s to 180ms across three changes: a composite index, a projection, and batch fetching"* — which is one of the most compelling things you can bring to an interview. **Put the table in Vigil's README.**

**3. Reproduce thread starvation and read the dump.** Set `server.tomcat.threads.max=10`, make an endpoint sleep 2s, drive 100 concurrent requests. Then:
```bash
jcmd <pid> Thread.print | grep -A5 "http-nio"
```
**Count threads in the same frame.** Take three dumps 10s apart and identify which threads are genuinely stuck. Then enable virtual threads and repeat.

**4. Profile with async-profiler.** Under load, capture both `-e cpu` and `-e wall` flame graphs. **Compare them.** For Vigil's I/O-bound checks the CPU graph will look nearly empty while the wall graph shows exactly where you wait. **That contrast teaches the difference better than any explanation.**

**5. Read a GC log.** Run with `-Xlog:gc*:file=gc.log:time,uptime` under sustained load. Upload to GCEasy. Note pause times, frequency, and allocation rate. **Then set `-Xmx128m` and watch it thrash** — frequent full GCs, latency spikes, and eventually an OOM. **Seeing GC pathology once makes it recognizable forever.**

**6. ⭐ Reproduce the container OOMKill.**
```bash
docker run -m 512m -e JAVA_TOOL_OPTIONS="-Xmx512m" vigil:latest
```
Drive load until it's killed. **Note there is no Java exception, no heap dump, nothing in the logs** — just exit code 137. `docker inspect <container> | grep OOMKilled`. **Then fix it with `-XX:MaxRAMPercentage=75` and confirm stability.** This is the most valuable container-JVM lesson there is.

**7. Create a real memory leak.**
```java
private static final List<CheckResult> LEAK = new ArrayList<>();   // never cleared
```
Add to it on every check. Watch `jvm_memory_used_bytes` climb in Grafana. Then:
```bash
jcmd <pid> GC.class_histogram | head -20        # find the culprit
jcmd <pid> GC.heap_dump /tmp/leak.hprof
```
Open in Eclipse MAT, run Leak Suspects, and **confirm it names your list.** **Doing this once means you can diagnose a real leak with confidence.**

**8. Measure startup, then improve it.** Add `BufferingApplicationStartup` and read `/actuator/startup`. Record the baseline. Then measure:
- `spring.main.lazy-initialization=true`
- CDS with `-XX:ArchiveClassesAtExit` / `-XX:SharedArchiveFile`

**Record all three numbers.** Then reason about why lazy init is a poor production choice despite being the fastest (Lesson 08).

**9. Optional but impressive: build a native image.**
```bash
./mvnw -Pnative native:compile
```
Expect a long build and some reflection hints to add. Then compare: startup time, RSS, **and peak throughput under sustained load.** **Confirm for yourself that the native image starts ~50× faster and has lower peak throughput.** Being able to state that trade-off from your own measurements is far more persuasive than quoting a blog post.

**10. Enable compression** and measure the response size of a 100-item monitor list with and without it. **Note the ratio** — usually 5–10×.

**11. Find the knee.** Ramp k6 from 10 to 1,000 VUs and plot latency against throughput. **Identify the inflection point.** That's Vigil's capacity per instance, and it's the number you'd use for capacity planning. Then sanity-check it against Little's Law.

**12. Guard against regression.** Add a k6 threshold check to CI (against a seeded test environment) so a performance regression fails the build the way a broken test does. **Very few projects do this**, and it's the "GUARD" step of the method actually implemented.

### Explain out loud (60s)

1. The method, in four words, and why the order matters.
2. What high latency with low CPU tells you, and what it rules out.
3. The three-thread-dump technique, and what it distinguishes.
4. Why `-Xmx` equal to the container limit guarantees an OOMKill, and what that failure looks like.
5. Why swapping garbage collectors is usually the wrong fix.

---

## 9. Production rules

| Do / Never | Why |
|---|---|
| **Never change anything without a measurement, and never accept a fix without a re-measurement** | Most "performance work" is people changing what they believe is slow; the sin isn't optimizing, it's optimizing without evidence |
| **Read the CPU first: high latency plus low CPU means I/O, not computation** | It immediately rules out GC tuning, algorithmic work and thread-pool sizing, which is most of what people try first |
| **Start from the trace when you have one** | It is the cheapest possible localization, and it tells you which span to profile instead of profiling everything |
| **Take three thread dumps ten seconds apart** | Threads present in all three are genuinely stuck; threads that move are just working — that comparison is what distinguishes a hang from load |
| **Profile with wall-clock, not just CPU, for latency problems** | CPU profiling an I/O-bound service shows you almost nothing, because the time is spent blocked rather than burning cycles |
| **Read flame graphs horizontally** | Width is time; a wide plateau is your hot spot, and a deep narrow stack is usually irrelevant |
| **Sort `pg_stat_statements` by `total_exec_time`, not mean** | A 5ms query run 100,000 times costs far more than one 2-second query, and sorting by mean hides exactly the N+1 signature |
| **Never set `-Xmx` equal to the container limit** | Metaspace, code cache, thread stacks and direct buffers live outside the heap — reserve 25–30% or the kernel kills you |
| **Use `-XX:MaxRAMPercentage` instead of a hardcoded `-Xmx`** | It reads the cgroup limit, so changing the pod's memory no longer silently leaves a latent OOMKill behind |
| **Recognize exit code 137 as an OOMKill, not a Java `OutOfMemoryError`** | There is no stack trace, no heap dump and nothing in your logs — the silence is what makes it confusing |
| **Leave G1 alone** | Most "GC problems" are allocation problems, and swapping collectors treats the symptom while the code keeps creating the garbage |
| **Set `HeapDumpOnOutOfMemoryError` and `ExitOnOutOfMemoryError`** | After an OOM some threads died mid-operation, so dying cleanly for the orchestrator beats serving corrupted responses |
| **Treat a heap dump as a production secret** | It contains every password, token and PII value that was in memory at the moment it was taken |
| **Trim dependencies before reaching for AOT or native images** | Fewer auto-configurations is free and reversible; a native image costs you reflection configuration and long builds |
| **Put a load-test threshold in CI** | It makes a performance regression fail the build the way a broken test does, which is the one step of the method people skip |

---

## 10. Interview traps

**Q1. "The app is slow." How do you approach it?**
Define what slow means (endpoint, percentile, target), reproduce and measure, localize the layer with tracing/thread dumps/profiling, fix one thing, re-measure, then add a guard so it can't regress. Never change anything without a measurement, and never accept a fix without a re-measurement.

**Q2. High latency with low CPU — what does that tell you?**
You're **waiting, not computing.** Almost always blocking I/O: slow database queries, an N+1, a slow downstream call, or thread/connection pool exhaustion. No JVM or GC tuning will help. Confirm with a thread dump — many threads in the same frame identify the bottleneck.

**Q3. What's the most common cause of slow Spring applications?**
The database: N+1 queries, a missing index, or loading entities where a projection would do. Second is blocking I/O plus thread starvation. Both present as high latency with low CPU.

**Q4. Why do Spring Boot pods get OOMKilled?**
Because heap isn't total memory. Metaspace, code cache, thread stacks (~1MB each) and direct buffers live outside the heap, so `-Xmx512m` in a 512MB container is guaranteed to be killed by the kernel — **exit code 137, no Java exception, no heap dump, nothing in the logs.** Use `-XX:MaxRAMPercentage=75` so the JVM reads the cgroup limit and leaves headroom.

**Q5. Which GC would you choose?**
G1 (the default) for almost everything. ZGC for sub-millisecond pause requirements or heaps over ~8GB. **But most "GC problems" are allocation problems** — code producing excessive garbage — so I'd read a GC log and profile allocations before changing collectors.

**Q6. How do you find a memory leak?**
GC log shows old gen growing monotonically after each full GC — that's the signature. Then `jcmd GC.class_histogram` for a quick view, and a heap dump analyzed in Eclipse MAT using the dominator tree to find what *retains* the memory. Common culprits: unbounded caches, uncleared `ThreadLocal`s on pooled threads, growing static collections, unclosed resources, and oversized persistence contexts.

**Q7. What's `-XX:+HeapDumpOnOutOfMemoryError` for, and what's the catch?**
It captures a heap dump when OOM occurs, so you can diagnose a crash you can't reproduce. Pair it with `-XX:+ExitOnOutOfMemoryError` so the process dies cleanly rather than limping in an undefined state. **The catch: the dump contains every credential and PII value in memory — treat it as a production secret.**

**Q8. How would you improve startup time?**
Measure first with `BufferingApplicationStartup` and `/actuator/startup`. Then: trim dependencies, **CDS / AOT caching** (20–40% for no code change — the best value), Spring AOT processing, and GraalVM native image if cold start genuinely dominates. Lazy initialization is fastest but moves errors to the first request, so it's a poor production default.

**Q9. GraalVM native image trade-offs?**
Gains: ~50ms startup, ~50MB memory, single binary. Costs: no dynamic class loading (so all reflection needs build-time hints, which is why Spring AOT exists), long memory-hungry builds, **lower peak throughput because there's no JIT**, some library incompatibility, and different tooling. It's a startup-and-footprint optimization, not a throughput one.

**Q10. What tools would you use to profile a production JVM?**
**JFR** first — built in, ~1% overhead, safe to run live, and it captures GC, allocation, locks, exceptions and JDBC in one recording (open in JMC). **Async-profiler** for flame graphs, using `-e wall` for latency problems since CPU mode shows nothing for I/O-bound work. Thread dumps (three, spaced 10s apart) for hangs. `pg_stat_statements` ordered by **total** time for the database.

**Q11. Why order `pg_stat_statements` by total time rather than mean?**
A 5ms query called 100,000 times costs far more than a 2-second query called once, and it's invisible when sorted by mean. High-count-low-mean is exactly the N+1 signature.

**Q12. What makes a load test valid?**
Sustained load (minutes, not seconds — leaks and GC behaviour are cumulative), production-like data volumes (query plans change with size), production-like infrastructure, monitoring the *system* including the database during the run, and ramping to find the knee where latency degrades non-linearly. Little's Law (`concurrency = throughput × latency`) lets you sanity-check the numbers.

**Q13. Is "premature optimization is the root of all evil" good advice?**
It's usually misquoted. Knuth's point was that you should optimize the critical few percent — **once you've identified it by measurement.** The sin isn't optimizing; it's optimizing without evidence. Deliberately choosing an O(n²) algorithm because "we'll optimize later" isn't avoiding premature optimization, it's a design mistake.

---

## 11. Retention check

1. Give the six-step performance method, and the two rules that bracket it.
2. High latency, 12% CPU. What are your first three hypotheses, and what do you run?
3. Why is `-Xmx512m` in a 512MB container guaranteed to be killed, and what do you use instead?
4. What's the definitive GC-log signature of a memory leak?
5. Why does a native image start faster but sometimes run slower?
6. Name three things that make a load test invalid, and the formula you'd use to sanity-check the result.

---

## Module 9 complete ✅

You can now run a service: background work that's safe across instances, caching that doesn't serve wrong data, resilience that contains other people's failures, observability that tells you what's happening, and a method for diagnosing performance rather than guessing at it.

**Next:** [Lesson 37 — Messaging with Kafka](../10-microservices/37-messaging-kafka.md) — asynchronous communication done properly, with the delivery guarantees spelled out.

# Lesson 33 — Caching done right

> **Why this lesson exists:** Caching is the easiest big performance win and the easiest way to serve wrong data. `@Cacheable` is three characters of effort and a permanent correctness obligation. This lesson covers the abstraction, Redis, the invalidation problem, the stampede problem, and — crucially — when *not* to cache.

**Time:** ~65 minutes · **Prereq:** Lesson 09 (proxies), Lesson 24 (caching layers)

---

## 1. Before you cache: measure

**The single most common caching mistake is caching instead of fixing.**

```java
// ❌ A 2-second query, "fixed" by caching it
@Cacheable("monitorList")
public List<MonitorSummary> list(UUID orgId) {
    return repo.findByOrganizationId(orgId);      // an N+1 problem, or a missing index
}
```

You've hidden a 2-second query behind a cache. Now: the first request after every deploy is slow, cache expiry produces random 2-second spikes, and a cache miss under load produces a stampede (§5). **And the actual bug — the missing index — is still there.**

**The order of operations:**
1. **Measure.** Where does the latency actually go? (Lesson 35)
2. **Fix the query.** Index, fetch join, projection (Lesson 24).
3. **Cache only if it's still slow, or the load is genuinely redundant.**

**Then ask three questions before adding `@Cacheable`:**
- **How stale can this be?** If the answer is "not at all," don't cache it.
- **What's the read:write ratio?** Caching a write-heavy resource means constant invalidation and near-zero hit rate.
- **What breaks if it's wrong?** A stale monitor name is cosmetic. A stale *permission* is a security hole.

**Saying "I'd profile before caching, because caching a slow query hides the bug and creates latency spikes at expiry" is a genuinely senior opening** to any caching question.

---

## 2. The Spring Cache abstraction

```java
@Configuration
@EnableCaching                          // ★ required, or @Cacheable is silently ignored
public class CacheConfig { }
```

```java
@Service
public class StatusPageService {

    @Cacheable(value = "statusPages", key = "#slug")
    public StatusPageResponse render(String slug) { }

    @CachePut(value = "statusPages", key = "#result.slug")
    public StatusPageResponse update(UUID id, UpdateRequest req) { }   // always executes, updates cache

    @CacheEvict(value = "statusPages", key = "#slug")
    public void delete(String slug) { }

    @CacheEvict(value = "statusPages", allEntries = true)
    public void invalidateAll() { }

    @Caching(evict = {                                        // multiple operations
        @CacheEvict(value = "statusPages", key = "#result.slug"),
        @CacheEvict(value = "monitorSummaries", key = "#result.orgId")
    })
    public Monitor save(Monitor monitor) { }

    @Cacheable(value = "uptime", key = "#monitorId", unless = "#result.isUnknown()")
    public Uptime uptime(UUID monitorId) { }                  // don't cache unknown results

    @Cacheable(value = "reports", condition = "#days <= 30")  // only cache small ranges
    public Report report(UUID orgId, int days) { }
}
```

| Annotation | Behaviour |
|---|---|
| `@Cacheable` | Check cache → hit? return it, **method never runs**. Miss? run and store. |
| `@CachePut` | **Always runs** the method, then stores the result |
| `@CacheEvict` | Removes entries (`beforeInvocation = true` to evict even on exception) |
| `@Caching` | Group several operations |
| `@CacheConfig` | Class-level defaults (`cacheNames`, `keyGenerator`) |

**`condition` vs `unless`:** `condition` is evaluated **before** the method (so it can't see the result); `unless` is evaluated **after** (so it can, via `#result`). Use `condition` to skip caching by input, `unless` to skip by output.

### **It's proxy-based** — every Lesson 09 rule applies

```java
@Service
public class ReportService {
    public Report monthly(UUID orgId) {
        return daily(orgId);                 // ❌ self-invocation → CACHE NEVER USED
    }
    @Cacheable("daily") public Report daily(UUID orgId) { }
}
```

**Silent.** No error, no warning, just zero cache hits and a mystery about why the cache "isn't working." Same for `private` and `final` methods. **This is the #1 "@Cacheable doesn't work" cause**, and it's the same root cause as the `@Transactional` and `@Async` versions.

### Cache keys — get this right or you'll leak data

**Default key generation (`SimpleKeyGenerator`):** no params → `SimpleKey.EMPTY`; one param → the param itself; multiple → a `SimpleKey` of all params.

```java
// ⚠️⚠️ CATASTROPHIC in a multi-tenant system
@Cacheable("monitors")
public List<MonitorSummary> listForCurrentOrg() {          // no parameters!
    return repo.findByOrganizationId(TenantContext.require());
}
// The key is SimpleKey.EMPTY for EVERY tenant.
// Org A populates the cache; org B gets ORG A'S DATA. A cross-tenant breach via caching.
```

```java
// ✅ Every tenant-scoped cache key MUST include the tenant
@Cacheable(value = "monitors", key = "#orgId")
public List<MonitorSummary> list(UUID orgId) { }

// ✅ Or with SpEL pulling from the security context
@Cacheable(value = "monitors", key = "@tenantContext.currentOrgId()")
public List<MonitorSummary> listForCurrentOrg() { }
```

**Rule: if a method's result depends on anything not in its parameters — tenant, user, locale, feature flag — that thing must be in the cache key.** This is the most dangerous caching bug there is, because it's a *data leak* rather than a performance issue, and it doesn't show up until two tenants use the feature concurrently.

**Also: never use a mutable object as a key** (its `hashCode` changes — Lesson 02), and keep keys small and readable so you can inspect them in Redis.

```java
// Custom key generator when you need consistency across many methods
@Bean("tenantAwareKeyGenerator")
KeyGenerator tenantAwareKeyGenerator() {
    return (target, method, params) -> TenantContext.require() + ":" + method.getName()
            + ":" + Arrays.deepHashCode(params);
}
```

---

## 3. Cache providers

| Provider | Scope | Use |
|---|---|---|
| `ConcurrentMapCacheManager` | In-JVM, **no eviction, no TTL** | The default with no provider on the classpath. **Never in production** — unbounded, so it's a memory leak |
| **Caffeine** | In-JVM, size + TTL, excellent hit rates | ✅ Single instance, or a short-TTL L1 in front of Redis |
| **Redis** | **Shared across instances** | ✅ **The default choice for a multi-instance service** |
| Hazelcast / Infinispan | Distributed, embedded | Clustered caches with computation |
| EhCache 3 (JSR-107) | In-JVM, disk overflow | Legacy-friendly |

### The critical distinction: local vs distributed

```
LOCAL (Caffeine)                          DISTRIBUTED (Redis)
┌──────┐ ┌──────┐ ┌──────┐                ┌──────┐ ┌──────┐ ┌──────┐
│ pod1 │ │ pod2 │ │ pod3 │                │ pod1 │ │ pod2 │ │ pod3 │
│cache │ │cache │ │cache │                └───┬──┘ └───┬──┘ └───┬──┘
└──────┘ └──────┘ └──────┘                    └────────┼────────┘
                                                   ┌───▼───┐
Three independent caches.                          │ Redis │
Evicting on pod1 leaves pod2 and pod3              └───────┘
serving STALE data. 💀                        One cache. Evict once, gone everywhere. ✅
```

**This is the reason to reject Hibernate's second-level cache in a multi-instance deployment** (Lesson 24) and the reason Redis is the default answer. **With a local cache and N instances, an eviction only affects 1/N of your traffic** — so a user sees updated data on some requests and stale data on others, apparently at random. That's a genuinely awful bug to debug, and describing it is a strong answer.

### Redis configuration

```xml
<dependency><groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-redis</artifactId></dependency>
```

```yaml
spring:
  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: 6379
      password: ${REDIS_PASSWORD:}
      timeout: 2s                      # ★ command timeout
      connect-timeout: 2s
      lettuce:
        pool:
          max-active: 16
          max-idle: 8
          min-idle: 2
```

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    RedisCacheManager cacheManager(RedisConnectionFactory factory, ObjectMapper objectMapper) {
        var json = new GenericJackson2JsonRedisSerializer(objectMapper);   // ★ see §3.1

        var defaults = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(10))                 // ★ ALWAYS set a TTL
                .serializeKeysWith(SerializationPair.fromSerializer(new StringRedisSerializer()))
                .serializeValuesWith(SerializationPair.fromSerializer(json))
                .disableCachingNullValues()                       // ★ but see §5 on null caching
                .prefixCacheNameWith("vigil:v1:");                // ★ see below

        return RedisCacheManager.builder(factory)
                .cacheDefaults(defaults)
                .withCacheConfiguration("statusPages",
                        defaults.entryTtl(Duration.ofSeconds(30)))     // hot, must be fresh
                .withCacheConfiguration("uptime",
                        defaults.entryTtl(Duration.ofMinutes(5)))
                .withCacheConfiguration("planLimits",
                        defaults.entryTtl(Duration.ofHours(24)))       // changes almost never
                .transactionAware()                                    // ★ see §4
                .build();
    }
}
```

**Three configuration details that matter more than they look:**

**1. `prefixCacheNameWith("vigil:v1:")`** — the version prefix is a **deployment safety feature**. Change a DTO's shape and old cached entries can't be deserialized (§3.1). Bumping the prefix to `v2` makes the entire old cache unreachable — an instant, safe, zero-downtime cache flush. **Far better than `FLUSHDB`, which affects everything sharing the Redis instance.** This is a genuinely good detail to mention.

**2. Per-cache TTLs.** A 24-hour TTL for plan limits and 30 seconds for status pages reflects how often each actually changes. One global TTL means either stale data or a useless hit rate.

**3. Timeouts.** **Redis must have a command timeout.** Without one, a Redis hiccup blocks your request threads indefinitely — the cache you added for performance takes your whole application down (Lessons 14, 19). §6 covers this properly.

### 3.1 Serialization — where the real operational pain is

**The default is JDK serialization**, which is a bad choice:
- Values are unreadable in `redis-cli`, so you can't debug
- **Java serialization is a deserialization-attack surface** (Lesson 28, A08)
- Adding a field to a DTO throws `InvalidClassException` on every existing entry

**Use JSON.** But know the trap it brings:

```java
// Cached with Jackson today:
record MonitorSummary(UUID id, String name, MonitorStatus status) {}

// You deploy a version that adds a field:
record MonitorSummary(UUID id, String name, MonitorStatus status, BigDecimal uptime) {}
// → old entries deserialize with uptime = null → NullPointerException in production
// → or, with strict config, deserialization exceptions on every cache hit
```

**Mitigations, and you should be able to name them:**
1. **Version the cache prefix on any breaking DTO change** (`vigil:v2:`) — the simplest and most reliable
2. Configure Jackson to ignore unknown properties and tolerate missing ones (handles additive changes)
3. Only cache stable, purpose-built cache DTOs — **never entities**, whose shape changes with your schema
4. Treat the cache as **disposable**: the application must be correct with a completely empty cache, always

**That last point is the principle:** **a cache must never be load-bearing for correctness.** If your app breaks when Redis is empty or unavailable, you haven't built a cache — you've built an undocumented database with no durability.

---

## 4. Invalidation — the actual hard problem

> *"There are only two hard things in computer science: cache invalidation and naming things."* — Phil Karlton

### Strategy 1 — TTL only (simplest, use it by default)

```java
@Cacheable(value = "uptime", key = "#monitorId")     // TTL 5 minutes
```
Accept staleness up to the TTL. **No invalidation logic to get wrong.** For anything where "up to N seconds old" is acceptable — dashboards, uptime percentages, status pages, aggregate counts — this is the right answer and you should stop there.

### Strategy 2 — Explicit eviction on write

```java
@CacheEvict(value = "monitors", key = "#monitor.organizationId")
@Transactional
public Monitor update(Monitor monitor) { }
```

**Two problems people miss:**

**a) Eviction happens *before* commit by default.** `@CacheEvict` runs when the method returns, but the transaction commits *after* the proxy chain unwinds. Between those two moments, a concurrent request can miss the cache, read the **old** committed value from the database, and re-populate the cache with stale data — which then persists for the whole TTL.

```java
// ✅ Fix 1: make the cache manager transaction-aware
RedisCacheManager.builder(factory).transactionAware().build();   // defers writes to after commit

// ✅ Fix 2 (better for cross-instance correctness): evict AFTER commit
@TransactionalEventListener(phase = AFTER_COMMIT)
void onMonitorUpdated(MonitorUpdatedEvent e) {
    cacheManager.getCache("monitors").evict(e.orgId());
}
```
**This "evict before commit" race is a genuinely subtle bug and an excellent interview answer** — it explains real-world reports of "I evicted the cache but it still served old data."

**b) You must find every affected key.** Updating one monitor invalidates: that monitor, its org's monitor list, its status pages, its uptime aggregate, the org dashboard. **Miss one and it's stale for the TTL.** This is why TTL-only is often the pragmatic choice — it bounds the damage of every invalidation bug you haven't found.

### Strategy 3 — Cache-aside, written explicitly

```java
public StatusPageResponse render(String slug) {
    var cached = redis.opsForValue().get(key(slug));
    if (cached != null) return deserialize(cached);
    var rendered = doRender(slug);
    redis.opsForValue().set(key(slug), serialize(rendered), Duration.ofSeconds(30));
    return rendered;
}
```
More code, but **full control** — you can implement stampede protection, probabilistic early expiry, and stale-while-revalidate, none of which `@Cacheable` supports.

### The strategies, compared

| Pattern | Behaviour | Notes |
|---|---|---|
| **Cache-aside** (lazy) | App checks cache, loads on miss | ✅ What `@Cacheable` does. The default. |
| **Write-through** | Write to cache and DB together | Cache always warm; writes slower |
| **Write-behind** | Write to cache, flush to DB async | Fast writes, **risk of data loss** |
| **Refresh-ahead** | Proactively refresh before expiry | Avoids miss latency; wastes work on unread keys |

---

## 5. The stampede (thundering herd)

```
A popular status page is cached with a 30s TTL. It expires.
In the next 50ms, 500 concurrent requests all miss.
→ 500 identical expensive queries hit the database simultaneously.
→ The database saturates. Latency spikes. Some requests time out.
→ Those clients retry, adding more load. 💀
```

**This is the failure mode that turns a cache from a performance win into an availability risk**, and it happens precisely when you're busiest.

### Fix 1 — `sync = true` (per-JVM only)

```java
@Cacheable(value = "statusPages", key = "#slug", sync = true)
public StatusPageResponse render(String slug) { }
```
Only one thread computes the value; others **block and wait**. Simple and effective — **but it only synchronizes within one JVM.** With 5 pods you get 5 concurrent computations instead of 500. Usually good enough; say so precisely rather than overselling it.

### Fix 2 — Distributed lock

```java
public StatusPageResponse render(String slug) {
    var cached = get(slug);
    if (cached != null) return cached;

    String lockKey = "lock:statusPage:" + slug;
    boolean acquired = Boolean.TRUE.equals(
            redis.opsForValue().setIfAbsent(lockKey, "1", Duration.ofSeconds(10)));

    if (acquired) {
        try {
            var fresh = doRender(slug);
            put(slug, fresh);
            return fresh;
        } finally { redis.delete(lockKey); }
    }
    // Didn't get the lock: serve stale if we have it, else wait briefly and re-read
    return getStale(slug).orElseGet(() -> waitAndRead(slug));
}
```
**Exactly one computation cluster-wide.** More code; use Redisson's `RLock` rather than hand-rolling in production (Lesson 10 §3).

### Fix 3 — Stale-while-revalidate (the best UX)

Store the value with a **logical** expiry shorter than the physical TTL. On a logically-expired read, **return the stale value immediately** and trigger an async refresh.

```java
record CachedValue<T>(T value, Instant freshUntil) {}

public StatusPageResponse render(String slug) {
    CachedValue<StatusPageResponse> c = get(slug);
    if (c == null) return computeAndCache(slug);                 // genuine miss: must block

    if (c.freshUntil().isBefore(Instant.now())) {
        refreshExecutor.submit(() -> computeAndCache(slug));      // refresh in the background
    }
    return c.value();                                            // ✅ ALWAYS fast
}
```

**No user ever waits for a recomputation** (except the very first). This is what CDNs do, and it's ideal for Vigil's status pages — where "30 seconds stale" is completely acceptable and "slow during an incident" is not. **Note the irony worth pointing out: a status page is most heavily loaded exactly when your system is having problems, so its caching must be its most robust.**

### Fix 4 — Jittered TTLs

If you cache 1,000 keys at startup all with a 10-minute TTL, they all expire **together**, producing a synchronized stampede every 10 minutes.

```java
Duration ttl = Duration.ofMinutes(10).plusSeconds(ThreadLocalRandom.current().nextInt(120));
```
**Add jitter to every bulk-populated cache.** Cheap, and it prevents a whole class of periodic load spike. **A genuinely non-obvious detail that lands well.**

### Cache penetration — the other herd problem

**Repeated requests for keys that don't exist** never populate the cache, so every one hits the database. An attacker can weaponize this by requesting random slugs.

**Fixes:** cache the negative result with a short TTL (note this requires *enabling* null caching, contradicting `disableCachingNullValues()` — a deliberate per-cache choice), or use a bloom filter for existence checks. **Mentioning negative caching unprompted is a nice touch**, because most people only think about the positive path.

---

## 6. When Redis is down

**Your cache must be a performance optimization, never a dependency.**

```java
@Bean
CacheErrorHandler cacheErrorHandler() {
    return new CacheErrorHandler() {
        public void handleCacheGetError(RuntimeException e, Cache cache, Object key) {
            log.warn("cache GET failed for {}:{} — falling through to source", cache.getName(), key);
            // ★ swallow → the method executes normally. Degraded performance, not an outage.
        }
        public void handleCachePutError(RuntimeException e, Cache cache, Object key, Object v) {
            log.warn("cache PUT failed", e);
        }
        public void handleCacheEvictError(RuntimeException e, Cache cache, Object key) {
            log.error("cache EVICT failed — STALE DATA RISK for {}:{}", cache.getName(), key, e);
            // ⚠️ Consider NOT swallowing this one. A failed eviction means serving wrong data.
        }
        public void handleCacheClearError(RuntimeException e, Cache cache) { log.error("...", e); }
    };
}
```

**The asymmetry in that code is the interesting part, and it's a great thing to articulate:**

- **A failed cache GET is safe** — you fall through to the database. Slower, still correct.
- **A failed cache PUT is safe** — you just don't cache it.
- **A failed cache EVICT is dangerous** — the cache now serves data you know is wrong, potentially for the full TTL. Swallowing that error trades correctness for availability, and **that trade should be deliberate**, not a default.

**Also mandatory:** a short Redis command timeout (2s), a circuit breaker so you stop calling a dead Redis (Lesson 34), and a health indicator that reports Redis as degraded rather than failing your readiness probe — **you want to keep serving traffic without a cache, not get taken out of the load balancer.**

```java
@Component
public class RedisCacheHealthIndicator implements HealthIndicator {
    @Override public Health health() {
        try {
            redis.getConnectionFactory().getConnection().ping();
            return Health.up().build();
        } catch (Exception e) {
            return Health.status("DEGRADED")           // ★ not DOWN — we still work
                    .withDetail("error", e.getMessage()).build();
        }
    }
}
```

---

## 7. HTTP caching — the cache you didn't have to build

Before caching server-side, ask whether the client can cache (Lesson 15 §5):

```java
return ResponseEntity.ok()
        .cacheControl(CacheControl.maxAge(Duration.ofSeconds(30)).cachePublic())
        .eTag("\"" + page.checksum() + "\"")
        .body(page);
```

**A 304 Not Modified means zero bytes of body and zero server work** — if you check the ETag *before* doing the expensive computation. For Vigil's status pages, polled by thousands of browsers every 30 seconds, this is the highest-leverage cache in the system, and it costs you nothing to operate.

**And a CDN in front of the status pages** (Cloudflare/Fastly) means most requests never reach your application at all. **"The cheapest cache is the one you don't run"** — for public, cacheable content, CDN + HTTP caching beats Redis on cost, latency, and operational burden. Raising that shows you think about the whole system rather than just your application.

---

## 8. Build this — Vigil's caching

**1. Redis via Docker + Testcontainers:**
```bash
docker run --name vigil-redis -p 6379:6379 -d redis:7-alpine
```

**2. `CacheConfig`** from §3 with per-cache TTLs, JSON serialization, a `vigil:v1:` prefix, `transactionAware()`, and the error handler.

**3. Cache the status page** (`@Cacheable`, 30s TTL) and plan limits (24h TTL). Measure before and after with `curl -w "%{time_total}"` and **write the numbers down.**

**4. Inspect the cache in Redis.**
```bash
docker exec -it vigil-redis redis-cli
> KEYS vigil:v1:*
> GET "vigil:v1:statusPages::acme"
> TTL "vigil:v1:statusPages::acme"
```
**Confirm the value is readable JSON.** Then switch to the default JDK serializer and look at the same key — unreadable binary. **That comparison is why you configure JSON.**

**5. ⭐ Reproduce the multi-tenant key leak.** Write `@Cacheable("monitors")` on a no-argument method that reads the tenant from `TenantContext`. Then:
- Request as org A → populates the cache
- Request as org B → **receive org A's monitors**

**Sit with that.** You have just written a cross-tenant data breach in one annotation. Then fix it with `key = "#orgId"` and confirm isolation. **Then write the test that prevents it forever:**
```java
@Test void cacheKeysAreTenantScoped() {
    var a = service.list(orgA); var b = service.list(orgB);
    assertThat(b).doesNotContainAnyElementsOf(a);
}
```

**6. Prove the self-invocation trap.** A method calling its own `@Cacheable` method. Add cache-hit logging (or watch Redis `MONITOR`) and confirm **zero hits.** Then extract to a second bean and confirm hits appear.

**7. ⭐ Reproduce a stampede, then fix it.** A `@Cacheable` method with `Thread.sleep(2000)` and a 5-second TTL. Hit it with 50 concurrent requests just after expiry (`hey -n 50 -c 50`) and **count the log lines from inside the method — you'll see ~50.** Then add `sync = true` and count again — **1.** Then reason about what happens with three replicas.

**8. Implement stale-while-revalidate** for the status page with a logical `freshUntil` shorter than the TTL. Verify that after logical expiry the response is still fast and a background refresh happens. **This is the version you'd actually ship, and it's a great thing to demo.**

**9. Add TTL jitter** to any bulk-populated cache and verify keys have different TTLs in `redis-cli`.

**10. Prove graceful degradation.**
```bash
docker stop vigil-redis
```
**The application must keep serving requests**, just slower. Verify via the logs (cache warnings) and that `/actuator/health` reports Redis as degraded rather than DOWN. Then `docker start vigil-redis` and confirm caching resumes with no restart. **A demo where you kill Redis live and the app keeps working is memorable in an interview.**

**11. Set a Redis timeout and prove it.** With `timeout: 2s`, use `redis-cli DEBUG SLEEP 10` to hang Redis and confirm your requests fail over to the database within ~2s rather than hanging.

**12. Version the prefix.** Add a field to a cached DTO, bump the prefix to `vigil:v2:`, deploy, and confirm old entries are unreachable and new ones populate cleanly. **That's a zero-downtime cache migration.**

**13. HTTP caching + ETag** on the status page (Lesson 15). Verify a 304 with `If-None-Match` and **confirm the expensive render doesn't run** on the 304 path.

---

## 9. Interview traps

**Q1. When should you NOT cache?**
When staleness is unacceptable (permissions, balances, anything security-relevant); when the data is write-heavy (constant invalidation, poor hit rate); and — most importantly — **before you've measured**, because caching a slow query hides the real bug and converts steady latency into spikes at expiry plus stampede risk. Fix the index or the N+1 first.

**Q2. Why doesn't my `@Cacheable` work?**
In order of likelihood: missing `@EnableCaching`; self-invocation (the proxy is bypassed, so it's silently skipped); a `private`/`final` method; or a key that differs between calls. All silent failures.

**Q3. What's the most dangerous caching bug?**
A cache key that omits the tenant (or user). A method with no parameters reading the tenant from a `ThreadLocal` caches under one key for everyone — so one tenant's data is served to another. It's a data breach, not a performance issue, and it only appears once two tenants use the feature concurrently.

**Q4. Local cache vs distributed cache?**
Local (Caffeine) is per-JVM: fastest, but with N instances you have N independent caches, so an eviction only affects 1/N of traffic and users see stale data on some requests and fresh on others. Distributed (Redis) is one shared cache: slightly slower per access, but evictions are global. For a multi-instance service, Redis — which is also why Hibernate's per-JVM second-level cache is a poor fit there.

**Q5. What's a cache stampede and how do you prevent it?**
When a popular key expires, all concurrent requests miss simultaneously and hammer the source — hardest exactly when you're busiest. Fixes: `sync = true` (per-JVM only — N pods still means N computations), a distributed lock so exactly one instance recomputes, **stale-while-revalidate** (serve stale immediately, refresh asynchronously — no user ever waits), and TTL jitter to stop bulk-loaded keys expiring together.

**Q6. Explain the `@CacheEvict` and transaction race.**
`@CacheEvict` fires when the method returns, but the transaction commits later. In that window a concurrent request misses, reads the **old** committed row, and re-caches it — stale for the full TTL. Fix with a `transactionAware()` cache manager or by evicting from a `@TransactionalEventListener(AFTER_COMMIT)`.

**Q7. What happens to your application when Redis goes down?**
It must degrade, not fail. Configure a short command timeout, a `CacheErrorHandler` that swallows GET/PUT failures and falls through to the source, a circuit breaker so you stop calling a dead Redis, and a health indicator reporting **DEGRADED** rather than DOWN so you aren't removed from the load balancer. **Note the asymmetry: a failed EVICT is different — swallowing it means knowingly serving stale data, so that trade must be deliberate.**

**Q8. Why not JDK serialization for cache values?**
Unreadable in `redis-cli` (so undebuggable), a known deserialization-attack surface, and brittle — any class change breaks every existing entry with `InvalidClassException`. Use JSON, and version the cache key prefix on breaking DTO changes.

**Q9. How do you deploy a change to a cached DTO's shape?**
Bump a version segment in the cache key prefix (`vigil:v1:` → `vigil:v2:`), which makes all old entries unreachable without touching Redis — a safe, instant, zero-downtime flush that doesn't affect other users of the same Redis instance (unlike `FLUSHDB`).

**Q10. `condition` vs `unless`?**
`condition` is evaluated before the method, on the inputs, so it can prevent both lookup and store. `unless` is evaluated after, on `#result`, so it can skip caching based on the outcome (e.g. `unless = "#result.isEmpty()"`).

**Q11. What's cache penetration?**
Repeated requests for non-existent keys, which never populate the cache and so always hit the database — weaponizable by requesting random IDs. Fix by caching negative results with a short TTL, or a bloom filter for existence.

**Q12. What's the cheapest cache?**
The one you don't run: HTTP caching (`Cache-Control` + ETag → 304 with no body and no server work) and a CDN in front of public content, so most requests never reach your application. For Vigil's status pages that beats Redis on latency, cost, and operational burden.

---

## 10. Retention check

1. Give the three questions to ask before adding `@Cacheable`.
2. Explain the multi-tenant cache key bug and why it's a breach rather than a bug.
3. Why is `sync = true` only a partial stampede fix?
4. Describe the `@CacheEvict`/commit race and both fixes.
5. Why is a failed cache *evict* more dangerous than a failed cache *get*?
6. How do you flush a cache safely during a deploy without `FLUSHDB`?

---

**Next:** [Lesson 34 — Resilience & outbound integration](34-resilience-and-integration.md) — timeouts, retries, circuit breakers, and idempotency.

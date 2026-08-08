# Lesson 35 — Observability

> **Why this lesson exists:** The gap between "I built a feature" and "I run a service" is observability. It's also the topic where Vigil has a delightful recursive property: it's a monitoring tool, so it must be observable, and you'll instrument it with the same techniques it provides to its users. That's a genuinely great thing to talk about in an interview.

**Time:** ~70 minutes · **Prereq:** Module 9 so far

---

## 1. Monitoring vs observability

| | Monitoring | Observability |
|---|---|---|
| Answers | *"Is the thing I predicted happening?"* | *"Why is this happening?"* — including things you didn't predict |
| Built from | Dashboards, alerts on known metrics | High-cardinality data you can query arbitrarily |
| Fails when | The failure is novel | (Rarely — that's the point) |

**The distinction that matters:** monitoring answers questions you thought of in advance. Observability lets you ask new questions *during* an incident, without deploying code. **"Why are requests from this one customer's region slow on Tuesdays?" is not a dashboard you built — it's a query you need to be able to run.**

### The three pillars, and what each is for

```
METRICS       aggregated numbers over time — cheap, low cardinality
              "p99 latency is 2s"                → tells you SOMETHING is wrong

LOGS          discrete timestamped events — expensive, high cardinality
              "monitor 7f3a failed: connection refused"  → tells you WHAT happened

TRACES        one request's path across services — moderate cost
              "the 2s was 1.9s waiting on Slack"  → tells you WHERE the time went
```

**The workflow: metric alerts you → trace localizes it → logs explain it.** Each pillar is nearly useless alone. A metric spike with no traces means you know something's wrong and nothing else. Logs without correlation IDs mean you can't reconstruct a single request. **Being able to describe that flow is the answer to "how do you debug a production issue?"**

---

## 2. Actuator

```xml
<dependency><groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId></dependency>
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,loggers    # ★ allowlist (Lesson 28)
  endpoint:
    health:
      show-details: when-authorized
      probes:
        enabled: true                                       # ★ liveness + readiness
      group:
        liveness:
          include: ping
        readiness:
          include: db, redis                                # ★ see §2.1 — choose carefully
  server:
    port: 9090                                              # ★ separate port
  metrics:
    tags:
      application: ${spring.application.name}
      environment: ${ENVIRONMENT:local}
      instance: ${HOSTNAME:localhost}
  info:
    env: { enabled: true }
    git: { mode: full }
    build: { enabled: true }
```

### 2.1 Liveness vs readiness — get this wrong and you'll cause an outage

| Probe | Question | Failure action |
|---|---|---|
| **Liveness** | *"Is this process broken beyond recovery?"* | **Kubernetes KILLS and restarts the pod** |
| **Readiness** | *"Can this instance serve traffic right now?"* | Removed from the load balancer; **not** killed |

**The catastrophic mistake:**
```yaml
management.endpoint.health.group.liveness.include: db     # ❌❌ NEVER
```

**Why this is a disaster:** the database has a brief hiccup. Every pod's liveness probe fails. **Kubernetes kills every pod simultaneously.** They restart, all try to connect at once, overwhelm the recovering database, fail liveness again, and get killed again — **a restart loop that turns a 30-second blip into a total outage.**

**The rules:**
- **Liveness = `ping` only.** Is the JVM alive and the event loop responsive? A restart must be capable of fixing whatever liveness reports.
- **Readiness = dependencies you cannot serve without.** A failed readiness probe removes you from the load balancer temporarily, which is exactly right during a dependency blip. **And note: if every pod is unready, traffic goes nowhere — so put only *essential* dependencies here.**
- **Redis is usually NOT a readiness dependency** (Lesson 33): you can serve without a cache, just slower. Reporting DOWN would take you out of rotation unnecessarily.

**This is one of the highest-value operational facts in the course**, and "I've seen a liveness probe that checked the database take down a whole cluster" is a story that lands.

```yaml
# Kubernetes side
livenessProbe:
  httpGet: { path: /actuator/health/liveness, port: 9090 }
  initialDelaySeconds: 30
  failureThreshold: 3
readinessProbe:
  httpGet: { path: /actuator/health/readiness, port: 9090 }
  periodSeconds: 5
  failureThreshold: 2
startupProbe:                          # ★ for slow JVM startup — protects liveness during boot
  httpGet: { path: /actuator/health/liveness, port: 9090 }
  failureThreshold: 30
  periodSeconds: 5
```

**`startupProbe` is the one people miss:** without it, `initialDelaySeconds` must cover worst-case startup, which delays detection of genuine hangs. A startup probe disables liveness until the app is up, so you can then use an aggressive liveness period.

### Custom health indicators

```java
@Component
@RequiredArgsConstructor
public class CheckWorkerHealthIndicator implements HealthIndicator {

    private final CheckMetrics metrics;

    @Override
    public Health health() {
        Duration sinceLastCheck = Duration.between(metrics.lastDispatchAt(), Instant.now());
        long backlog = metrics.overdueMonitorCount();

        if (sinceLastCheck.toMinutes() > 5) {
            return Health.down()
                    .withDetail("reason", "no checks dispatched in " + sinceLastCheck)
                    .build();
        }
        if (backlog > 1000) {
            return Health.status("DEGRADED").withDetail("backlog", backlog).build();
        }
        return Health.up().withDetail("backlog", backlog)
                .withDetail("lastDispatch", metrics.lastDispatchAt()).build();
    }
}
```

**This is genuinely useful for Vigil:** the API can be perfectly healthy while the check engine has silently died (a scheduled task that threw and was cancelled — Lesson 32 §2.1). **A health check that only tests HTTP responsiveness would report everything green while the product does nothing.** Health checks should reflect *whether the service is doing its job*, not merely whether it responds.

---

## 3. Metrics with Micrometer

**Micrometer is a facade** (SLF4J for metrics): you code against one API and swap Prometheus, Datadog, CloudWatch, or New Relic by changing a dependency.

```xml
<dependency><groupId>io.micrometer</groupId>
  <artifactId>micrometer-registry-prometheus</artifactId></dependency>
```

### The four meter types

```java
@Service
@RequiredArgsConstructor
public class CheckMetricsRecorder {

    private final MeterRegistry registry;

    // 1. COUNTER — monotonically increasing. "How many?"
    public void recordCheck(MonitorType type, CheckStatus status) {
        registry.counter("vigil.checks.executed",
                "type", type.name(),
                "status", status.name()).increment();
    }

    // 2. TIMER — duration + count + percentiles. "How long?"
    public <T> T timeCheck(MonitorType type, Supplier<T> check) {
        return registry.timer("vigil.check.duration", "type", type.name())
                .record(check);
    }

    // 3. GAUGE — a value that goes up and down, SAMPLED. "How many right now?"
    @PostConstruct
    void registerGauges() {
        Gauge.builder("vigil.monitors.active", monitorRepository, MonitorRepository::countActive)
                .description("monitors currently being checked")
                .register(registry);
        // ★ Micrometer holds a WEAK reference — if the object is GC'd, the gauge reports NaN.
        //   Keep a strong reference (as a field) or use a supplier that can't be collected.
    }

    // 4. DISTRIBUTION SUMMARY — distribution of a non-time value
    public void recordResponseSize(int bytes) {
        registry.summary("vigil.check.response.size").record(bytes);
    }
}
```

| Type | Use | Never use for |
|---|---|---|
| **Counter** | requests, errors, events | anything that can decrease |
| **Timer** | latency, durations | non-time values |
| **Gauge** | queue depth, pool size, active connections | **counting events** (you'd lose data between samples) |
| **Distribution summary** | payload sizes, batch sizes | durations (use a Timer) |

### Percentiles — and why averages lie

```yaml
management:
  metrics:
    distribution:
      percentiles-histogram:
        http.server.requests: true       # ★ server-side histograms → aggregatable percentiles
      percentiles:
        http.server.requests: 0.5, 0.95, 0.99
      slo:
        http.server.requests: 100ms, 300ms, 500ms, 1s
```

**An average is nearly useless for latency.** 99 requests at 10ms and one at 10s averages 110ms — invisible on a dashboard, and that one request is a customer having a terrible time. **p99 is what your unhappiest 1% experience**, and at scale that's a lot of people.

**The critical technical point: you cannot average percentiles across instances.** If pod A's p99 is 200ms and pod B's is 400ms, the fleet p99 is not 300ms. **That's why `percentiles-histogram: true` matters** — it exports histogram *buckets*, which Prometheus can aggregate correctly with `histogram_quantile()`. Client-side `percentiles` are per-instance only.

**Knowing that percentiles aren't averageable, and that histogram buckets are the fix, is a genuinely differentiating piece of knowledge.** Most people configure `percentiles` and wonder why their fleet-wide dashboard is wrong.

### ⚠️ Cardinality — the mistake that costs real money

```java
// ❌❌ Each unique tag combination creates a new time series.
registry.counter("vigil.checks", "monitorId", monitorId.toString()).increment();
// 50,000 monitors × 3 statuses = 150,000 time series from ONE metric.
// This will destabilize Prometheus and generate an enormous bill on a SaaS vendor.
```

**Tags must be low-cardinality and bounded.** ✅ `type` (4 values), `status` (3), `region` (5). ❌ user ID, monitor ID, URL, email, request ID, error message, timestamp.

**The rule: if you can't enumerate the possible values, it's not a tag.**

**Where does high-cardinality data go?** **Logs and traces** — that's exactly what they're for. `monitorId` belongs in a structured log field where you can query it, not in a metric label.

**Metric naming convention** (Micrometer normalizes per backend):
```
vigil.checks.executed          dot-separated, lowercase   → vigil_checks_executed_total (Prometheus)
```
Counters get `_total`; timers get `_seconds`. Follow the convention or your dashboards look foreign.

### What Boot gives you free

`http.server.requests` (with `uri`, `method`, `status`, `outcome` tags), `jvm.memory.*`, `jvm.gc.*`, `jvm.threads.*`, `hikaricp.connections.*`, `spring.data.repository.invocations`, `cache.gets`/`cache.puts`, `resilience4j.*`, `executor.*`, `tomcat.*`, `process.cpu.usage`, `application.started.time`.

**`http.server.requests`'s `uri` tag is templated** (`/api/v1/monitors/{id}`, not the actual UUID) — which is precisely the cardinality discipline above, applied by the framework. If you ever see raw IDs in that tag, something is wrong.

### The metrics Vigil actually needs

```java
vigil.checks.executed{type, status}          counter    — throughput and failure rate
vigil.check.duration{type}                    timer      — how long checks take
vigil.checks.overdue                          gauge      — ★ THE key health signal
vigil.incidents.open{severity}                gauge      — current incident count
vigil.notifications.sent{channel, outcome}    counter    — delivery success rate
vigil.notification.delivery.duration{channel} timer
vigil.outbox.pending                          gauge      — ★ growing = the publisher is stuck
vigil.dead_letters.total{reason}              counter    — ★ any increase deserves attention
```

**`vigil.checks.overdue` is the metric that matters most**, and it's worth explaining why: it directly measures whether the product is doing its job. Every other metric can look fine while checks silently stop (a cancelled scheduled task, an exhausted executor, a stuck claim). **A metric that measures the business outcome rather than a technical proxy is the mark of good instrumentation** — and identifying it for your own system is exactly what an interviewer wants to see.

---

## 4. Structured logging

### Why structured

```
# ❌ Unstructured — a human can read it; a machine can't query it
2026-07-30 14:32:11 INFO  Check failed for monitor api-health: connection refused after 5023ms

# ✅ Structured — queryable: level=ERROR AND monitorType=HTTP AND durationMs>5000
{"timestamp":"2026-07-30T14:32:11.123Z","level":"ERROR","logger":"c.v.check.HttpCheckExecutor",
 "message":"check failed","correlationId":"8f3a2b1c","orgId":"a1b2","monitorId":"7f3a",
 "monitorType":"HTTP","error":"connection refused","durationMs":5023,"thread":"check-3"}
```

**With unstructured logs you `grep`. With structured logs you query, aggregate, and alert.** "Show me the p95 check duration for HTTP monitors in org X over the last hour" is a query against structured logs and an impossibility against text.

```xml
<!-- Boot 3.4+ has built-in structured logging — no extra dependency -->
```
```yaml
logging:
  structured:
    format:
      console: ecs               # or logstash, gelf
  pattern:
    correlation: "[${spring.application.name},%X{traceId:-},%X{spanId:-}] "
  level:
    root: INFO
    com.vigil: DEBUG
```

**Boot 3.4 added first-class structured logging**, which previously needed `logstash-logback-encoder`. Worth knowing both — you'll see the manual encoder in existing projects.

### MDC — the mechanism that makes logs correlatable

```java
MDC.put("correlationId", id);
MDC.put("orgId", orgId.toString());
try {
    chain.doFilter(req, res);
} finally {
    MDC.clear();          // ★ MANDATORY — pooled thread reuse (Lesson 14)
}
```

**Every log line in that request now carries the correlation ID automatically**, without threading it through every method signature. This is what turns thousands of interleaved log lines from concurrent requests into reconstructible per-request narratives.

**And the corollary you now know well:** MDC is `ThreadLocal`, so it doesn't cross `@Async`, `@Scheduled`, or reactive boundaries — hence the `TaskDecorator` (Lesson 32 §1.3). **Fifth appearance of this lesson; by now it should be automatic.**

### Log hygiene

```java
// ✅ Parameterized — no string building unless the level is enabled
log.debug("check completed for monitor={} in {}ms", monitorId, duration);

// ❌ Concatenation — builds the string even when DEBUG is off
log.debug("check completed for " + monitorId + " in " + duration + "ms");

// ✅ Exceptions: message with context, then the throwable LAST
log.error("check failed monitor={} url={}", monitorId, url, e);

// ❌ Loses the stack trace entirely
log.error("check failed: " + e.getMessage());
```

**Log levels, used consistently:**

| Level | Meaning |
|---|---|
| `ERROR` | **A human must act.** Every ERROR should be actionable, or people learn to ignore them. |
| `WARN` | Unexpected but handled — a retry succeeded, a fallback engaged |
| `INFO` | Significant business events — monitor created, incident opened. Low volume. |
| `DEBUG` | Developer detail, off in production |
| `TRACE` | Very verbose, temporary |

**The most valuable logging discipline: ERROR must mean "someone needs to do something."** A codebase that logs ERROR for expected conditions (a monitor being down is *normal* for Vigil!) trains its operators to ignore errors — and then a real one is missed. **In Vigil specifically, a failed check is `INFO`; a failure to *record* a check is `ERROR`.** That distinction is worth articulating.

**Never log:** passwords, tokens, API keys, full request bodies, PII beyond identifiers, credit-card data. Redact in filters and aspects (Lessons 10, 17). **And note that logs typically have wider access than the database** — a leak into your log aggregator can be worse than a database exposure.

**Runtime log level changes** — genuinely useful during an incident:
```bash
curl -X POST localhost:9090/actuator/loggers/com.vigil.check \
  -H 'Content-Type: application/json' -d '{"configuredLevel":"DEBUG"}'
```
Debug a production problem without a deploy. **Remember it's a *writable* endpoint, so it must be authenticated** (Lesson 28).

---

## 5. Distributed tracing

```xml
<dependency><groupId>io.micrometer</groupId><artifactId>micrometer-tracing-bridge-otel</artifactId></dependency>
<dependency><groupId>io.opentelemetry</groupId><artifactId>opentelemetry-exporter-otlp</artifactId></dependency>
```
```yaml
management:
  tracing:
    sampling:
      probability: 0.1              # ★ 10% in production; 1.0 locally
  otlp:
    tracing:
      endpoint: http://otel-collector:4318/v1/traces
```

**Concepts:**
```
TRACE     one end-to-end operation, identified by a traceId
 └─ SPAN  one unit of work within it, with a spanId and a parentId

Trace: POST /api/v1/monitors                            250ms
├─ span: MonitorController.create                       248ms
│  ├─ span: validate                                      2ms
│  ├─ span: INSERT monitor                               15ms
│  ├─ span: HTTP POST hooks.slack.com                   180ms   ← ★ found it
│  └─ span: INSERT outbox_message                          8ms
```

**Context propagation** happens via W3C `traceparent` headers, injected automatically by instrumented clients. So a trace follows a request across service boundaries — which is the entire point when Vigil splits into api/checker/notifier (Module 10).

**Custom spans:**
```java
@Observed(name = "check.execute", contextualName = "execute-check")     // ★ the modern way
public CheckOutcome execute(Monitor monitor) { }
```

**`@Observed` is worth highlighting:** one annotation produces a **timer metric, a trace span, and log correlation** from a single instrumentation point. Micrometer's Observation API unified what used to be three separate concerns. **This is the answer to "how do you instrument a method?" in modern Spring** — and mentioning it instead of hand-rolling an aspect (Lesson 10) shows currency.

### Sampling — the trade-off to state

100% tracing on a high-traffic service is expensive in storage and network. But sampling means **the trace for the one request you care about may not exist.**

Approaches, and knowing them is the differentiator:
- **Head-based** (Spring's default): decide at the start, e.g. 10%. Simple, cheap, may miss the interesting request.
- **Tail-based** (collector-side): buffer all spans, then keep traces that are slow or errored. **Much better** — you keep the ones that matter — but requires a collector that supports it (the OpenTelemetry Collector does).
- **Always sample errors:** a common hybrid — 1% of successes, 100% of errors and slow requests.

**"I'd use tail-based sampling at the collector so slow and failed traces are always retained" is a strong, specific answer** to a question most candidates answer with "we sample 10%."

### Logs ↔ traces ↔ metrics correlation

The `traceId` in MDC (via the `logging.pattern.correlation` above) is what ties the pillars together:
```
Metric alert: p99 > 1s
  → find a slow trace in that window
  → copy its traceId
  → query logs for traceId=abc123
  → see exactly what happened, with all the high-cardinality detail
```
**That workflow is the whole point of observability, and being able to narrate it is the answer to "walk me through debugging a latency regression."**

---

## 6. SLOs and alerting

### Define what "working" means

```
SLI (indicator):  the proportion of requests served successfully in under 300ms
SLO (objective):  99.9% over a rolling 30 days
Error budget:     0.1% → ~43 minutes of failure per month
```

**Why an error budget is a genuinely useful concept:** it converts reliability from an argument into arithmetic. Budget remaining → ship features. Budget exhausted → stop feature work and fix reliability. **It also makes "100% uptime" visibly the wrong target** — it would mean never deploying.

### Alert on symptoms, not causes

```
❌ CPU > 80%                        — so what? It might be fine.
❌ A pod restarted                   — Kubernetes did its job.
❌ A single request failed           — noise.

✅ Error rate > 1% for 5 minutes                        users are affected
✅ p99 latency > 1s for 10 minutes                      users are affected
✅ vigil.checks.overdue > 1000 for 5 minutes            ★ the product isn't working
✅ vigil.outbox.pending growing for 15 minutes           notifications aren't being delivered
✅ Error budget burn rate > 10× normal                   we'll blow the SLO soon
```

**Alert on user-visible symptoms.** Causes belong on dashboards you consult *after* an alert. High CPU that harms nobody should not wake anyone.

**The two rules of alerting that matter:**

**1. Every alert must be actionable.** If the response is "acknowledge and ignore," delete the alert. **Alert fatigue is the actual failure mode** — a team that receives 50 alerts a night stops reading them, and then the real one is missed. Fewer, better alerts beat comprehensive coverage.

**2. Multi-window burn-rate alerts** are the modern practice: a fast burn (2% of monthly budget in 1 hour) pages immediately; a slow burn (10% in 3 days) creates a ticket. This catches both sudden outages and gradual degradation without the false positives of a simple threshold.

**Vigil's alerting has the same recursion:** it alerts *customers* about *their* services, so its own alerting must be independent. **Vigil cannot be the thing that monitors Vigil** — you need an external check (a third-party uptime service, or a dead-man's-switch/heartbeat like Healthchecks.io that alerts when Vigil *stops* reporting). **That's a genuinely interesting design point and a great thing to raise unprompted**: it's the same reason you don't store your backups on the server you're backing up.

---

## 7. The stack

```
Vigil (Micrometer + OTel)
   │
   ├── /actuator/prometheus ──scraped by── Prometheus ──▶ Grafana (dashboards)
   │                                          └────────▶ Alertmanager ──▶ PagerDuty/Slack
   │
   ├── structured JSON logs ──▶ Promtail/Fluent Bit ──▶ Loki ──▶ Grafana
   │
   └── OTLP spans ──▶ OpenTelemetry Collector ──▶ Tempo/Jaeger ──▶ Grafana
```

**Grafana + Prometheus + Loki + Tempo** ("the LGTM stack") is the standard open-source combination and runs locally in Docker Compose. Commercial alternatives (Datadog, New Relic, Honeycomb) trade cost for less operational work.

```yaml
# docker-compose.observability.yml — a real local stack
services:
  prometheus:
    image: prom/prometheus
    volumes: [ ./prometheus.yml:/etc/prometheus/prometheus.yml ]
    ports: [ "9091:9090" ]
  grafana:
    image: grafana/grafana
    ports: [ "3000:3000" ]
    environment: { GF_AUTH_ANONYMOUS_ENABLED: "true", GF_AUTH_ANONYMOUS_ORG_ROLE: Admin }
  tempo:
    image: grafana/tempo
    command: [ "-config.file=/etc/tempo.yaml" ]
  loki:
    image: grafana/loki
```

**Because Micrometer and OpenTelemetry are facades, this is a configuration choice, not an architectural one** — you can move to Datadog by swapping a dependency. That's a good point to make.

---

## 8. Build this — make Vigil observable

**1. Actuator with probes** and a separate management port. Hit `/actuator/health/liveness` and `/actuator/health/readiness` and note they return different things.

**2. ⭐ The liveness mistake, demonstrated.** Put `db` in the **liveness** group. Stop Postgres. **Confirm liveness reports DOWN.** Then reason through what Kubernetes would do: kill every pod, restart, all reconnect at once, fail again. **Write down the incident narrative.** Then fix it — liveness = `ping`, readiness = `db`. **This is the most important operational lesson in the module.**

**3. Custom health indicators:** `CheckWorkerHealthIndicator` (backlog and last-dispatch age) and a Redis indicator reporting **DEGRADED** rather than DOWN (Lesson 33 §6). Confirm that stopping Redis does **not** make you unready.

**4. Prometheus metrics.** Add the registry, hit `/actuator/prometheus`, and read the raw output. Find `http_server_requests_seconds_bucket` and note the templated `uri` label.

**5. Vigil's business metrics** from §3: counters for checks and notifications, a timer for check duration, and gauges for `checks.overdue`, `incidents.open`, and `outbox.pending`.

**6. ⭐ Cause a cardinality explosion.** Add `monitorId` as a metric tag. Seed 1,000 monitors, run checks, then hit `/actuator/prometheus` and **count the lines.** Watch the endpoint slow down and the response balloon. **Then remove it and note where that data belongs instead** — a structured log field. This experiment makes the cardinality rule visceral rather than theoretical.

**7. Structured logging** with `logging.structured.format.console: ecs`. Confirm log lines are JSON with `correlationId` and `traceId`. Then make a request and **grep your logs by correlation ID** to reconstruct the full request narrative.

**8. Prove MDC propagation.** Log inside an `@Async` method. Without the `TaskDecorator`, `correlationId` is absent. Add it, and it appears. **Sixth and final appearance of the ThreadLocal lesson.**

**9. Tracing.** Add the OTel bridge and run Tempo (or Jaeger) locally. Trigger a request that hits the database *and* WireMock. **Open the trace UI and look at the waterfall.** Find the span that dominates. **Seeing your own request decomposed by time is the moment tracing clicks.**

**10. `@Observed`.** Annotate `CheckExecutor.execute`. Confirm you get **both** a `check_execute_seconds` metric **and** a trace span from one annotation.

**11. The full local stack.** Docker Compose with Prometheus, Grafana, Loki, and Tempo. Build a Grafana dashboard with: request rate, error rate, p50/p95/p99 latency, `checks.overdue`, open incidents, Hikari pool usage, and circuit-breaker state. **Screenshot it for your README** — a real dashboard is one of the most credible things a portfolio project can show.

**12. An alert rule.** In Prometheus:
```yaml
- alert: ChecksBacklogGrowing
  expr: vigil_checks_overdue > 1000
  for: 5m
  labels: { severity: page }
  annotations:
    summary: "Check backlog is {{ $value }} — monitors are not being checked on schedule"
```
**Then trigger it:** stop the dispatcher and watch the alert fire in Alertmanager. **An alert you've actually seen fire is worth ten you've configured.**

**13. Practise the debugging workflow.** Deliberately introduce an N+1 or a slow dependency. Then, using only your observability stack: notice the metric, find the trace, identify the slow span, query the logs by trace ID, and diagnose it. **Time yourself.** That rehearsal is exactly what an interviewer is probing when they ask "how would you debug this?"

**14. External monitoring for Vigil itself.** Add a heartbeat: after each successful dispatch cycle, ping an external dead-man's-switch (Healthchecks.io has a free tier). **Then stop Vigil and confirm you get an email.** **You have now solved the "who watches the watcher" problem** — and that's a genuinely memorable thing to describe.

---

## 9. Interview traps

**Q1. Monitoring vs observability?**
Monitoring answers predefined questions about known failure modes. Observability means having enough high-cardinality data to ask *new* questions during an incident without deploying code. The three pillars work together: metrics tell you something is wrong, traces tell you where, logs tell you what.

**Q2. Liveness vs readiness, and what's the classic mistake?**
Liveness = "is this process irrecoverably broken?" → Kubernetes **kills** the pod. Readiness = "can it serve traffic now?" → removed from the load balancer. **The mistake is checking the database in liveness:** a brief DB blip fails every pod's liveness, so Kubernetes kills them all, they restart and overwhelm the recovering database, and a 30-second blip becomes a restart-loop outage. Liveness should be `ping`; readiness should list only dependencies you genuinely cannot serve without.

**Q3. Should Redis be a readiness dependency?**
Usually no. You can serve without a cache, just slower, so reporting DOWN would remove you from rotation unnecessarily. Report DEGRADED in a custom indicator and keep serving.

**Q4. What is high cardinality and why does it matter for metrics?**
Each unique tag combination is a separate time series. A user ID or monitor ID as a tag creates thousands of series from one metric, destabilizing Prometheus and generating large SaaS bills. **Rule: if you can't enumerate the possible values, it isn't a tag** — put it in logs or traces, which are built for high cardinality.

**Q5. Why are averages useless for latency, and what's the trap with percentiles?**
An average hides the tail: 99 requests at 10ms and one at 10s averages 110ms while a customer waits ten seconds. Use p95/p99. **The trap: percentiles cannot be averaged across instances.** Export histogram buckets (`percentiles-histogram: true`) so Prometheus can compute fleet-wide quantiles with `histogram_quantile()`.

**Q6. Why structured logging?**
Text logs can only be grepped; JSON logs can be queried, aggregated and alerted on — "p95 check duration for HTTP monitors in org X over the last hour" is a query, not a grep. Boot 3.4+ has built-in structured logging (`logging.structured.format`).

**Q7. What is MDC and what's its limitation?**
A per-thread map of context (correlation ID, tenant, trace ID) automatically included in every log line, so concurrent requests remain reconstructible. It's `ThreadLocal`, so it doesn't cross `@Async`/`@Scheduled`/reactive boundaries — you need a `TaskDecorator` (or Micrometer context propagation), and you **must** clear it in a `finally` because pooled threads are reused.

**Q8. What should `ERROR` mean?**
"A human must act." If ERROR is used for expected conditions — and for Vigil a failed check is completely normal — operators learn to ignore errors and miss real ones. A failed check is INFO; a failure to *record* one is ERROR.

**Q9. Explain traces, spans, and context propagation.**
A trace is one end-to-end operation with a `traceId`; spans are units of work within it with parent/child relationships. Context propagates across services via W3C `traceparent` headers, injected automatically by instrumented clients — which is what lets you follow one request through multiple services and see where its time went.

**Q10. How do you handle trace sampling?**
Head-based sampling (Spring's default) decides up front, e.g. 10% — cheap but may discard the request you care about. **Tail-based sampling** at the collector buffers spans and retains traces that are slow or errored, so the interesting ones are always kept. A common hybrid: 1% of successes, 100% of errors.

**Q11. What are SLIs, SLOs and error budgets?**
SLI = the measurement (proportion of requests under 300ms). SLO = the target (99.9% over 30 days). Error budget = the allowed shortfall (~43 min/month), which turns reliability into arithmetic: budget remaining → ship features; budget exhausted → fix reliability. It also shows why 100% is the wrong target — it would mean never deploying.

**Q12. What should you alert on?**
User-visible symptoms — error rate, latency, and business signals like `checks.overdue` — not causes like CPU or pod restarts. Every alert must be actionable, because alert fatigue is the real failure mode. Modern practice is multi-window error-budget burn-rate alerts: fast burn pages, slow burn files a ticket.

**Q13. What does `@Observed` give you?**
A timer metric, a trace span, and log correlation from a single annotation, via Micrometer's Observation API — which unified what used to be three separate instrumentation concerns. It's the modern replacement for hand-rolled timing aspects.

**Q14. How do you monitor a monitoring system?**
Externally. Vigil cannot be the thing that watches Vigil — a shared failure takes out both the service and its alerting. Use a third-party check or a dead-man's-switch heartbeat that alerts when Vigil *stops* reporting. Same principle as not storing backups on the machine you're backing up.

---

## 10. Retention check

1. Explain the liveness-checks-the-database failure, step by step, to its conclusion.
2. Why can't you average p99 across instances, and what's the fix?
3. Give the rule for what may be a metric tag, and where high-cardinality data belongs.
4. What should ERROR mean, and give Vigil's specific example of the distinction.
5. Narrate the metric → trace → log debugging workflow.
6. Why can't Vigil monitor itself, and what do you do instead?

---

**Next:** [Lesson 36 — Performance & JVM tuning](36-performance-and-jvm-tuning.md) — startup time, memory, GC, and diagnosing latency with real tools.

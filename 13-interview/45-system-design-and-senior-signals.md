# Lesson 45 — System design & senior signals

> **Why this lesson exists:** this is the final lesson. Everything so far has been knowledge. This is about *how you use it under pressure* — in a design round, in a code review, in a behavioural question. The difference between a mid-level and a senior candidate is almost never knowledge; it's **judgment, trade-off awareness, and knowing what you don't know.** Those are demonstrable, and this lesson shows you how.

**Time:** ~65 minutes

---

## 1. The idea in one sentence

> **Seniority in an interview is almost never demonstrated by knowing more — it's demonstrated by naming the trade-off, saying plainly what you don't know, and choosing the simpler option for a stated reason.**

Everything that follows is a way of making that visible under time pressure: a design framework that forces you to clarify before you solve, phrases that signal calibration rather than confidence, and the discipline of saying "it depends" and then immediately saying what it depends on.

---

## 2. What "senior" actually means in an interview

Most candidates think they're being assessed on knowledge. They're not, or not primarily. Here's what's actually being scored:

| Signal | Junior | Mid | **Senior** |
|---|---|---|---|
| **Answers** | Recites | Explains | **Explains, then gives the trade-off** |
| **Certainty** | Confident about everything | Confident about what they know | **Explicit about uncertainty and how they'd resolve it** |
| **Solutions** | The one they know | The right one | **The simplest one that works, with the reason it's enough** |
| **Technology** | Uses what's fashionable | Uses what's appropriate | **Can argue against their own choice** |
| **Problems** | Fixes symptoms | Fixes causes | **Prevents recurrence structurally** |
| **Scope** | Builds what's asked | Asks clarifying questions | **Challenges the requirement when it's wrong** |
| **Failure** | Doesn't consider it | Handles it | **Assumes it, designs for degradation** |
| **Own work** | Defends it | Explains it | **Critiques it unprompted** |

**The single strongest signal on that table is the last row.** A candidate who volunteers "here's what I'd do differently" without being asked has demonstrated something no amount of correct answers can: that they evaluate their own work the way an experienced engineer does.

**The second strongest is the certainty row.** "I don't know, but here's how I'd find out" is a *better* answer than a confident guess, and interviewers know that guessing is what causes production incidents.

---

## 3. The habits that make you sound senior

These are mechanical. You can adopt them today, and they change how every answer lands.

### Habit 1 — Always give the trade-off

```
Q: "Should we cache this?"

Mid:    "Yes, use @Cacheable with Redis."
Senior: "Probably — but first: how stale can it be, and what's the read:write ratio?
         If it's read-heavy and 30 seconds of staleness is fine, Redis with a TTL is
         easy. If it's write-heavy, invalidation will cost more than the cache saves.
         And I'd want to check the query first — caching a slow query hides the bug
         and turns steady latency into spikes at expiry."
```

**The pattern: answer, then name the cost, then name the condition under which you'd choose differently.** Every technical decision has a cost. Naming it proves you've made the decision rather than absorbed it.

### Habit 2 — Say "it depends," then say on what

"It depends" alone is a non-answer and interviewers hate it. **"It depends on X, and here's my answer for each value of X"** is a senior answer.

```
Q: "Kafka or RabbitMQ?"

Senior: "It depends on whether I need replay and multiple independent consumers.
         Kafka if I do — it's a retained log, so a new consumer can process history.
         RabbitMQ if I need rich routing, priorities, or delays, and it's meaningfully
         simpler to operate. And honestly, 'we need Kafka' is often 'we need a queue' —
         running Kafka is a real ongoing cost that a 100-message-per-second workload
         doesn't justify."
```

### Habit 3 — Volunteer the failure mode

```
Q: "How would you handle notifications?"

Senior: "@Async off the request thread, with a bounded executor — the default is
         unbounded, which is a memory hazard. But the interesting question is what
         happens when Slack is down: retries with jittered backoff, a circuit breaker
         so I stop hammering it, and a dead-letter table so failures aren't silently
         lost. And I'd publish from a transactional outbox rather than inside the
         transaction, so a notification can't be sent for work that rolled back."
```

**Unprompted failure analysis is one of the most reliable senior markers**, because juniors design the happy path and seniors design the recovery.

### Habit 4 — Reach for the simplest thing that works

```
Q: "How would you distribute work across instances?"

Junior: "Kafka with a consumer group."
Senior: "If the work is already in Postgres, `SELECT ... FOR UPDATE SKIP LOCKED`
         gives me disjoint batches per instance with no new infrastructure. I'd only
         add a broker when I need fan-out to independent consumers, replay, or
         cross-service decoupling — because a broker is a stateful system to run,
         monitor and upgrade."
```

**Proposing less is harder than proposing more, and interviewers notice.** Over-engineering is the most common failure of technically strong candidates.

### Habit 5 — Bound your uncertainty explicitly

```
Q: "How does Hibernate's second-level cache handle invalidation across a cluster?"

Bad:    [confident guess]
Senior: "I know the second-level cache is per-JVM by default, which is why I avoid
         it in multi-instance deployments — you get N caches that diverge. I know
         distributed providers like Infinispan exist to solve that, but I haven't
         operated one, so I couldn't tell you their invalidation semantics in detail.
         I'd read the provider docs and test it with two instances before relying on it.
         In practice I'd reach for an explicit Redis cache instead, so invalidation
         is visible in my own code."
```

**That answer is stronger than a correct one**, because it demonstrates calibrated confidence — which is the single most useful property in an engineer you're going to trust with production.

### Habit 6 — Connect to something you've actually done

"When I built Vigil, I hit exactly this…" is worth more than any abstract explanation. **This is the entire reason the project exists.** Every major topic in this course maps to something concrete you can point at.

---

## 4. The system design round

Spring-flavoured design rounds are usually 35–50 minutes for something like: *"Design an uptime monitoring service"* (convenient), *"Design a notification service"*, *"Design a URL shortener"*, *"Design a rate limiter"*, *"Design a job scheduler."*

### The framework — spend the time in this ratio

```
① CLARIFY (5 min)              ← ⭐ most candidates skip this and it costs them
② SCALE ESTIMATE (3 min)       ← turns hand-waving into arithmetic
③ API + DATA MODEL (8 min)     ← concrete, and it anchors everything after
④ HIGH-LEVEL DESIGN (10 min)   ← boxes and arrows, happy path
⑤ DEEP DIVE (10 min)           ← let them pick, or pick the interesting part
⑥ FAILURE + SCALE (8 min)      ← ⭐ where senior candidates separate
⑦ TRADE-OFFS + WHAT'S NEXT (3 min)
```

### ① Clarify — do not skip this

**Jumping into a design without clarifying is the most common way strong candidates lose points.** It signals you build what you're told rather than what's needed.

Questions that always apply:
- **Scale:** how many entities, how many requests/second, what growth?
- **Read:write ratio?** Determines caching, replication, and indexing strategy
- **Consistency:** is eventual consistency acceptable, and where isn't it?
- **Latency target:** p99 in milliseconds or seconds?
- **Multi-tenant?** Changes everything about isolation and querying
- **Who are the clients?** Browsers, mobile, other services, third parties?
- **What's explicitly out of scope?** Get this agreed early

**Then state your assumptions out loud:** *"I'll assume 50,000 monitors, 60-second intervals, so ~850 checks/second; read-heavy on the dashboard; eventual consistency is fine for uptime figures but not for quota enforcement. Shout if any of that's wrong."*

**That last clause matters** — it invites correction, which turns the round into a collaboration rather than an examination. Interviewers score collaboration.

### ② Scale estimate — do the arithmetic out loud

```
50,000 monitors ÷ 60s interval          = 833 checks/second
833/s × 86,400s                          = 72 million check results/day
72M × ~100 bytes                         = ~7 GB/day  → ~210 GB/month
                                         → ★ so: partitioning and retention, not one big table

Dashboard: 500 orgs × 1 poll/30s          = ~17 req/s   → trivial
Status pages: 500 pages × 100 viewers/min = ~830 req/s  → ★ THIS is the hot path, so cache it
```

**The estimate drives the design.** Note how it just told you two architectural decisions: partition the results table, and cache the status pages. **Numbers make your design defensible instead of aesthetic** — and doing this out loud is a strong differentiator, because most candidates skip straight to boxes.

Useful figures to have memorized:
```
L1 cache             ~1 ns
Main memory          ~100 ns
SSD random read      ~100 µs
Network same DC      ~500 µs
Postgres simple query ~1 ms
Network cross-region ~50–150 ms
```
```
1 req/s ≈ 86,400/day ≈ 2.6M/month
Little's Law: concurrency = throughput × latency
```

### ③ API + data model

Keep it tight — 5 endpoints and 4 tables. **Name your primary key strategy and say why** (UUIDv7: non-enumerable for a multi-tenant API, time-ordered so inserts don't fragment the index). **Name your indexes.** These details signal you've built something rather than diagrammed something.

### ④ High-level design

Draw boxes. Narrate the happy path end to end. **Say what you're deliberately keeping simple:** *"I'm starting with a modular monolith. I'll show you where I'd split it and why, but I don't want to distribute this until there's a measured reason."*

**Opening with a monolith is a strength, not a weakness** — provided you can articulate the extraction path. Candidates who open with twelve microservices for a greenfield system are usually asked why, and rarely have a good answer.

### ⑤ Deep dive

Offer them a choice: *"I could go deeper on the check scheduling and how it scales horizontally, the data model and retention, or how notifications get guaranteed delivery — which is most useful?"*

**Offering a choice signals confidence in all three.** If they don't choose, pick the one with the most interesting mechanics.

### ⑥ Failure and scale — where the round is actually decided

**Walk through failures systematically.** This is the section most candidates neglect and where senior signals cluster.

```
What if the database is down?            → readiness fails, out of the LB, no cascading restarts
What if Redis is down?                    → degrade to the DB, DEGRADED not DOWN, keep serving
What if a customer's endpoint hangs?      → timeouts, per-target circuit breaker, bulkheaded workers
What if the notification provider is down?→ retries with jitter, circuit breaker, dead-letter, replay
What if an instance dies mid-work?        → claims expire (claimed_at), a reaper reclaims
What if we deploy during an incident?     → graceful shutdown + preStop, zero dropped requests
What if a message is delivered twice?     → unique constraint makes it a no-op
What if one tenant sends 100× traffic?    → per-tenant rate limits; RLS bounds the blast radius
What if the check backlog grows?          → it's the alert AND the autoscaling signal
```

**Then scale it in stages, and be specific about what breaks first:**
```
10×  → read replicas for the dashboard; more checker replicas (already horizontal)
100× → partition check_result by month (done); consider a time-series store;
       PgBouncer because pooled connections × replicas exceeds max_connections
1000×→ shard by org_id; regional checkers for latency and jurisdiction;
       the status pages become fully static on a CDN
```

**Always name what breaks first.** *"At 100× my bottleneck is Postgres connections, not CPU — 40 checker pods × 10 connections exceeds max_connections, so PgBouncer comes before anything else."* That's the answer of someone who has hit a limit rather than read about one.

### ⑦ Trade-offs and what's next

Close by summarizing what you chose *and rejected*:
> *"I chose Postgres `SKIP LOCKED` over a message broker because the work is already in the database and it scales by adding replicas — no new stateful system. I'd revisit that if I needed fan-out to independent consumers or replay. I kept quotas in the same service as monitors to avoid a saga. The main thing I've left out is multi-region, which I'd need for both latency and jurisdictional requirements, and it's the change I'd expect to be hardest."*

**Naming what would be hardest is a confidence signal**, not an admission.

---

## 5. The code review round

Increasingly common, and Spring-specific. You're shown code and asked what's wrong. **Have a scan order** so you don't miss the important things while noticing the trivial ones.

### The scan order — highest severity first

```
1. SECURITY        missing authorization? IDOR? tenant filter? secrets? injection?
2. CORRECTNESS     transaction boundaries? self-invocation? thread safety? nulls?
3. DATA            N+1? entity returned from a controller? missing index? cascade?
4. RESILIENCE      timeouts? unbounded retries? no circuit breaker?
5. TESTABILITY     field injection? static clock? no seam?
6. DESIGN          business logic in the wrong layer? god class?
7. STYLE           naming, formatting — mention last, briefly
```

**Lead with the security and correctness findings.** A reviewer who opens with formatting has told you their priorities.

### A worked example — find the problems

```java
@RestController
@RequestMapping("/api/monitors")
public class MonitorController {

    @Autowired
    private MonitorRepository monitorRepository;

    @Autowired
    private RestTemplate restTemplate;

    private SimpleDateFormat formatter = new SimpleDateFormat("yyyy-MM-dd");

    @GetMapping("/{id}")
    public Monitor get(@PathVariable Long id) {
        return monitorRepository.findById(id).get();
    }

    @GetMapping
    public List<Monitor> list() {
        return monitorRepository.findAll();
    }

    @PostMapping
    @Transactional
    public Monitor create(@RequestBody Monitor monitor) {
        Monitor saved = monitorRepository.save(monitor);
        restTemplate.postForObject("https://hooks.slack.com/xxx",
                "Created " + saved.getName(), String.class);
        return saved;
    }

    @GetMapping("/search")
    public List<Monitor> search(@RequestParam String name) {
        return monitorRepository.searchByName("SELECT * FROM monitor WHERE name = '" + name + "'");
    }
}
```

**Fourteen problems. How many did you find?**

**Security (report these first)**
1. **No authorization anywhere** — any caller can read, list, or create
2. **IDOR** — `findById(id)` with no tenant scoping: any user reads any monitor
3. **`list()` returns every monitor across every tenant** — a full cross-tenant data dump
4. **SQL injection** in `search` — string-concatenated into a native query
5. **Mass assignment** — `@RequestBody Monitor` lets a client set `id`, `status`, `orgId`
6. **`findAll()` with no pagination** — a DoS vector as the table grows
7. Hardcoded Slack webhook URL — a secret in source

**Correctness**
8. **`@Transactional` spanning an HTTP call** — holds a DB connection for the whole round trip → pool exhaustion; and non-atomic (Slack can succeed while the transaction rolls back)
9. **`SimpleDateFormat` as an instance field on a singleton** — not thread-safe, corrupts data under concurrency
10. **`.get()` on an `Optional`** — `NoSuchElementException` with no context instead of a 404
11. **No `RestTemplate` timeout** — a hung Slack blocks a request thread indefinitely

**Data / API**
12. **Returns the entity** — leaks the schema, risks lazy-loading failures, and exposes every field
13. **No `@Valid`** on the request body — no validation at all

**Design / testability**
14. **Field injection** — untestable without reflection, non-final; plus no API version in the path, and `RestTemplate` should be `RestClient`

**How to deliver it:** group by severity, lead with the security cluster, and for the top three explain the *consequence* rather than just naming the rule. *"`@Transactional` around the Slack call means a slow Slack exhausts your connection pool and takes down every endpoint — I've seen exactly that cause an outage"* is far more effective than "don't do I/O in transactions."

**And offer the fix for the worst one**, briefly: publish a domain event and send from `@TransactionalEventListener(AFTER_COMMIT)`, or an outbox.

---

## 6. Behavioural questions — with real answers

Use **STAR** (Situation, Task, Action, Result) but **weight it toward Action and Result** — the situation should be two sentences, not two minutes. And **always end with what you changed structurally**, because that's what's actually being assessed.

### "Tell me about a technical decision you'd make differently"

> *"On Vigil I extracted the notification delivery into its own service. The reasoning was that it talks to unreliable third parties and should be isolated — which is true, but it needs three pods either way, so I got no scaling benefit. What I actually needed was a bulkhead and a circuit breaker inside the monolith; I'd already built both. The extraction cost me a deployable, a database, a migration stream, and a cross-service contract for a benefit I could have had with a `@Bulkhead` annotation.*
>
> *What I took from it: 'this component has different failure characteristics' justifies **isolation**, not necessarily a **process boundary** — and I'd now check whether an in-process mechanism gets me the same property first. I wrote it up as a retrospective in the repo, partly so I'd remember."*

**Why this works:** it's specific, it's genuinely self-critical, it names the mistaken reasoning rather than just the mistake, and the lesson generalizes. **This is the highest-value behavioural answer you have** — prepare it properly.

### "Tell me about a difficult bug"

Use the Lesson 43 §8 structure: symptom → wrong assumption → how you found the truth → fix → structural change. **Prepare two.**

### "How do you handle disagreement about a technical decision?"

> *"I try to find the thing we actually disagree about, because it's usually not the technology. Someone wanted to add Kafka for what was, in practice, one producer and one consumer. Arguing about Kafka wasn't going to work, so I asked what problem we were solving. It turned out he was worried the consumer would fall behind during traffic spikes — a legitimate concern I hadn't considered.*
>
> *So we agreed on the requirement first: absorb a 10× spike without losing work. Then we listed options — a database queue with `SKIP LOCKED`, an in-memory queue with an outbox, or Kafka — and compared them against that requirement plus operational cost. We went with the database queue and agreed a specific trigger for revisiting: if consumer lag exceeded a minute at p99, we'd move to a broker.*
>
> *The pattern I try to follow: agree on the requirement before arguing about the solution, and define what would change our minds. Otherwise it's just preference versus preference."*

**"Define what would change our minds" is the senior move in that answer** — it converts an argument into a decision with an expiry date.

### "How do you approach an unfamiliar codebase?"

> *"Run it first — I want to see it work before I read it. Then find the entry points: for a Spring app that's `@RestController`s, `@KafkaListener`s and `@Scheduled` methods, which tells me everything the system does from the outside. Then I trace one important request all the way down.*
>
> *After that I read the schema, because the data model is usually the most honest description of the domain — code lies about intent, schemas don't. Then the tests, which tell me what the team believes is important and what they're afraid of breaking.*
>
> *And I write things down as I go, because whatever confused me will confuse the next person — a couple of those notes usually become the README section that was missing."*

### "How do you decide what to test?"

> *"By risk and by change frequency. Business logic that encodes real rules — pricing, state machines, uptime calculation — gets thorough unit tests, because a subtle error there is expensive and silent. Anything crossing a boundary gets integration tests against real infrastructure, because that's where bugs actually are and mocks prove nothing about SQL or serialization.*
>
> *Two things I always test that people often skip: negative security cases — user A cannot read user B's data, one test per endpoint — and that timeouts actually fire, because timeout config is easy to get wrong or silently override, and an unenforced timeout is how one slow dependency takes down a whole service.*
>
> *And every bug I fix gets a regression test, without exception. That's the test with the highest proven value, because it's the only one where I know the failure was real."*

### "Where do you want to be in three years?"

Answer honestly, but connect it to depth: *"I want to be the person a team relies on for hard technical judgment — which mostly means having seen enough production failures to have good instincts. Concretely: more distributed systems experience, more time on-call, and more design ownership rather than implementation."*

---

## 7. Questions to ask *them*

The questions you ask are assessed. Ask things that reveal engineering culture — and that you'd genuinely want to know.

**Strong questions:**
- *"How long does it take from merging a PR to it running in production?"* — reveals CI/CD maturity, and therefore what your days will be like
- *"What does on-call look like, and how often does someone get paged at night?"* — reveals operational health and alert discipline
- *"What's the most painful part of the codebase, and what stops you fixing it?"* — reveals honesty and whether tech debt is acknowledged
- *"How do you decide what to build?"* — reveals whether engineers have input
- *"What happened after your last significant incident?"* — reveals whether they do blameless postmortems or find someone to blame
- *"How much of the test suite do you trust? Do people re-run CI until it goes green?"* — a very revealing question about flakiness culture
- *"If I joined, what would you want me to have shipped in the first month?"* — reveals whether the role is actually defined

**Weak questions:** "What's the tech stack?" (it's in the job ad), "Is there room for growth?" (nobody says no), anything you could have Googled.

**The pattern:** ask about *process and reality*, not facts. And **listen to how they answer** — hesitation on the on-call question tells you more than the answer does.

---

## 8. Red flags to avoid

| Don't | Why it costs you |
|---|---|
| Bluff | Interviewers probe, and a collapsing bluff destroys credibility for everything else you said |
| Say "best practice" without a reason | Best practices are context-dependent; citing them as authority signals you haven't tested them |
| Criticize a previous employer or colleague | You become the risk. Describe the *problem* without blaming *people* |
| Over-engineer the design round | Proposing microservices, Kafka and Kubernetes for a CRUD app is the most common strong-candidate failure |
| Skip clarifying questions | Signals you build what you're told rather than what's needed |
| Ignore the interviewer's hints | *"What if there were two instances?"* is not idle curiosity — follow it |
| Only design the happy path | Failure handling is where the round is decided |
| Say "it's just a side project" | You're discarding your own leverage |
| Claim "production-ready" | Invites a demolition. Show evidence instead |
| Go silent while thinking | Narrate: *"I'm weighing two options — let me think out loud."* They're assessing your process, and silence gives them nothing |

---

## 9. The twelve statements that signal expertise

Each is a compressed, memorable version of something from this course. **Deploy them naturally, not as a list.** Any one of them, said at the right moment, changes how you're perceived.

1. *"`this.method()` bypasses the proxy, so the annotation silently does nothing."* — **the single most useful sentence in Spring**
2. *"Annotations do nothing. Something has to read them via reflection and act."*
3. *"Auto-configuration is just `@Conditional` plus a text file of class names, evaluated last so your beans win."*
4. *"`open-in-view` is true by default and it's a connection-pool exhaustion risk — I turn it off day one."*
5. *"That's an N+1. I'd assert the query count in a test, because it's a regression, not a one-off."*
6. *"Checked exceptions don't roll back by default. That surprises everyone exactly once."*
7. *"An HTTP call inside a transaction holds a database connection for the whole round trip — one slow third party and your pool is gone."*
8. *"Return 404, not 403, for cross-tenant access. A 403 confirms the resource exists."*
9. *"Timeouts on everything. The defaults are effectively infinite, and that's how one service's slowness becomes your outage."*
10. *"`@Scheduled` has no cluster awareness — three replicas run everything three times."*
11. *"Exactly-once delivery is impossible. What you want is at-least-once plus an idempotent consumer."*
12. *"I'd start with a modular monolith and extract when there's a measured reason. Microservices solve an organizational problem."*

**And the meta-statement, which is worth more than all twelve:**

> *"I don't know that, but here's how I'd find out — and here's the adjacent thing I do know."*

---

## 10. Your final self-assessment

**Go through this honestly.** For each item: can you explain it out loud, to an imagined colleague, without notes, in under 90 seconds?

**Mechanisms — the foundation**
- [ ] Annotations, reflection, proxies: why Spring isn't magic
- [ ] The container: definitions vs instances, the startup sequence, `BeanPostProcessor`
- [ ] Self-invocation, and the five annotations it silently breaks
- [ ] Auto-configuration: `DeferredImportSelector`, conditions, why ordering matters

**Web**
- [ ] A request end to end, naming six Spring components
- [ ] Thread-per-request, starvation, and what virtual threads do and don't fix
- [ ] Filter vs interceptor vs aspect, with a concrete example each
- [ ] Why entities never leave the service layer

**Data — the deepest area**
- [ ] The persistence context: identity, dirty checking, write-behind
- [ ] Managed vs detached, and why an update silently didn't persist
- [ ] `open-in-view`, and three reasons to disable it
- [ ] N+1: five ways it hides, and how you *detect* it
- [ ] `@Transactional`: propagation, `UnexpectedRollbackException`, checked exceptions
- [ ] Optimistic vs pessimistic locking, and `SKIP LOCKED`
- [ ] Expand/contract, and why rolling deploys force it

**Security**
- [ ] The filter chain, and 401 vs 403
- [ ] CSRF: when to disable, and why "we use JWT" isn't sufficient
- [ ] JWT revocation, and why it undermines statelessness
- [ ] IDOR, query scoping, and 404-over-403
- [ ] Four layers of tenant isolation

**Production**
- [ ] `@Scheduled` across replicas: two solutions and when each applies
- [ ] The transactional outbox, and what it fixes that `AFTER_COMMIT` doesn't
- [ ] Timeouts, jitter, circuit breakers, bulkheads — and idempotency underneath all of it
- [ ] Liveness vs readiness, and the restart cascade
- [ ] Cardinality, percentile aggregation, and what to alert on
- [ ] Container memory: why `-Xmx` is wrong and OOMKill is silent

**Architecture & judgment**
- [ ] Package-by-feature, and the compiler-enforcement argument
- [ ] When to extract a service — and when to redraw a boundary instead
- [ ] Exactly-once, honestly
- [ ] The argument *against* microservices
- [ ] Zero-downtime deploys, including the preStop sleep

**Yourself**
- [ ] The 60-second Vigil pitch
- [ ] Two "hardest bug" stories with structural lessons
- [ ] One decision you'd make differently, and why the reasoning was wrong
- [ ] Three questions you'd genuinely want to ask an employer

**If you can do all of that, you are — functionally and demonstrably — an expert.** Not because you've memorized answers, but because every one of those items is a *mechanism* you can reason from, which means you can also handle the questions this list doesn't contain.

---

## 11. Where to go from here

You've reached the end of the curriculum. **The material stops being the constraint now; experience becomes the constraint.** Three things compound faster than more reading:

**1. Keep Vigil alive.** A project that's been running for a year, with real uptime data, a few external users, and a commit history showing sustained work, is worth more every month you keep it. Add one feature a month. Respond to issues. **A deployed system you've operated is the closest thing to production experience you can manufacture.**

**2. Read primary sources, not tutorials.** The Spring Framework reference docs are genuinely excellent and almost nobody reads them. Then read *source code* — `SimpleJpaRepository`, `TransactionAspectSupport`, `AutoConfigurationImportSelector`, `FilterChainProxy`. **You now have enough context to read them, which you didn't before.** An afternoon in `TransactionAspectSupport` will teach you more than any blog post about `@Transactional`.

**3. Write about what you learn.** A blog post forces you to find the gaps in your own understanding — you cannot write clearly about something you only half-know. It also builds the communication ability that senior interviews explicitly score. Write about `SKIP LOCKED`. Write about the preStop sleep. Write about the bug you spent two days on.

**Books genuinely worth the time**, in this order:
- *Designing Data-Intensive Applications* (Kleppmann) — **the most valuable technical book of the last decade.** Everything in Modules 9–10 goes deeper here
- *Release It!* (Nygard) — where the resilience patterns in Lesson 34 come from, with the war stories
- *High-Performance Java Persistence* (Mihalcea) — Module 6, at three times the depth
- *Java Concurrency in Practice* (Goetz) — dated on APIs, still the definitive treatment of the model

---

## One last thing

The goal at the start was: **when someone in the room has a Spring question, you are the person they turn to.**

That doesn't come from knowing more annotations. It comes from a specific and slightly unusual habit: **an unwillingness to accept that anything "just works."** Every time you hit something that behaves surprisingly and you chase it to the mechanism — the proxy, the ThreadLocal, the flush ordering, the condition evaluation — you become a little more the person other people ask.

That habit is the actual skill. The 45 lessons were just a structured way to practise it on a large surface area.

**Now go break something on purpose and find out why.**

---

## Course complete 🎓

| Module | Lessons | What you can do |
|---|---|---|
| 1 | 01–04 | Explain Spring's "magic" mechanically |
| 2 | 05–08 | Describe the container completely |
| 3 | 09–10 | Diagnose every proxy-related bug |
| 4 | 11–13 | Trace why any bean exists; write a starter |
| 5 | 14–18 | Design and reason about the web layer |
| 6 | 19–24 | Persistence from the pool up — the deepest module |
| 7 | 25–28 | Authenticate, authorize, and harden |
| 8 | 29–31 | Test in a way that makes change safe |
| 9 | 32–36 | Run a service in production |
| 10 | 37–39 | Distribute a system — and argue against doing so |
| 11 | 40 | Ship it to a real URL |
| 12 | 41–43 | Build, deploy and present Vigil |
| 13 | 44–45 | Interview like the expert you now are |

**← [Back to the table of contents](../README.md)**

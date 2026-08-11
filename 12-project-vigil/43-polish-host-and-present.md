# Lesson 43 — Polish, host & present it

> **A project nobody looks at is worth nothing.** This lesson is about the last 5% of effort that determines whether Vigil gets you interviews or sits unviewed on GitHub. It's the least technical lesson in the course and possibly the highest-return one.

**Time:** ~50 minutes of reading, several hours of doing

---

## 1. The brutal reality of how your project is evaluated

A recruiter or engineer looking at your GitHub gives you, honestly:

```
 5 seconds   → the repo name, description, and whether there's a live link
20 seconds   → the top of the README: does this look real?
 2 minutes   → IF the README survived: click the demo, skim the architecture
10 minutes   → IF still interested: read some code, check the commit history
```

**Almost nobody clones your repo. Almost nobody runs it.** Which means your README and your live demo *are* your project as far as most evaluation is concerned. The code matters — but only after those two have earned the click.

**The three things that convert a skim into a conversation:**
1. **A live URL that works, with a demo login** — proves it's real, not aspirational
2. **A README that explains *why*, not just *what*** — proves you think, not just type
3. **One genuinely interesting technical detail, stated early** — gives the interviewer something to ask about

That third point is underrated. **Interviewers are looking for a hook** — something specific to dig into. Give them one deliberately rather than hoping they find it.

---

## 2. The README

This is the most important file in your repository. Structure it in this order — **the order is deliberate**, because each section only gets read if the previous one worked.

```markdown
# Vigil

**Self-hostable uptime monitoring and incident management.** Multi-tenant, horizontally
scalable, with public status pages and incident escalation.

🔗 **[Live demo](https://vigil.example.dev)** — `demo@vigil.io` / `demo`
📊 **[Public status page](https://vigil.example.dev/status/demo)**
📖 **[API docs](https://vigil.example.dev/swagger-ui.html)**

![Dashboard screenshot](docs/images/dashboard.png)

---

## What it does

Register HTTP/TCP/SSL/heartbeat monitors. Vigil checks them on schedule from a pool of
workers, opens and escalates incidents on failure, delivers notifications via
Slack/email/webhook, and publishes cached public status pages with uptime and SLA reporting.

## Why I built it

I wanted a project where every piece of infrastructure was *required by the domain* rather
than added for a CV. Uptime monitoring inherently needs distributed scheduling, resilient
outbound HTTP, high-volume time-series writes, caching, async delivery, and multi-tenancy —
so every architectural decision has a real justification I can defend.

## The technically interesting parts

**Horizontal scaling without a coordinator.** Check workers claim due monitors with
`SELECT ... FOR UPDATE SKIP LOCKED`, so every instance takes a disjoint batch. Scaling is
just adding replicas — no message broker, no leader election, no ShedLock on the hot path.
Measured: 5 workers process a 10,000-monitor backlog ~5× faster than 1, with no duplicate
checks. ([code](src/main/java/com/vigil/check/CheckQueueRepository.java) ·
[test](src/test/java/com/vigil/check/ConcurrentClaimingIT.java))

**Transactional outbox for guaranteed delivery.** A monitor's status change and the
notification it owes are written in one database transaction, then relayed to Kafka by a
poller. Verified by `kill -9` between commit and relay: the notification still arrives.
Delivery is at-least-once, so consumers dedupe via a unique constraint.

**Four layers of tenant isolation**, ending in Postgres row-level security — so a bug in
application code still can't leak another tenant's data. There's a test that removes the
application-level filter and asserts the database blocks it anyway.

## Architecture

![Architecture diagram](docs/images/architecture.png)

Modular monolith with one justified service extraction. See
[ARCHITECTURE.md](docs/ARCHITECTURE.md) and the [ADRs](docs/adr/).

## Performance

Measured on 5,000 monitors / 2,000,000 check results (Hetzner CPX21, 3 vCPU / 4GB):

| Endpoint | Before | After | Change |
|---|---|---|---|
| `GET /monitors` (p99) | 2,410ms | 178ms | composite index + projection |
| `GET /status/{slug}` (p99) | 890ms | 12ms | Redis + stale-while-revalidate |
| Check throughput | 40/s | 850/s | virtual threads + batched writes |
| Deploy downtime | 340 failed reqs | **0** | graceful shutdown + preStop |

[Load test scripts](load-tests/) · [full methodology](docs/PERFORMANCE.md)

## Running it

```bash
git clone https://github.com/you/vigil && cd vigil
cp .env.example .env          # fill in JWT_SECRET
docker compose up
```
→ http://localhost:8080 · Grafana at :3000 · Swagger at /swagger-ui.html

## Tech

Java 21 · Spring Boot 3.5 · Postgres 16 (partitioned, RLS) · Redis · Kafka ·
Flyway · Testcontainers · Resilience4j · Micrometer + Prometheus + Grafana + Tempo ·
Docker · GitHub Actions

## Known limitations

- Check scheduling drifts up to the 5s dispatch interval — fine for uptime monitoring,
  not for anything requiring precise timing
- Status pages are up to 30s stale (deliberate: correctness of freshness matters less
  than availability during an incident)
- Single-region Postgres is the availability ceiling
- Minimal dashboard UI — this is a backend project and I've kept it scoped as one

## What I'd do differently

Extracting the notification service wasn't worth it — it needs 3 pods either way and the
only real benefit was isolating third-party retry state. See the
[retrospective](docs/MICROSERVICES-RETROSPECTIVE.md).
```

### Why this structure works

| Section | What it's doing |
|---|---|
| **Live links above the fold** | Proves it's real in 5 seconds. This is the single highest-impact element. |
| **A screenshot immediately** | People are visual; a wall of text gets skipped |
| **"Why I built it"** | Pre-empts "is this a tutorial follow-along?" — the question every reviewer silently asks |
| **"Technically interesting parts" with links to code and tests** | ⭐ **Gives the interviewer their hook.** Also proves the claims are testable |
| **Measured performance numbers** | Almost nobody does this. It's the difference between "I know about indexes" and "I measured a 13× improvement" |
| **One-command run** | Removes every excuse not to try it |
| **Known limitations** | Signals engineering judgment. A project claiming no limitations reads as inexperience |
| **"What I'd do differently"** | ⭐ **The most senior thing in the file.** Willingness to critique your own architecture |

**The two starred rows are what separate this README from a good one.** Specific technical claims backed by linked tests, and honest self-critique.

### README mistakes that cost you

| Mistake | Why it hurts |
|---|---|
| Opening with a tech stack list | Tells the reader nothing about what it does or why it's interesting |
| No live link | The reviewer must take your word that it works |
| No screenshots | Backend projects are invisible without them — screenshot Grafana, a trace, the status page |
| A wall of setup instructions at the top | Nobody is setting it up before they know if they care |
| Claiming "production-ready" or "enterprise-grade" | These phrases read as inexperience. Show the evidence instead |
| Buzzword bingo ("leveraging cutting-edge microservices") | Plain language reads as competence |
| A `TODO` list of unbuilt features | Makes the project feel unfinished. Put those in issues |

---

## 3. The supporting documents

Four files, each doing a specific job. **These are read more often than you'd expect** — a reviewer who's interested goes looking for exactly this.

### `docs/ARCHITECTURE.md`

```markdown
# Architecture

## Boundaries and why they're there

`check` is separate from `monitor` because monitor *configuration* changes rarely and is
read-heavy, while check *execution* runs continuously at ~850/s and is write-heavy,
append-only, and time-series shaped. Different scaling profile, different data access
pattern, different backup policy.

`organization`, `monitor`, `incident` and `billing` deliberately stay together: creating a
monitor must check the plan quota in one transaction. Splitting them would require a saga,
and redrawing the boundary is better engineering than writing one.

## The dependency rule
Features never import each other's repositories or entities. Communication is via public
service interfaces or domain events. Enforced by package-private visibility and
[ArchUnit tests](../src/test/java/com/vigil/architecture/ArchitectureTest.java).

## What I'd extract next, and why
`statuspage` — it must stay available when everything else is down, and it's the only
component whose availability requirement differs from the rest. Arguably it should be a
statically generated page on a CDN with no application involvement at all.
```

**The "what I'd extract next" section is a gift to an interviewer** — it's an invitation to a design conversation you're prepared for.

### `docs/adr/001-extract-checker-service.md`

Use the format from [Lesson 38](../10-microservices/38-microservices-fundamentals.md) §7: Context (with **measurements**) → Decision → Consequences (**including the negatives**) → Alternatives considered (**including what you rejected and why**).

**ADRs are one of the strongest artifacts in a portfolio** because they're evidence of *reasoning*, and reasoning is what's actually being assessed. Write 3–5. Include one where you chose the boring option.

### `docs/PERFORMANCE.md`

The methodology behind your numbers: hardware, data volume, load profile, what you changed, and the before/after. **A number without methodology is a claim; a number with methodology is evidence.**

### `docs/MICROSERVICES-RETROSPECTIVE.md`

Where you say what wasn't worth it. **This file will get you more credit than any feature.**

---

## 4. The repository itself

**Things that are noticed in the 10-minute code skim:**

- [ ] **Commit history tells a story.** Conventional commits (`feat(check): …`, `fix(auth): …`), one logical change each, no `wip`/`asdf`/`fix stuff`
- [ ] **No commented-out code.** Delete it; git remembers
- [ ] **No `TODO` without a ticket reference.** Bare TODOs read as abandonment
- [ ] **Consistent formatting** — Spotless in CI means every file looks deliberate
- [ ] **A green CI badge** in the README, plus coverage if you're comfortable with the number
- [ ] **`.env.example` committed, `.env` gitignored**, and gitleaks in CI
- [ ] **`CONTRIBUTING.md`** — even if nobody contributes, it signals you think about maintainers
- [ ] **Issues used** for known work rather than a README TODO list
- [ ] **A tagged release** (`v1.0.0`) with notes. Free, and it makes the project look finished
- [ ] **Topics/tags** on the GitHub repo (`spring-boot`, `java`, `monitoring`, `postgresql`) so it's discoverable

**On squashing:** don't squash your entire history into one commit before publishing. A history showing incremental, thoughtful development over weeks is *evidence of process*. One giant "initial commit" throws that away and looks like a copied repo.

---

## 5. The live deployment checklist

**The demo is the project.** Get these right.

- [ ] **A custom domain.** `vigil.yourdomain.dev` reads as intentional; `vigil-prod-a1b2.fly.dev` reads as a throwaway
- [ ] **HTTPS with a valid certificate** (free via the platform)
- [ ] **A demo login prominently displayed** in the README *and* pre-filled on the login page. ⭐ **Nobody will register to evaluate your project** — this single item probably doubles your engagement
- [ ] **Seeded realistic demo data.** Not `test1`, `test2`, `asdf`. Real-looking monitors with plausible names, a mix of statuses, and **weeks of history** so the charts aren't empty
- [ ] **A real public status page** monitoring genuinely public endpoints (github.com, your own site) so a visitor sees the product actually working
- [ ] **Hourly demo-data reset** (a `@Scheduled` job) so visitors can't leave it broken
- [ ] **Rate limiting on the demo account** — you're publishing an endpoint that makes outbound HTTP requests to user-supplied URLs. **Cap it, restrict schemes, and block private ranges.** (You built this in Phase 1; verify it's on.)
- [ ] **A cost cap or budget alert.** A runaway check loop on a paid platform is a real risk
- [ ] **It stays up.** A dead demo link is worse than no demo link — it actively signals abandonment. Monitor it externally

**On the SSRF risk specifically:** you're hosting a service whose purpose is fetching arbitrary user-supplied URLs, publicly. That's a genuine abuse vector — for port scanning, for hitting internal cloud metadata, and for using your server as a request proxy. **Verify your validation and your egress rules before the demo is public.** Being able to say "I thought carefully about hosting a public SSRF-shaped service and here's what I did" is itself a strong point.

---

## 6. Putting it on your resume

**Bad:**
> *Vigil — a monitoring application. Used Java, Spring Boot, PostgreSQL, Redis, Kafka, Docker.*

**Good:**
> **Vigil** — Open-source uptime & incident management platform · [vigil.example.dev](https://vigil.example.dev) · [github.com/you/vigil](https://github.com/you/vigil)
> Multi-tenant SaaS monitoring thousands of endpoints on schedule. Horizontally scaled check workers claim disjoint batches via Postgres `SKIP LOCKED` (no broker or leader election); measured ~850 checks/s. Transactional outbox for guaranteed notification delivery, verified across process kills. Four-layer tenant isolation ending in row-level security. Cut p99 latency 2.4s → 178ms via indexing and projections; zero-downtime deploys verified under load.
> *Java 21, Spring Boot 3.5, Postgres (partitioned, RLS), Redis, Kafka, Testcontainers, Prometheus/Grafana/Tempo, Docker, GitHub Actions*

**What changed:**
- **A live URL first.** Everything else is a claim until that link works
- **Specific numbers** (850/s, 2.4s → 178ms, zero failed requests). Numbers are checkable, so they read as true
- **Named techniques** (`SKIP LOCKED`, outbox, RLS) — these are *keyword hooks* an experienced interviewer will latch onto
- **Verification mentioned** ("verified across process kills") — signals you test claims rather than assume them
- **Stack last**, because the stack is the least interesting thing about it

**Length guidance:** 3–4 lines maximum on a resume. **Every line must contain something checkable.** Cut anything that could be said about any project.

---

## 7. Talking about it in an interview

### The 60-second version — rehearse this until it's automatic

> *"Vigil is a self-hostable uptime and incident management platform — multi-tenant, deployed, with a public demo. Users register monitors; a pool of workers checks them on schedule, opens and escalates incidents on failure, and publishes cached public status pages.*
>
> *I chose it because every piece of infrastructure in it is required by the domain rather than added to look impressive — scheduling, resilient outbound HTTP, high-volume time-series writes, caching, and multi-tenancy all fall out of the problem.*
>
> *The part I'd most want to talk about is how the check workers scale: every instance claims a disjoint batch of due monitors using `SELECT FOR UPDATE SKIP LOCKED`, so scaling is just adding replicas — no broker, no leader election. And it's deliberately a modular monolith with one justified extraction; I have a retrospective explaining which part of that split wasn't worth doing."*

**That last sentence is doing real work.** It pre-empts "did you use microservices because they're fashionable?" and signals judgment before you're asked for it.

### The follow-ups you *will* get — prepare all of these

| Question | Your material |
|---|---|
| "Walk me through the architecture" | Lesson 41 §4 — and lead with *why* each boundary is there |
| "How does the scheduling work across instances?" | `SKIP LOCKED`, why not ShedLock on the hot path, what happens if a worker dies mid-claim |
| "How do you prevent one tenant seeing another's data?" | Four layers, ending with RLS + the test that removes the app filter |
| "What was the hardest bug?" | **Have a real one ready.** §8 below |
| "What would you do differently?" | The retrospective + the notification-split admission |
| "How do you know it works?" | Test pyramid, Testcontainers, cross-tenant 404 tests, N+1 query-count assertions, the timeout test |
| "How does it perform?" | The measured table, and the *method* you used to get there |
| "What happens if Redis dies?" | Degrades to the database, health reports DEGRADED not DOWN, stays in the load balancer — *and you can demo it* |
| "Why not use X instead?" | Have a considered answer for Quartz, Eureka, H2, ModelMapper, RabbitMQ |
| "What's not finished?" | Lesson 41 §9 — deliberate scope limits, stated as choices |

### Two things to avoid

**Don't oversell.** "Production-ready, enterprise-grade, handles millions of users" invites a demolition. Say what you measured and what you didn't.

**Don't undersell either.** "It's just a small side project" throws away your own leverage. It's a deployed, tested, monitored, multi-tenant service — describe it accurately.

---

## 8. The "hardest bug" story

**You will be asked this.** It's one of the most reliable behavioural questions in engineering interviews, and a vague answer wastes a real opportunity.

**Structure it as: symptom → what you assumed → how you found the truth → the fix → what you changed structurally.**

**Vigil gives you several genuinely good ones. Pick one you actually hit:**

> *"Notifications were being sent for incidents that didn't exist in the database. Intermittently — maybe one in fifty.*
>
> *My first assumption was a race between the incident write and the notification read. I added logging and found the notification and the incident had the same correlation ID, so it wasn't concurrency between requests.*
>
> *What it actually was: I was publishing the domain event with a plain `@EventListener`, which runs **inside** the publisher's transaction. When a later step in that transaction failed and rolled back, the notification had already been sent. The database was correct; the outside world wasn't.*
>
> *The fix was `@TransactionalEventListener(AFTER_COMMIT)`, so the listener only runs if the transaction actually committed. But that still had a gap — the process could die between commit and send — so I moved to a transactional outbox: the notification is written to a table in the same transaction as the state change, and a relay publishes it. I verified it by `kill -9`-ing the app between commit and relay and confirming the notification arrived after restart.*
>
> *The structural change was realizing that any time I do I/O inside a transaction, I've created a consistency gap between my database and the outside world. I now treat that as a rule rather than a case-by-case decision."*

**Why this story works:**
- A concrete, intermittent symptom (credible — real bugs are intermittent)
- **A wrong first hypothesis**, stated openly (honest, and shows a debugging process)
- A specific mechanism identified (demonstrates real understanding of transaction/proxy semantics)
- **Two fixes, with the first shown to be insufficient** (shows you kept going past "it works now")
- **Verified deliberately** (`kill -9`) rather than assumed
- **A generalized lesson** at the end — this is the part interviewers are actually scoring

**Other good candidates from this project:** the `@Cacheable` multi-tenant key leak (a data breach in one annotation), the liveness-probe-checks-the-database cascade, the `IDENTITY` ID strategy silently disabling batch inserts, or the container OOMKill with no Java exception.

**Prepare two.** Sometimes the first one leads somewhere the interviewer isn't interested in.

---

## 9. Beyond the interview — making it a real open-source project

Optional, and disproportionately valuable if you do it.

- **Publish it properly:** a clear license, a `CONTRIBUTING.md`, issue templates, and good first issues
- **Write about it.** A blog post — *"Horizontal work distribution with Postgres SKIP LOCKED, no broker required"* — is genuinely useful content and it demonstrates communication ability, which is scored in every senior interview
- **Post it where practitioners are:** r/java, Hacker News (Show HN), lobste.rs. Be ready for critical feedback; **responding well to criticism in public is itself a strong signal**
- **Respond to issues.** Even one real external user changes the story from "a portfolio project" to "software people use"
- **A `SECURITY.md`** with a disclosure policy. Small, and it signals you take running a service seriously

**The compounding effect:** a project with external users, a blog post, and a few closed issues stops being a portfolio piece and becomes professional experience you can describe in the same terms as employment.

---

## 10. The final checklist

Print this. Go through it literally.

**The repository**
- [ ] README with live links above the fold, a screenshot, and "why I built it"
- [ ] "Technically interesting parts" section with links to code *and* tests
- [ ] A measured performance table with methodology
- [ ] Known limitations and "what I'd do differently"
- [ ] `ARCHITECTURE.md`, 3+ ADRs, `PERFORMANCE.md`, retrospective
- [ ] Clean conventional commit history, no commented-out code, no bare TODOs
- [ ] Green CI badge; Spotless-formatted; gitleaks passing
- [ ] `.env.example` committed, no secrets anywhere in history
- [ ] A tagged `v1.0.0` release with notes
- [ ] GitHub topics set

**The deployment**
- [ ] Live on a custom domain with valid HTTPS
- [ ] Demo login in the README and pre-filled on the login page
- [ ] Realistic seeded data with weeks of history
- [ ] A working public status page monitoring real endpoints
- [ ] Hourly demo reset; rate limits and SSRF protections verified live
- [ ] Actuator not publicly reachable
- [ ] Externally monitored, so you know if it goes down
- [ ] A cost cap configured

**Your preparation**
- [ ] The 60-second pitch, rehearsed out loud
- [ ] The 5-minute demo script, rehearsed with the live app
- [ ] Two "hardest bug" stories with the symptom → wrong assumption → truth → fix → lesson structure
- [ ] An answer for each follow-up in §7
- [ ] A considered "why not X?" for Quartz, Eureka, H2, ModelMapper, RabbitMQ, microservices
- [ ] You can draw the architecture on a whiteboard from memory

**Optional, high leverage**
- [ ] A blog post about one technical decision
- [ ] Posted somewhere practitioners will see it
- [ ] `SECURITY.md` and issue templates

---

## Module 12 complete ✅

Vigil is built, deployed, documented, and you can talk about it fluently. **That combination is rarer than any individual skill in this course.**

**Next:** [Lesson 44 — Rapid-fire master Q&A](../13-interview/44-rapid-fire-master-qa.md) — the whole course, revisable in 90 minutes.

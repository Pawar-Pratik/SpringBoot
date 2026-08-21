# Spring Boot: Zero → Absolute Authority

> The goal of this curriculum is not "you can build a CRUD app."
> The goal is: **when someone in the room has a Spring question, you are the person they turn to.**

This is a self-contained, sequential course. No videos. No courses. Just deep, honest explanations of how Spring actually works, why it was built that way, what happens in production when you get it wrong, and one serious project you'll be proud to put on your resume.

---

## How this course is different

Most Spring material teaches you **incantations**: "add `@Transactional` and it works." Then an interviewer asks *"why didn't your `@Transactional` method roll back?"* and the whole illusion collapses.

This course teaches **mechanisms**. Every single "magic" feature in Spring is built from a small number of boring, understandable ideas:

1. A `Map` of objects (the container)
2. Reflection reading annotations
3. Proxies wrapping your objects
4. Conditional configuration based on the classpath

That's genuinely most of it. Once you see those four gears turning, nothing in Spring surprises you again — including features that didn't exist when you learned it.

### The rules I follow in every lesson

| Rule | What it means |
|---|---|
| **Mechanism before syntax** | You learn *why the problem exists* before you learn the annotation that solves it |
| **No orphan facts** | Every fact connects to something you can do or debug |
| **Production reality** | I tell you what breaks at 3am, not just what compiles |
| **Interview traps** | Each lesson ends with the questions people confidently get *wrong*, and why |
| **Build it** | Each lesson advances the Vigil project, so knowledge lands in muscle memory |
| **No padding** | I will not write 60 lessons about annotations. Important things get depth; trivia gets a table |

---

## The project you'll build: **Vigil**

**Vigil** is an open-source uptime & incident management platform — think a self-hostable Better Uptime / Statuspage / Pingdom.

- You register monitors (HTTP endpoints, TCP ports, SSL certs, cron heartbeats).
- Vigil checks them on a schedule from a pool of workers.
- When something breaks, it opens an **incident**, escalates through a policy, and notifies people via email/Slack/webhook.
- It publishes a **public status page** and **SLA/uptime reports**.
- It monitors *itself* with metrics, traces, and health probes.

**Why this project and not a "todo app" or another e-commerce clone:**

1. **It's genuinely useful.** You can host it, use it for your own side projects, and other people will too. That's a very different resume conversation than "I built a shopping cart."
2. **It cannot be built without real Spring depth.** Uptime monitoring *inherently* needs scheduling, outbound HTTP with timeouts, retries, circuit breakers, async notification, time-series data at volume, caching, and horizontal scaling. You don't have to bolt on technologies to look impressive — the domain demands them.
3. **The microservices split is honest.** Most "microservices portfolios" are a monolith chopped into arbitrary pieces. Here, check-workers genuinely must scale independently of the API. You'll be able to defend every boundary in an interview.
4. **It's a portfolio differentiator.** Interviewers have seen 500 food-delivery clones. They have not seen a candidate who built and hosted a working monitoring platform.

Full blueprint: **[Vigil Blueprint](12-project-vigil/41-vigil-blueprint.md)**

> Read the blueprint early (skim it after Module 2) so you know where you're heading — but don't start coding it until the lessons tell you to.

---

## Before you start: setup

You have Java 17. That works, but **install Java 21** — it's the LTS the industry is standardizing on, and it brings virtual threads, which change how Spring handles concurrency (a topic that now shows up in interviews constantly).

| Tool | Version | Why |
|---|---|---|
| **JDK** | 21 (LTS) | Records, pattern matching, virtual threads. Get it from [Adoptium Temurin](https://adoptium.net/) |
| **IDE** | IntelliJ IDEA (Community is fine) | Spring support is dramatically better than VS Code's. Use it. |
| **Maven** | 3.9+ | Bundled with IntelliJ; we'll use the Maven wrapper anyway |
| **Docker Desktop** | latest | Non-negotiable from Module 6 onward — Postgres, Redis, Kafka, Testcontainers |
| **Postman / Bruno / `curl`** | any | API poking. Bruno is free and file-based, so it commits to git |
| **DB client** | DBeaver or IntelliJ's | You'll be reading real SQL |

We target **Spring Boot 3.5.x** on **Java 21**. That is what the overwhelming majority of production codebases and interviews are on. Where Spring Boot 4 / Spring Framework 7 changes something meaningful, you'll see a `> **Boot 4 delta**` callout — so you're never caught out by a version question.

---

## How to actually study this (read this part, it matters)

Reading is the *least* effective part of learning. The plan:

### The loop, per lesson
1. **Read the lesson once, fast.** Don't take notes. Just build the map.
2. **Read the "Interview traps" section and try to answer before reading the answers.** Failing to recall is what builds memory — being *taught* the answer builds recognition, which feels identical and is nearly useless in interviews.
3. **Do the "Build this" section.** Type the code. Do not copy-paste. Typing forces you to read every token.
4. **Break it deliberately.** Every lesson has a "Break it on purpose" step. Remove the annotation. Watch the exact exception. That exception text is now yours forever — and *"I've seen that stack trace"* is the single most credible thing you can say in an interview.
5. **Explain it out loud in 60 seconds,** as if to a colleague. If you stumble, you don't know it yet. This is the whole game.

### The rule about "magic"
Any time you think *"this just works somehow"* — **stop**. That's a debt. Write the question down in `QUESTIONS.md` and hunt it down. Mastery is just an unusually low tolerance for unexplained behaviour.

### Pacing

| Intensity | Duration | Notes |
|---|---|---|
| **Sprint** (4–5 h/day) | ~5–6 weeks | Aggressive but doable if this is your full focus |
| **Serious** (2–3 h/day) | ~9–11 weeks | The sweet spot. Enough sleep between sessions to consolidate |
| **Steady** (1 h/day) | ~5 months | Fine, but keep the project moving weekly or you'll lose context |

Modules 1–6 are the foundation and deserve the most time. Modules 7–11 go faster because you'll have the mental model by then.

---

## Table of contents

### Module 1 — Java & the JVM, only what Spring needs
> Skip nothing here. Most people who find Spring "magical" are actually just missing Java's annotation, reflection, and proxy model.

| # | Lesson | What you'll be able to do |
|---|---|---|
| 01 | [The JVM, the classpath & build tools](01-java-for-spring/01-jvm-build-and-classpath.md) | Explain what a JAR is, how the classpath resolves, and read a Maven dependency tree |
| 02 | [Java essentials for Spring](01-java-for-spring/02-java-essentials.md) | Write idiomatic Java: interfaces, generics, collections, exceptions, records |
| 03 | [Modern Java: lambdas, streams, Optional](01-java-for-spring/03-java-modern-features.md) | Read and write the functional Java that Spring APIs are built from |
| 04 | [Annotations, reflection & proxies](01-java-for-spring/04-annotations-reflection-proxies.md) | **The most important lesson in the course.** Explain Spring's "magic" mechanically |

### Module 2 — Spring Core: the container
| # | Lesson | What you'll be able to do |
|---|---|---|
| 05 | [Why Spring exists](02-spring-core/05-why-spring-exists.md) | Argue for/against Spring from first principles, not slogans |
| 06 | [The IoC container & beans](02-spring-core/06-ioc-container-and-beans.md) | Describe the full startup sequence of a Spring app |
| 07 | [Dependency injection, deeply](02-spring-core/07-dependency-injection-deep.md) | Resolve any ambiguous-bean or circular-dependency error instantly |
| 08 | [Scopes, lifecycle & conditional beans](02-spring-core/08-scopes-lifecycle-conditional.md) | Control exactly when and whether a bean exists |

### Module 3 — AOP: the other half of the magic
| # | Lesson | What you'll be able to do |
|---|---|---|
| 09 | [AOP concepts & how proxies really work](03-aop/09-aop-and-proxies.md) | Explain the self-invocation bug that breaks `@Transactional` and `@Cacheable` |
| 10 | [AOP in practice](03-aop/10-aop-in-practice.md) | Write aspects for auditing, timing and retries without abusing them |

### Module 4 — Spring Boot itself
| # | Lesson | What you'll be able to do |
|---|---|---|
| 11 | [Auto-configuration demystified](04-spring-boot/11-autoconfiguration.md) | Trace *why* any bean exists in your app, and write your own starter |
| 12 | [Configuration, profiles & secrets](04-spring-boot/12-configuration-and-profiles.md) | Recite property precedence and configure an app for 4 environments safely |
| 13 | [Project structure like a senior engineer](04-spring-boot/13-project-structure.md) | Defend a package layout in a design review |

### Module 5 — Web & REST APIs
| # | Lesson | What you'll be able to do |
|---|---|---|
| 14 | [Servlets, DispatcherServlet & the request lifecycle](05-web/14-servlet-and-dispatcherservlet.md) | Narrate an HTTP request's path through Spring, thread by thread |
| 15 | [Controllers, mapping & serialization](05-web/15-controllers-and-mapping.md) | Handle any request shape; control JSON precisely |
| 16 | [API design: DTOs, validation & error handling](05-web/16-api-design-dto-validation-errors.md) | Design an API a senior engineer would approve |
| 17 | [OpenAPI, CORS, filters & interceptors](05-web/17-openapi-cors-filters-interceptors.md) | Add cross-cutting HTTP concerns in the right layer |
| 18 | [MVC vs WebFlux vs virtual threads](05-web/18-mvc-vs-webflux-virtual-threads.md) | Answer "should we go reactive?" with real reasoning |

### Module 6 — Data & persistence (the hardest module — go slow)
| # | Lesson | What you'll be able to do |
|---|---|---|
| 19 | [JDBC, DataSources & connection pooling](06-data/19-jdbc-datasource-pooling.md) | Size a Hikari pool and explain pool exhaustion |
| 20 | [JPA & Hibernate fundamentals](06-data/20-jpa-hibernate-fundamentals.md) | Explain the persistence context — the concept most "JPA users" can't |
| 21 | [Entity mapping & relationships](06-data/21-entity-mapping-and-relationships.md) | Model any schema correctly, including the traps |
| 22 | [Spring Data JPA repositories](06-data/22-spring-data-jpa-repositories.md) | Query efficiently: derived, JPQL, native, projections, Specifications |
| 23 | [Transactions deep dive](06-data/23-transactions-deep-dive.md) | Reason about propagation, isolation, rollback and locking under pressure |
| 24 | [Performance, N+1 & migrations](06-data/24-performance-n1-and-migrations.md) | Find and kill the N+1 problem; run migrations safely in production |

### Module 7 — Security
| # | Lesson | What you'll be able to do |
|---|---|---|
| 25 | [Spring Security architecture](07-security/25-security-architecture.md) | Draw the filter chain from memory and debug a 403 |
| 26 | [JWT & stateless authentication](07-security/26-jwt-and-stateless-auth.md) | Implement access/refresh tokens with rotation and revocation |
| 27 | [OAuth2, OIDC & social login](07-security/27-oauth2-oidc-social-login.md) | Explain the auth code + PKCE flow and wire up Google/GitHub login |
| 28 | [Authorization & hardening](07-security/28-authorization-and-hardening.md) | Build multi-tenant permissions; defend against the OWASP top hits |

### Module 8 — Testing (this is what separates seniors from juniors)
| # | Lesson | What you'll be able to do |
|---|---|---|
| 29 | [The testing pyramid & unit tests](08-testing/29-testing-pyramid-and-unit.md) | Write fast tests that don't break on every refactor |
| 30 | [Spring integration testing & Testcontainers](08-testing/30-spring-integration-testing.md) | Test against real Postgres in CI, with slices for speed |
| 31 | [Advanced testing & CI strategy](08-testing/31-advanced-testing-strategy.md) | Contract tests, ArchUnit, flake elimination |

### Module 9 — Production concerns
| # | Lesson | What you'll be able to do |
|---|---|---|
| 32 | [Async, scheduling & events](09-production/32-async-scheduling-events.md) | Run background work safely across multiple instances |
| 33 | [Caching done right](09-production/33-caching.md) | Use Redis without serving stale or corrupt data |
| 34 | [Resilience & outbound integration](09-production/34-resilience-and-integration.md) | Timeouts, retries, circuit breakers, idempotency |
| 35 | [Observability](09-production/35-observability.md) | Ship metrics, structured logs and traces; define SLOs |
| 36 | [Performance & JVM tuning](09-production/36-performance-and-jvm-tuning.md) | Diagnose latency, memory and GC problems |

### Module 10 — Messaging & microservices
| # | Lesson | What you'll be able to do |
|---|---|---|
| 37 | [Messaging with Kafka](10-microservices/37-messaging-kafka.md) | Build idempotent consumers, handle DLQs, avoid duplicate side effects |
| 38 | [Microservices fundamentals](10-microservices/38-microservices-fundamentals.md) | Split a monolith defensibly; handle distributed data with sagas |
| 39 | [Gateway & cross-cutting concerns](10-microservices/39-gateway-and-cross-cutting.md) | Auth at the edge, tracing across services, contract safety |

### Module 11 — Shipping it
| # | Lesson | What you'll be able to do |
|---|---|---|
| 40 | [Docker, CI/CD & deployment](11-deployment/40-docker-cicd-deployment.md) | Containerize, automate and deploy Vigil to a real URL |

### Module 12 — The project
| # | Lesson | What you'll be able to do |
|---|---|---|
| 41 | [Vigil blueprint](12-project-vigil/41-vigil-blueprint.md) | Understand the whole system: domain, ERD, API, architecture |
| 42 | [Vigil build log](12-project-vigil/42-vigil-build-log.md) | Phase-by-phase build instructions, mapped to lessons |
| 43 | [Polish, host & present it](12-project-vigil/43-polish-host-and-present.md) | Turn the project into resume and interview leverage |

### Module 13 — Interview mastery
| # | Lesson | What you'll be able to do |
|---|---|---|
| 44 | [Rapid-fire master Q&A](13-interview/44-rapid-fire-master-qa.md) | Revise the entire course in 90 minutes |
| 45 | [System design & senior signals](13-interview/45-system-design-and-senior-signals.md) | Handle design rounds and "tell me about a time" with Spring depth |

---

## The 12 things that make someone sound senior

Bookmark this. If you can speak fluently on all twelve, you are, functionally, an expert. Every one of them is covered in this course.

1. **Why constructor injection** — and what field injection hides (Lesson 07)
2. **How proxies create the self-invocation trap** — the root cause of most "my annotation doesn't work" bugs (Lesson 09)
3. **How auto-configuration decides** — `@Conditional` + classpath, not magic (Lesson 11)
4. **The persistence context** — why an entity changes the DB without you calling `save()` (Lesson 20)
5. **The N+1 problem** — how to detect it before production does (Lesson 24)
6. **`@Transactional` propagation & rollback rules** — including why checked exceptions don't roll back (Lesson 23)
7. **Optimistic vs pessimistic locking** — and which one your feature actually needs (Lesson 23)
8. **The Spring Security filter chain** — the ability to debug a 401 vs 403 in under a minute (Lesson 25)
9. **Statelessness and token revocation** — the honest trade-offs of JWT (Lesson 26)
10. **Idempotency** — why every retry mechanism requires it (Lesson 34)
11. **Thread pools, blocking and virtual threads** — where your throughput actually goes (Lessons 18, 36)
12. **What you'd do differently** — the answer that signals real experience (Lesson 45)

---

## Progress tracker

Copy this into a `PROGRESS.md` and tick as you go. Add the date — seeing pace keeps you honest.

```
[ ] M1 Java for Spring        01 [ ]  02 [ ]  03 [ ]  04 [ ]
[ ] M2 Spring Core            05 [ ]  06 [ ]  07 [ ]  08 [ ]
[ ] M3 AOP                    09 [ ]  10 [ ]
[ ] M4 Spring Boot            11 [ ]  12 [ ]  13 [ ]
[ ] M5 Web & REST             14 [ ]  15 [ ]  16 [ ]  17 [ ]  18 [ ]
[ ] M6 Data                   19 [ ]  20 [ ]  21 [ ]  22 [ ]  23 [ ]  24 [ ]
[ ] M7 Security               25 [ ]  26 [ ]  27 [ ]  28 [ ]
[ ] M8 Testing                29 [ ]  30 [ ]  31 [ ]
[ ] M9 Production             32 [ ]  33 [ ]  34 [ ]  35 [ ]  36 [ ]
[ ] M10 Microservices         37 [ ]  38 [ ]  39 [ ]
[ ] M11 Deployment            40 [ ]
[ ] M12 Project               41 [ ]  42 [ ]  43 [ ]
[ ] M13 Interview             44 [ ]  45 [ ]
```

---

## One promise, and one warning

**The promise:** if you actually do the loop — read, self-test, type, break, explain out loud — you will not be "someone who knows Spring Boot." You'll be someone who can explain *why Spring is the way it is*, which is a much rarer and more valuable thing.

**The warning:** you cannot read your way to this. The people who fail at this are not the ones who lack intelligence; they're the ones who mistake *understanding a lesson while reading it* for *being able to produce it under pressure*. Close the file and explain it out loud. That discomfort is the learning.

Start here → **[Lesson 01: The JVM, the classpath & build tools](01-java-for-spring/01-jvm-build-and-classpath.md)**

---

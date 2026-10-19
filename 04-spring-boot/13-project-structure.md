# Lesson 13 — Project structure like a senior engineer

> **Why this lesson exists:** Structure is the most visible signal of engineering maturity in a codebase. An interviewer glancing at your GitHub decides in ~10 seconds whether you look senior, and package layout is most of that judgment. More importantly, "how would you organize this?" is a standard design-round question, and the good answer is a *reasoned* one, not a memorized diagram.

**Time:** ~50 minutes · **Prereq:** Module 2, Lesson 11

---

## 1. The idea in one sentence

> **Structure is only real when the compiler enforces it: package-by-feature plus package-private classes gives you module boundaries Java will defend, while package-by-layer gives you a diagram with nothing behind it.**

Everything else here is a judgement about how much enforcement a given feature is worth — package-private inside a feature for most code, Maven modules when a dependency rule must be unbreakable, hexagonal only where the business rules are rich enough to repay the mapping cost.

---

## 2. Layered architecture — the baseline everyone must know

```
┌───────────────────────────────────────────────────────────┐
│  PRESENTATION      @RestController                        │
│  · HTTP concerns only: status codes, headers, paths       │
│  · DTO ↔ domain mapping                                   │
│  · input validation (@Valid)                              │
│  · NO business logic. NO repository calls.                │
├───────────────────────────────────────────────────────────┤
│  SERVICE / APPLICATION      @Service                      │
│  · business rules, orchestration, use cases               │
│  · transaction boundaries (@Transactional lives HERE)     │
│  · knows nothing about HTTP                                │
├───────────────────────────────────────────────────────────┤
│  PERSISTENCE      @Repository                              │
│  · queries and persistence only                            │
│  · no business rules                                       │
├───────────────────────────────────────────────────────────┤
│  DOMAIN      entities, value objects, domain services     │
│  · the model; ideally the least framework-dependent layer │
└───────────────────────────────────────────────────────────┘

Dependencies point DOWNWARD only. Never upward, never skipping in a way that
lets HTTP concerns reach the database.
```

**The three rules that carry all the value:**

1. **`@Transactional` belongs on the service layer.** Not the controller (the transaction would span HTTP concerns and view rendering), not the repository (each method would be its own transaction, so you couldn't compose two writes atomically).
2. **Controllers must not touch repositories.** The moment they do, business logic starts living in controllers, and it becomes unreachable from any other entry point (a scheduled job, a message consumer, a CLI).
3. **Entities must not leak out of the service layer.** Returning JPA entities from controllers causes lazy-loading serialization explosions, leaks your schema into your API contract, and creates accidental breaking changes when you rename a column. (Lesson 16 covers this properly.)

**The most common real violation:**
```java
// ❌ Business logic in a controller. Now the scheduler can't reuse it.
@PostMapping("/monitors")
public ResponseEntity<?> create(@RequestBody CreateMonitorRequest req) {
    if (monitorRepo.countByOrgId(req.orgId()) >= 10) {
        return ResponseEntity.status(402).body("quota exceeded");
    }
    Monitor m = new Monitor(req.name(), req.url());
    m.setStatus(MonitorStatus.PAUSED);
    return ResponseEntity.ok(monitorRepo.save(m));    // ...and it leaks the entity
}
```

The test I use: **could a scheduled job, a Kafka consumer, and an HTTP request all trigger this use case without duplicating logic?** If not, the logic is in the wrong layer.

---

## 3. Package-by-layer vs package-by-feature

### Package-by-layer (what tutorials teach)

```
com.vigil
├── controller
│   ├── MonitorController.java
│   ├── IncidentController.java
│   └── NotificationController.java
├── service
│   ├── MonitorService.java
│   ├── IncidentService.java
│   └── NotificationService.java
├── repository
│   ├── MonitorRepository.java
│   └── IncidentRepository.java
├── entity
├── dto
└── config
```

**Why it feels right:** it mirrors the architecture diagram, and it's what every tutorial shows.

**Why it's wrong at scale:**
- **Every feature is smeared across every package.** Adding "SSL expiry monitoring" touches 6 directories. A code review diff is unreadable.
- **Everything must be `public`.** `MonitorService` is in `service`, `MonitorRepository` is in `repository` — different packages, so the repository must be `public`, so *any* class in the app can use it. **You have no enforceable boundaries at all.**
- **It doesn't scale.** `service/` with 80 files tells you nothing about the system.
- **It hides coupling.** You cannot tell by looking whether the incident code depends on the notification code.

### Package-by-feature (what senior engineers do)

```
com.vigil
├── VigilApplication.java
│
├── monitor                              ← a feature module
│   ├── MonitorController.java           public
│   ├── MonitorService.java              public — the module's API
│   ├── MonitorRepository.java           PACKAGE-PRIVATE ✅
│   ├── Monitor.java                     PACKAGE-PRIVATE ✅ (entity stays inside)
│   ├── MonitorStatus.java
│   ├── MonitorMapper.java               package-private
│   └── dto
│       ├── CreateMonitorRequest.java    public (part of the API)
│       └── MonitorResponse.java         public
│
├── incident
│   ├── IncidentController.java
│   ├── IncidentService.java
│   ├── IncidentRepository.java          package-private
│   └── Incident.java                    package-private
│
├── check                                ← the check execution engine
│   ├── CheckScheduler.java
│   ├── CheckDispatcher.java
│   ├── executor
│   │   ├── CheckExecutor.java           public interface (extension point)
│   │   ├── HttpCheckExecutor.java       package-private
│   │   ├── TcpCheckExecutor.java        package-private
│   │   └── SslCheckExecutor.java        package-private
│   └── CheckResult.java
│
├── notification
│   ├── NotificationChannel.java         public interface
│   ├── NotificationDispatcher.java      public
│   ├── SlackChannel.java                package-private
│   ├── EmailChannel.java                package-private
│   └── WebhookChannel.java              package-private
│
├── statuspage
├── organization                         ← tenants, users, memberships
├── billing
│
├── common                               ← genuinely shared, no feature knowledge
│   ├── exception
│   │   ├── GlobalExceptionHandler.java
│   │   ├── NotFoundException.java
│   │   └── ApiError.java
│   ├── audit
│   ├── security                          (SecurityContext helpers)
│   └── util
│
└── config                               ← infrastructure wiring only
    ├── SecurityConfig.java
    ├── JacksonConfig.java
    ├── AsyncConfig.java
    ├── CacheConfig.java
    └── OpenApiConfig.java
```

**Why this is better, concretely:**

1. **High cohesion.** Everything about monitors is in one directory. A new team member finds it instantly.
2. **Real encapsulation via package-private.** `MonitorRepository` is package-private, so the *compiler* prevents `IncidentService` from querying monitors directly — it must go through `MonitorService`. **This is the biggest win and almost nobody uses it.** Java's package-private access is a free, compiler-enforced module boundary, and package-by-layer throws it away.
3. **A visible dependency graph.** Imports at the top of each file show exactly which features depend on which. Coupling becomes reviewable.
4. **Deletable features.** Removing the billing feature is `rm -rf billing/` plus fixing compile errors. Try that with package-by-layer.
5. **Microservice-extraction-ready.** When `check/` needs to scale independently, it's already a self-contained unit. **This is the honest reason Vigil's structure matters** — the eventual service split follows existing package boundaries rather than requiring archaeology.

**Say this in interviews:** *"I organize by feature, not by layer, because it gives me cohesion and lets me use package-private visibility to enforce module boundaries at compile time. Package-by-layer forces everything public, so you have layers on a diagram but no real boundaries in the code."* That's a genuinely senior answer.

### The nuance (be fair)

Package-by-layer is fine for a small app (< ~20 classes) and it's what most tutorials and many existing codebases use. **Consistency with an existing codebase beats being right in isolation** — don't unilaterally restructure someone else's project. And hybrid layouts (feature packages with a `layer` sub-package inside for large features) are common and reasonable.

---

## 4. Hexagonal / ports-and-adapters (know it, apply it selectively)

For domain-heavy systems, the ambition is that **the domain has zero framework dependencies**:

```
                    ┌─────────────────────────────────┐
   INBOUND          │                                 │        OUTBOUND
   ADAPTERS         │           DOMAIN                │        ADAPTERS
                    │  · entities, value objects      │
  REST controller ──┤  · business rules               ├── JPA repository impl
  Kafka consumer  ──┤  · use-case services            ├── Slack HTTP client
  Scheduler       ──┤                                 ├── S3 storage
  CLI             ──┤  NO Spring. NO JPA. NO Jackson. ├── Kafka producer
                    │                                 │
                    │  defines PORTS (interfaces)     │
                    └─────────────────────────────────┘
                      Adapters depend on the domain. Never the reverse.
```

```java
// ── domain (pure Java) ────────────────────────────────────────────
package com.vigil.monitor.domain;

public record Monitor(MonitorId id, String name, Url url, Interval interval, MonitorStatus status) {
    public Monitor pause() {
        if (status == MonitorStatus.PAUSED) throw new AlreadyPausedException(id);
        return new Monitor(id, name, url, interval, MonitorStatus.PAUSED);
    }
}

// A PORT — the domain declares what it needs, in its own vocabulary
public interface MonitorRepositoryPort {
    Optional<Monitor> findById(MonitorId id);
    Monitor save(Monitor monitor);
}

// ── adapter (Spring/JPA lives here) ───────────────────────────────
package com.vigil.monitor.adapter.persistence;

@Repository
@RequiredArgsConstructor
class JpaMonitorRepositoryAdapter implements MonitorRepositoryPort {
    private final MonitorJpaRepository jpa;          // Spring Data
    private final MonitorEntityMapper mapper;

    public Optional<Monitor> findById(MonitorId id) {
        return jpa.findById(id.value()).map(mapper::toDomain);
    }
}
```

**What you gain:** the domain is testable with zero framework, business rules are expressible in domain language, and swapping persistence or transport is genuinely local. For a system whose *complexity is in the rules* — insurance underwriting, trading, tax, healthcare eligibility — this pays for itself.

**What you pay:** two models (domain + JPA entity) and a mapper for every aggregate. That's real, ongoing work. For CRUD-shaped features it's pure ceremony — you write 3 files and a mapper to save 4 columns.

**The mature position, and the one to give in an interview:**

> *"Hexagonal is a tool for protecting complex domain logic, not a default. I'd apply it where the business rules are rich enough to justify the mapping cost, and keep simple CRUD features as a plain layered slice. Mixing both in one codebase is normal — the mistake is applying either one dogmatically."*

Interviewers hear "clean architecture always" a lot. A calibrated answer with a cost/benefit stands out sharply.

---

## 5. Multi-module Maven — the pre-microservice step

When a monolith gets large, split it into Maven modules to make dependency rules **compiler-enforced**:

```
vigil/
├── pom.xml                     (packaging: pom — the parent)
├── vigil-domain/               no Spring dependency at all
├── vigil-application/          use cases; depends on domain
├── vigil-persistence/          JPA adapters; depends on domain + application
├── vigil-web/                  controllers; depends on application
├── vigil-notification/         channel adapters
└── vigil-bootstrap/            @SpringBootApplication; depends on everything
```

```xml
<!-- parent pom.xml -->
<packaging>pom</packaging>
<modules>
    <module>vigil-domain</module>
    <module>vigil-application</module>
    <module>vigil-persistence</module>
    <module>vigil-web</module>
    <module>vigil-bootstrap</module>
</modules>
```

**The point:** if `vigil-domain`'s POM doesn't declare Spring, then `import org.springframework...` in the domain **won't compile**. The build enforces the architecture, so it can't erode over time under deadline pressure. That's a much stronger guarantee than a convention in a wiki.

**Costs:** slower builds, more POMs, IDE friction, and awkwardness for cross-cutting changes. **When it's worth it:** several teams sharing a codebase, an architecture you must protect, or preparing a service extraction.

**The lighter-weight alternative worth knowing: ArchUnit.** Enforce the same rules as a *test* in a single-module project:

```java
@Test
void domainMustNotDependOnSpring() {
    noClasses().that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAnyPackage("org.springframework..")
            .check(new ClassFileImporter().importPackages("com.vigil"));
}

@Test
void controllersMustNotTouchRepositories() {
    noClasses().that().haveSimpleNameEndingWith("Controller")
            .should().dependOnClassesThat().haveSimpleNameEndingWith("Repository")
            .check(new ClassFileImporter().importPackages("com.vigil"));
}
```

**This is one of the highest-value/lowest-cost things you can add to a codebase**, and mentioning it in a design discussion is a strong signal: you get enforcement without the build complexity. Lesson 31 covers it properly, and **Vigil will have these tests.**

> **Also worth knowing: Spring Modulith.** A Spring project specifically for enforcing module boundaries *within* a monolith — verifying allowed dependencies, documenting the module structure, and supporting module-scoped integration tests and event-based decoupling. It's the framework's own answer to "modular monolith," and naming it shows you're current.

---

## 6. Naming conventions that signal experience

| Element | Convention | Avoid |
|---|---|---|
| Entity | `Monitor`, `Incident` | `MonitorEntity`, `MonitorDAO`, `TblMonitor` |
| Repository | `MonitorRepository` | `MonitorDAO`, `MonitorRepo` |
| Service | `MonitorService`; for CQRS-ish designs, use-case names like `CreateMonitorUseCase` | `MonitorManager`, `MonitorHelper`, `MonitorUtil` |
| Controller | `MonitorController` | `MonitorAPI`, `MonitorResource` (fine in JAX-RS, off-convention in Spring) |
| Request DTO | `CreateMonitorRequest`, `UpdateMonitorRequest` | `MonitorDTO` (which direction?) |
| Response DTO | `MonitorResponse`, `MonitorSummaryResponse` | `MonitorVO`, `MonitorBean` |
| Mapper | `MonitorMapper` | `MonitorConverter`, `MonitorTransformer` (fine, just be consistent) |
| Config | `SecurityConfig`, `AsyncConfig` | `Configuration1`, `Beans` |
| Exception | `MonitorNotFoundException` | `MonitorException` (too vague to handle) |
| Event | `MonitorWentDownEvent` — **past tense, always** | `MonitorDownEvent`, `MonitorDown` |

**The two that reveal the most about a developer:**

**"Manager"/"Helper"/"Util" in a class name is a smell.** It means the author couldn't name the responsibility, which usually means there's more than one. `MonitorManager` — what does it *do*? `MonitorScheduler`, `MonitorValidator`, `UptimeCalculator` all say something.

**Events must be past tense**, because an event is a *fact that already happened* and cannot be rejected. `MonitorWentDownEvent` (fact) vs `CheckMonitorCommand` (a request that might be refused). Commands are imperative, events are past-tense. Getting this distinction right in a design conversation is a genuine seniority marker.

---

## 7. The `config` package — what belongs there

**Belongs:**
```
config/
├── SecurityConfig.java        SecurityFilterChain
├── JacksonConfig.java         ObjectMapper customization
├── AsyncConfig.java           TaskExecutor beans
├── CacheConfig.java           CacheManager, Redis serializers
├── OpenApiConfig.java         API docs metadata
├── WebConfig.java             CORS, interceptors, converters
└── ObservabilityConfig.java   metrics tags, tracing
```

**Doesn't belong:** anything with business rules. A `config` class deciding "free plans get 10 monitors" is business logic wearing a config costume — that belongs in the feature, ideally as a `@ConfigurationProperties`-driven policy object inside `billing/`.

**Also:** put feature-specific `@ConfigurationProperties` **in the feature package**, not in `config`. `CheckProperties` belongs in `check/`, next to what uses it.

---

## 8. Test structure — mirror the source, exactly

```
src/test/java/com/vigil/
├── monitor
│   ├── MonitorServiceTest.java              unit, mocked deps, fast
│   ├── MonitorControllerTest.java           @WebMvcTest slice
│   └── MonitorRepositoryTest.java           @DataJpaTest + Testcontainers
├── check
│   └── CheckDispatcherTest.java
├── architecture
│   └── ArchitectureTest.java                ← ArchUnit rules
└── support
    ├── AbstractIntegrationTest.java         shared Testcontainers setup
    ├── TestDataFactory.java                 object builders
    └── VigilApplicationTests.java           context-loads smoke test
```

`src/test/resources/application-test.yml` holds test config.

**Rules:** mirror the main package structure exactly (so `Ctrl+Shift+T` navigation works and package-private classes are testable), name tests `<ClassUnderTest>Test` for unit and `<...>IT`/`<...>IntegrationTest` for integration (Maven's Surefire runs `*Test`, Failsafe runs `*IT` — so you can run fast tests separately in CI). Module 8 covers this fully.

---

## 9. Vigil's actual structure

This is what you'll build, and it's designed so the eventual microservice split falls out of the existing packages:

```
com.vigil
├── VigilApplication.java
├── organization        orgs, users, memberships, invitations, API keys
├── monitor             monitor CRUD, config, ownership
├── check               the execution engine  → becomes vigil-checker service
│   ├── scheduler
│   ├── executor        HTTP / TCP / SSL / heartbeat executors
│   └── result          storage + aggregation of check results
├── incident            incident lifecycle, escalation policies, state machine
├── notification        channels + dispatch  → becomes vigil-notifier service
├── statuspage          public status pages, cached, rate-limited
├── report              uptime %, SLA, p95 latency
├── billing             plans, quotas, limits
├── common              exceptions, audit, security helpers, pagination
└── config              infrastructure wiring
```

**Why the boundaries are where they are** (this is the answer to "defend your architecture"):
- **`check` is separate from `monitor`** because monitor *configuration* changes rarely while *execution* runs continuously at high volume. Different data-access patterns, different scaling needs, different failure modes. This is the split that makes the microservice extraction honest rather than decorative.
- **`incident` is separate from `check`** because incident state is a business workflow (open → acknowledged → resolved, escalation, notification), whereas a check result is an immutable measurement. Mixing them would put workflow state in a hot write path.
- **`notification` is separate** because it talks to unreliable third parties and must be retried and rate-limited independently of everything else.
- **`common` contains no feature knowledge.** If `common` ever imports from `monitor`, the boundary has failed and ArchUnit will catch it.

---

## 10. Build & break

**1. Restructure your project.** Create the Vigil package skeleton from §8 (empty packages with a `package-info.java` are fine). Move your playground classes into `playground/` so they don't pollute it.

**2. Implement one vertical slice properly, with real encapsulation:**
```
monitor/
├── MonitorController.java        public
├── MonitorService.java          public
├── MonitorRepository.java       PACKAGE-PRIVATE (no `public` keyword)
├── Monitor.java                 PACKAGE-PRIVATE
├── MonitorStatus.java           public (used in DTOs)
├── MonitorMapper.java           package-private
└── dto/
    ├── CreateMonitorRequest.java
    └── MonitorResponse.java
```
Then, from a class in `com.vigil.incident`, try to `import com.vigil.monitor.MonitorRepository`. **It won't compile.** That compiler error is your architecture being enforced for free. Sit with how good that is compared to a wiki page nobody reads.

**3. Add ArchUnit and write four rules.**
```xml
<dependency>
    <groupId>com.tngtech.archunit</groupId>
    <artifactId>archunit-junit5</artifactId>
    <version>1.3.0</version>
    <scope>test</scope>
</dependency>
```
Rules to write:
- Controllers must not depend on repositories
- Classes named `*Repository` must live in a package matching `..vigil..`
- `common` must not depend on any feature package
- Nothing may depend on `..playground..`

Then **deliberately violate each rule** and watch the test fail with a precise message. **This is the exercise that makes architecture feel real** rather than aspirational.

**4. Write the layering test as documentation:**
```java
@Test
void layerDependenciesAreRespected() {
    layeredArchitecture().consideringOnlyDependenciesInLayers()
        .layer("Controller").definedBy("..vigil.*.*Controller")
        .layer("Service").definedBy("..vigil.*.*Service")
        .layer("Repository").definedBy("..vigil.*.*Repository")
        .whereLayer("Controller").mayNotBeAccessedByAnyLayer()
        .whereLayer("Service").mayOnlyBeAccessedByLayers("Controller")
        .whereLayer("Repository").mayOnlyBeAccessedByLayers("Service")
        .check(new ClassFileImporter().importPackages("com.vigil"));
}
```
**An executable architecture diagram.** Put this in your README when you publish Vigil — it's a genuine differentiator.

**5. Write a one-page `ARCHITECTURE.md`.** Package structure, the dependency rule, why `check` is separate from `monitor`, and what you'd extract first if you needed to scale. **Interviewers read this file.** Having one is unusual; having a good one is memorable.

### Explain out loud (60s)

1. Why package-by-feature beats package-by-layer, in terms of what the compiler enforces.
2. The three layering rules, and the specific failure each prevents.
3. The "three entry points" test, and a use case that fails it.
4. What hexagonal buys you, what it costs, and where you'd actually apply it.
5. When you'd escalate from package-private to a separate Maven module.

---

## 11. Production rules

| Do / Never | Why |
|---|---|
| **Organize by feature, not by layer** | Layer packages force every class `public`, so you get layers on a diagram and no boundary the compiler will defend |
| **Make repositories and entities package-private inside their feature** | It is a free, compiler-enforced module boundary that stops another feature querying your tables directly |
| **Put `@Transactional` on the service layer only** | On a controller it would span HTTP concerns; on a repository every method is its own transaction, so two writes can never be atomic |
| **Never call a repository from a controller** | Business logic then lives where only HTTP can reach it — a scheduled job or a Kafka consumer has to duplicate it |
| **Never return a JPA entity from a controller** | It leaks your schema into your API contract, turns a column rename into a breaking change, and invites lazy-loading serialization explosions |
| **Apply the "three entry points" test to every use case** | If an HTTP request, a scheduled job and a message consumer can't all trigger it without duplication, the logic sits in the wrong layer |
| **Apply hexagonal selectively, never dogmatically** | Two models and a mapper per aggregate is real ongoing cost — worth it for rich domain rules, pure ceremony for four-column CRUD |
| **Reach for Maven modules when a dependency rule must be unbreakable** | Module boundaries fail the build; a convention in a wiki fails silently over the following year |
| **Match an existing codebase's structure rather than unilaterally improving it** | Consistency beats being right in isolation, and a half-migrated layout is worse than either one done consistently |
| **Mirror the source tree exactly in `src/test/java`** | It keeps package-private tests possible and makes the missing test for a class obvious at a glance |

---

## 12. Interview traps

**Q1. How do you structure a Spring Boot project?**
Package-by-feature, not package-by-layer: each feature owns its controller, service, repository, entity and DTOs. That gives cohesion and lets me make repositories and entities **package-private**, so the compiler enforces that other features go through the public service API. Package-by-layer forces everything public, so you have layers on paper but no real boundaries. `common` and `config` hold genuinely shared infrastructure with no feature knowledge.

**Q2. Where does `@Transactional` belong, and why not the other layers?**
The service layer. Not the controller (a transaction would span HTTP/serialization concerns and stay open too long). Not the repository (each method becomes its own transaction, so you can't compose two writes atomically). The service layer is where a use case — and therefore a unit of work — is defined.

**Q3. Can a controller call a repository directly?**
It compiles, but no. Business logic migrates into controllers and becomes unreachable from other entry points (schedulers, message consumers, CLI). My test: could an HTTP request, a scheduled job, and a Kafka consumer all trigger this use case without duplicating logic?

**Q4. Should you return JPA entities from controllers?**
No. It leaks your schema into your API contract (so a column rename is a breaking change), triggers lazy-loading serialization failures or accidental N+1, and can expose fields you never meant to publish. Use response DTOs.

**Q5. What is hexagonal architecture, and when would you use it?**
Domain at the centre with no framework dependencies, declaring ports (interfaces); adapters for REST/JPA/messaging implement them and depend inward. Use it where domain complexity justifies the double model and mapping cost. For CRUD features it's ceremony — mixing approaches within one codebase is normal and correct.

**Q6. How do you stop architecture eroding over time?**
Three tiers: package-private visibility (free, compiler-enforced), ArchUnit tests (cheap, precise, runs in CI), and multi-module Maven or Spring Modulith (strongest, at the cost of build complexity). Start with the first two — they cover most of the value.

**Q7. When would you split into multiple Maven modules?**
When multiple teams share the codebase, when an architectural boundary must be guaranteed rather than encouraged, or when preparing a service extraction. Costs: slower builds, more POMs, cross-cutting changes get awkward. ArchUnit is often the better first step.

**Q8. Why is `MonitorManager` a bad class name?**
"Manager"/"Helper"/"Util" means the author couldn't name the responsibility — usually because there's more than one. Name what it does: `MonitorScheduler`, `MonitorValidator`, `UptimeCalculator`.

**Q9. Why must domain events be past tense?**
An event is a fact that already happened and cannot be rejected; a command is a request that can be refused. `MonitorWentDownEvent` vs `PauseMonitorCommand`. The tense encodes whether the receiver may say no — which is exactly the distinction that keeps event-driven systems comprehensible.

**Q10. How would you organize this monolith so it can later become microservices?**
Feature packages with explicit, enforced boundaries; communication between features through public service interfaces or domain events rather than shared repositories; no cross-feature entity relationships or shared tables. Then extraction follows the package boundary instead of requiring archaeology. Spring Modulith exists specifically to support this.

---

## 13. Retention check

1. Give the strongest argument for package-by-feature — the one about the compiler.
2. Why not `@Transactional` on a controller? Why not on a repository?
3. What's the cost of hexagonal architecture, and when is it not worth paying?
4. Name three mechanisms for preventing architectural erosion, in increasing cost order.
5. Why is `check` a separate package from `monitor` in Vigil?
6. Give the naming rule for events, and explain what the tense communicates.

---

## Module 4 complete ✅

You understand Spring Boot itself now: how auto-configuration decides what exists, how configuration flows in from the environment, and how to organize the result so it survives contact with a growing team.

**Next:** [Lesson 14 — Servlets, DispatcherServlet & the request lifecycle](../05-web/14-servlet-and-dispatcherservlet.md) — following one HTTP request all the way through Spring, thread by thread.

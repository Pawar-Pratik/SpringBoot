# Lesson 05 — Why Spring exists

> **Why this lesson exists:** "What is dependency injection and why do we need it?" is asked in almost every Spring interview. Most candidates recite "loose coupling" and "testability" without being able to *demonstrate* the pain that motivated it. You're going to feel the pain first, then the solution will be obvious — and you'll be able to argue about it from first principles, including the parts where Spring's critics are right.

**Time:** ~45 minutes · **Prereq:** Module 1

---

## 1. Let's build the pain

Forget Spring. You're writing a Java service that saves a monitor and notifies a Slack channel.

```java
public class MonitorService {
    private final MonitorRepository repo = new MonitorRepository();
    private final SlackNotifier notifier = new SlackNotifier();

    public void createMonitor(String name, String url) {
        Monitor m = repo.save(new Monitor(name, url));
        notifier.send("Monitor created: " + name);
    }
}
```

Perfectly reasonable-looking code. Here's everything wrong with it.

### Problem 1 — It's welded to its dependencies

`MonitorService` decides that notifications go to Slack. Now the requirement changes: enterprise customers want email, some want a webhook, and during tests you want nothing at all.

```java
public class MonitorService {
    private final SlackNotifier slack = new SlackNotifier();
    private final EmailNotifier email = new EmailNotifier();
    private final WebhookNotifier webhook = new WebhookNotifier();

    public void createMonitor(String name, String url, String channelType) {
        Monitor m = repo.save(new Monitor(name, url));
        if ("slack".equals(channelType))        slack.send(...);
        else if ("email".equals(channelType))   email.send(...);
        else if ("webhook".equals(channelType)) webhook.send(...);
    }
}
```

Every new channel means **editing this class**. That violates the Open/Closed Principle, and more practically: a class that changes for reasons unrelated to its purpose becomes a merge-conflict magnet and a bug farm.

### Problem 2 — You cannot test it

```java
@Test
void createsMonitor() {
    var service = new MonitorService();
    service.createMonitor("API", "https://api.example.com");
    // ...and this test just posted a real message to your team's Slack.
    // And tried to open a real database connection.
}
```

There is **no seam**. The `new` keyword inside the class is a hard-coded decision you cannot override from outside. To test `MonitorService`'s logic you must have a live database and a live Slack workspace. So people don't write the test. So the code rots.

**This is the deepest argument for DI, and the one candidates undersell:** `new` inside a class is a *compile-time commitment* to a specific collaborator. DI turns it into a *runtime choice*.

### Problem 3 — Object graph construction becomes your job

Real dependencies have dependencies:

```java
var config       = new AppConfig("application.properties");
var dataSource   = new HikariDataSource(config.getDbUrl(), config.getDbUser(), config.getDbPass());
var txManager    = new TransactionManager(dataSource);
var monitorRepo  = new MonitorRepository(dataSource, txManager);
var httpClient   = HttpClient.newBuilder().connectTimeout(Duration.ofSeconds(5)).build();
var slackClient  = new SlackClient(httpClient, config.getSlackToken());
var slackNotifier= new SlackNotifier(slackClient, config.getSlackChannel());
var auditLog     = new AuditLog(dataSource);
var monitorSvc   = new MonitorService(monitorRepo, slackNotifier, auditLog, txManager);
var incidentSvc  = new IncidentService(monitorRepo, slackNotifier, auditLog, txManager);
// ...and 60 more lines, in the right order, with sharing rules:
// - ONE dataSource shared by everyone (else you get N connection pools)
// - ONE httpClient shared (else you leak thread pools)
// - a NEW request context per request
```

This file is real. It has a name in the industry: **the "God" wiring class**, or `ApplicationBootstrap`. It's 400 lines, everybody's changes collide in it, the ordering is load-bearing and undocumented, and getting a sharing rule wrong (a second `DataSource`, a fresh `HttpClient` per call) causes production resource exhaustion.

### Problem 4 — Cross-cutting concerns get copy-pasted

Now add transactions, logging, metrics, and security to *every* service method:

```java
public void createMonitor(...) {
    long start = System.nanoTime();
    log.info("createMonitor start");
    if (!securityContext.hasRole("ADMIN")) throw new AccessDeniedException();
    Transaction tx = txManager.begin();
    try {
        // ~2 lines of actual business logic, buried
        tx.commit();
    } catch (Exception e) {
        tx.rollback();
        throw e;
    } finally {
        metrics.record("createMonitor", System.nanoTime() - start);
        log.info("createMonitor end");
    }
}
```

**14 lines of plumbing around 2 lines of business value**, repeated in 200 methods. Any inconsistency (a forgotten `rollback`, a missed permission check) is a bug or a vulnerability. This is the problem AOP solves (Module 3).

---

## 2. Step one: invert the control

The fix for Problems 1 and 2 requires no framework whatsoever. Just stop calling `new`:

```java
public interface NotificationChannel {
    void send(String subject, String body);
}

public class MonitorService {
    private final MonitorRepository repo;
    private final NotificationChannel channel;

    public MonitorService(MonitorRepository repo, NotificationChannel channel) {  // ← handed in
        this.repo = repo;
        this.channel = channel;
    }

    public void createMonitor(String name, String url) {
        Monitor m = repo.save(new Monitor(name, url));
        channel.send("Monitor created", name);
    }
}
```

Look at what changed:

- `MonitorService` no longer knows Slack exists. It knows a *contract*.
- Adding an email channel requires **zero** changes here.
- Testing is now trivial:

```java
@Test
void notifiesOnCreate() {
    var fakeRepo = new InMemoryMonitorRepository();
    var recorder = new RecordingChannel();                  // a 5-line test double
    var service = new MonitorService(fakeRepo, recorder);

    service.createMonitor("API", "https://api.example.com");

    assertThat(recorder.sent()).hasSize(1);
}
```

Fast, no network, no database, deterministic.

### The three terms, precisely (interviewers test this distinction)

| Term | Meaning |
|---|---|
| **Inversion of Control (IoC)** | The general principle: a class does not control the creation/lookup of what it uses; something external does. Broader than DI — callbacks, template method, and event loops are also IoC. |
| **Dependency Injection (DI)** | A *specific* IoC technique: dependencies are *supplied from outside* (constructor, setter, field). |
| **Dependency Inversion Principle (DIP)** | A *design* principle (the "D" in SOLID): depend on abstractions, not concretions. High-level policy shouldn't depend on low-level detail. |

**How they fit:** DIP tells you to depend on `NotificationChannel` rather than `SlackNotifier`. DI is how the concrete `SlackNotifier` reaches you. IoC is the umbrella idea that makes it possible.

> **Interview polish:** *"IoC is the principle, DI is one implementation of it, and DIP is the design rule that makes DI worth doing."* That one sentence separates you from candidates who use the three terms interchangeably.

> **Fun fact:** The term "Dependency Injection" was coined by **Martin Fowler in 2004**, precisely because "Inversion of Control" was too vague — everyone's framework claimed IoC, so the word had stopped carrying information. He wanted a name for the specific pattern. Knowing this bit of etymology is a cheap way to show you read beyond tutorials.

---

## 3. Step two: something has to assemble the graph

We've fixed coupling but made Problem 3 *worse* — now every caller must construct everything.

Enter **the container**. Conceptually it is startlingly simple:

> A container is a **registry of objects** plus **rules for creating them**. You describe what you need; it works out the construction order, creates each object once, and hands out references.

```java
// Instead of you writing 400 lines of wiring:
ApplicationContext ctx = SpringApplication.run(VigilApplication.class, args);
// Spring has now created every bean, in dependency order, exactly once.
```

Because the container owns creation, it can also solve Problem 4: at creation time it can **wrap** beans in proxies that add transactions, metrics, security and caching (Lesson 04, Module 3). You get to write:

```java
@Service
public class MonitorService {
    private final MonitorRepository repo;
    private final NotificationChannel channel;

    public MonitorService(MonitorRepository repo, NotificationChannel channel) {
        this.repo = repo;
        this.channel = channel;
    }

    @Transactional
    public void createMonitor(String name, String url) {
        Monitor m = repo.save(new Monitor(name, url));
        channel.send("Monitor created", name);
    }
}
```

**Business logic and nothing else,** with infrastructure declared rather than coded. That's the entire pitch of Spring, and now you've derived it rather than been told it.

---

## 4. What Spring actually is (the map)

People say "Spring" to mean at least three different things. Being precise here is a small but noticeable credibility marker.

```
┌──────────────────────────────────────────────────────────────────────┐
│  SPRING BOOT           opinionated defaults, auto-config, starters,  │
│                        embedded server, executable JAR, Actuator     │
│                        → "make Spring usable without ceremony"       │
├──────────────────────────────────────────────────────────────────────┤
│  SPRING PROJECTS       Spring Data · Spring Security · Spring Cloud  │
│                        Spring Batch · Spring Integration · AMQP/Kafka│
│                        → separate products built on the Framework    │
├──────────────────────────────────────────────────────────────────────┤
│  SPRING FRAMEWORK      spring-core     (IoC container, DI)           │
│  (the foundation)      spring-beans    (bean definitions, factories) │
│                        spring-context  (ApplicationContext, events)  │
│                        spring-aop      (proxies, aspects)            │
│                        spring-tx       (transaction abstraction)     │
│                        spring-web(mvc) (DispatcherServlet, REST)     │
│                        spring-webflux  (reactive stack)              │
│                        spring-jdbc/orm (data access + exception xlat)│
└──────────────────────────────────────────────────────────────────────┘
```

**Spring Boot is not a different framework.** It is Spring Framework plus (a) auto-configuration, (b) starter dependency bundles, (c) an embedded server, (d) production tooling. Every Boot app is a Spring app. Saying "we use Spring Boot, not Spring" reveals a misunderstanding — and it's a question interviewers deliberately ask to see if you know the layering.

### What Boot actually added, in order of impact

| # | Feature | The pain it removed |
|---|---|---|
| 1 | **Auto-configuration** | Hundreds of lines of `@Bean` boilerplate for DataSource, EntityManager, Jackson, Tomcat, security... |
| 2 | **Starters + BOM** | The "which versions of these 40 libraries work together?" research project |
| 3 | **Embedded server + fat JAR** | Installing/administering Tomcat; WAR deployment; `java -jar` instead |
| 4 | **Sensible defaults** | Every project reinventing the same 50 config decisions |
| 5 | **Actuator** | Hand-rolling health checks, metrics endpoints, thread dumps |
| 6 | **`start.spring.io`** | Day-one project setup |

> **Fun fact — the origin of Spring Boot:** In 2012, Mike Youngstrom filed a JIRA issue (SPR-9888) titled *"Improved support for 'containerless' web application architectures"*, arguing Spring's web setup had too much ceremony. Phil Webb and Dave Syer built a prototype; Spring Boot 1.0 shipped in April 2014. Within a few years it was the default way to write Java server software on Earth. A feature request that reshaped an entire ecosystem.

> **Fun fact — the origin of Spring itself:** Rod Johnson published *"Expert One-on-One J2EE Design and Development"* in 2002 with ~30,000 lines of accompanying framework code, arguing that Enterprise JavaBeans (EJB) — the official standard — was so heavyweight it was harming projects. Readers wanted the code more than the book. It was open-sourced as "Spring" (as in, "the spring after the winter of traditional J2EE"). Spring 1.0 landed in March 2004. **Spring is, at its root, a protest against enterprise complexity** — which is a little ironic given how large Spring later became, and a very good thing to say with a smile in an interview.

---

## 5. The honest case against Spring

A true expert can argue *both* sides. Interviewers notice, because it signals judgment rather than fandom. Here are the criticisms, and the fair responses.

**"It's too magical — I can't tell why my app behaves the way it does."**
Legitimate, especially before you understand Module 1's mechanisms. But the magic is *inspectable*: `--debug` prints the full auto-configuration report, `/actuator/beans` lists every bean and its dependencies, `/actuator/conditions` shows exactly which conditions matched. The tooling is there — most developers just never learn it. (You will, in Lesson 11.)

**"Startup is slow and memory usage is high."**
True, and it matters for serverless and rapid autoscaling. Real numbers: a typical Boot app starts in 2–6 seconds and idles at 200–500 MB. That's fine for a long-running service, painful for a Lambda. Responses: lazy initialization, CDS, Spring AOT, GraalVM native images (~50ms startup, ~50MB), or choosing Quarkus/Micronaut — frameworks that do the same job with **compile-time** DI instead of runtime reflection, precisely to fix this. Knowing those alternatives exist and *why* they exist is a strong signal.

**"Runtime errors that should be compile-time errors."**
Fair. Reflective, string-based configuration means a typo in a property key or a missing bean fails at boot. Mitigations: `@ConfigurationProperties` with validation, constructor injection (fails fast and loudly), and Spring's genuinely excellent error messages. Compile-time DI frameworks (Dagger, Micronaut) trade flexibility for compile-time safety — a real trade-off, not a clear win either way.

**"It's enormous."**
Spring Framework + Boot is a lot of surface area, and nobody knows all of it. But you don't need to: the core is small (container + AOP), and the rest is opt-in modules. The counter-question worth asking: what's the alternative for a team of 30 building a system that needs security, transactions, data access, messaging, and observability? Assembling that from ten unrelated libraries has its own — usually larger — cost.

**"DI itself is overrated; just use `new`."**
There's a real school of thought here (particularly in Go and functional communities): explicit construction is simpler, traceable, and compile-checked. It's a genuinely reasonable position for small services. The honest answer: **DI's value scales with the size of the object graph and the number of contributors.** For a 200-line tool, `new` is better. For a 200,000-line system with 40 engineers, manual wiring becomes the bottleneck.

**When you'd genuinely not pick Spring Boot:**
- Serverless functions where cold start dominates → native image, or a lighter framework
- Tiny CLI tools → plain Java, or `picocli`
- Extreme-throughput low-latency systems where you want every layer under your control
- A team with zero Java/JVM experience and no reason to adopt it

**When Spring Boot is obviously right:** long-running services, anything needing transactions + security + data access, teams that value hiring pool and ecosystem, and any organization that already runs the JVM. Which is most of the industry.

---

## 6. Where Spring sits in the industry (context you'll be expected to have)

- **Adoption:** the dominant server-side Java framework, by a wide margin, especially in banking, insurance, telecom, retail, and enterprise SaaS. In many companies "Java backend developer" and "Spring developer" are used interchangeably.
- **Jakarta EE:** the standards-based alternative (formerly Java EE / J2EE — renamed after Oracle transferred it to the Eclipse Foundation and lost the "Java" trademark for it). Spring *implements or uses* several Jakarta specs — Servlet, JPA, Bean Validation, Jakarta Annotations — while rejecting the full EJB programming model. **Boot 3's biggest breaking change was `javax.*` → `jakarta.*`**, which is exactly why Boot 2 → 3 migration was painful. Expect a question on this.
- **Competitors:** Quarkus (Red Hat) and Micronaut both use build-time DI for fast startup and low memory, and both deliberately mimic Spring's annotations to ease migration. That imitation is itself a comment on Spring's dominance.
- **Versions you should be able to place:**

| Version | Era | Headline |
|---|---|---|
| Spring 1.0 | 2004 | XML config, IoC container |
| Spring 2.5 | 2007 | Annotation-driven (`@Component`, `@Autowired`) |
| Spring 3.0 | 2009 | Java config (`@Configuration`, `@Bean`), SpEL |
| Spring 4.0 | 2013 | Java 8, WebSockets |
| Spring Boot 1.0 | 2014 | Auto-config, starters, embedded server |
| Spring 5.0 / Boot 2.0 | 2017/18 | **WebFlux** (reactive), Java 8 baseline, Kotlin |
| Spring 6.0 / **Boot 3.0** | 2022 | **Java 17 baseline, `jakarta.*` migration, AOT/native, Micrometer Observation** |
| Boot 3.2–3.5 | 2023–25 | **Virtual threads**, `RestClient`, JVM checkpoint restore, refinements |
| Spring 7 / **Boot 4.0** | 2025 | Java 17+ baseline, module reorganization, JSpecify null-safety, API modernization |

> **Boot 4 delta:** Boot 4 / Framework 7 continue the direction of travel — more AOT, cleaner module boundaries, standardized null-safety, and deprecating older HTTP clients in favour of `RestClient`/HTTP interfaces. Nothing in this course's mental model changes; verify specific API details against current docs when you adopt it.

---

## 7. Build this

Small but genuinely instructive. In `com.vigil.playground.di`:

**Step 1 — Write the painful version.** `MonitorService` with `new SlackNotifier()` and `new MonitorRepository()` inside it. Have `SlackNotifier.send()` print `"POST https://slack.com/api/chat.postMessage"`.

**Step 2 — Try to unit-test it** so that "no Slack message is sent." Genuinely attempt this. You'll find your only options are ugly: a global static flag, an environment variable check inside `SlackNotifier`, or PowerMock-style bytecode hacking. **Feel that dead end.** That feeling is the entire justification for DI, and describing it in an interview is far more convincing than reciting "loose coupling."

**Step 3 — Refactor to constructor injection** with a `NotificationChannel` interface. Write `RecordingChannel implements NotificationChannel` that stores messages in a list.

**Step 4 — Write the test that was impossible.** Note how long it took: probably 90 seconds.

**Step 5 — Now make it Spring.** Add `@Service` to `MonitorService`, `@Component` to `SlackNotifier`, and:
```java
@Component
public class StartupProbe implements ApplicationRunner {
    private final MonitorService service;
    public StartupProbe(MonitorService service) { this.service = service; }

    @Override public void run(ApplicationArguments args) {
        service.createMonitor("API", "https://api.example.com");
    }
}
```
Run the app. It works, and you wrote zero wiring code. **You now know exactly what Spring did for you** — reflected on the constructor, resolved by type, called `newInstance`. No magic remains.

**Step 6 — Break it on purpose:** delete `@Component` from `SlackNotifier`. Read the full error message carefully. Spring will tell you which bean was required, by which class, and suggest defining one. Notice how much better that is than the error from your mini-container in Lesson 04.

---

## 8. Interview traps

**Q1. What is dependency injection, and what problem does it solve?**
Supplying a class's collaborators from outside instead of having it construct them. It solves: hard coupling to concrete implementations, untestability (no seam to substitute a double), and centralized object-graph construction. The sharpest framing: `new` inside a class is a compile-time commitment; DI makes it a runtime choice.

**Q2. IoC vs DI vs DIP?**
IoC = the principle that an external party controls creation/flow. DI = a specific IoC technique (constructor/setter/field). DIP = the SOLID rule to depend on abstractions. DIP motivates, DI implements, IoC generalizes.

**Q3. Can you do DI without a framework?**
Yes — pass dependencies into constructors and wire them in `main()`. That's "manual DI" or "pure DI," and it's a completely valid choice for small systems. Frameworks add value when the graph is large, has sharing/scope rules, and needs cross-cutting concerns woven in.

**Q4. Spring vs Spring Boot?**
Spring Framework is the core (IoC, AOP, MVC, transactions). Spring Boot is a layer on top adding auto-configuration, starters/BOM, embedded server, executable JAR, and production tooling. Boot doesn't replace Spring — it configures it.

**Q5. What were the actual problems Spring Boot fixed?**
Bean boilerplate (→ auto-config), dependency-version compatibility (→ BOM/starters), deployment ceremony (→ embedded server + fat JAR), and lack of production endpoints (→ Actuator).

**Q6. When would you NOT use Spring Boot?**
Cold-start-sensitive serverless (→ native image or Quarkus/Micronaut), tiny CLIs, extremely latency-sensitive systems where framework layers are unwelcome, or a team with no JVM footing. Being able to answer this is a maturity signal — a candidate who thinks Spring is always right hasn't hit its edges.

**Q7. What's the biggest breaking change in Spring Boot 3, and why was it disruptive?**
The `javax.*` → `jakarta.*` namespace migration (Servlet, JPA, Validation), following Oracle's transfer of Java EE to Eclipse. Disruptive because every dependency in the ecosystem had to release a Jakarta-compatible version simultaneously — one un-migrated library could block a whole upgrade. Also: Java 17 baseline.

**Q8. What's the trade-off between annotation and XML configuration?**
Annotations: concise, co-located with code, refactor-safe, type-checked — but scattered, and require recompilation to change. XML/external: centralized, visible in one place, changeable without recompiling, and able to configure third-party classes you can't annotate — but verbose and not type-safe. Modern practice: annotations for beans, externalized properties for environment config.

**Q9. Why do Quarkus and Micronaut exist?**
They perform DI and AOP at **build time** via annotation processors rather than at runtime via reflection, yielding much faster startup and lower memory — targeting serverless and container-dense deployments. Spring's answer is AOT processing plus GraalVM native images.

---

## 9. Retention check

1. Explain in 60 seconds why `new SlackNotifier()` inside a service class is a design problem — without using the phrase "loose coupling."
2. Give the one-sentence distinction between IoC, DI, and DIP.
3. Name the four things Spring Boot adds to Spring Framework.
4. Give two situations where you'd argue *against* Spring Boot, and what you'd use instead.
5. What happened in Boot 3 that made upgrades painful, and why?

---

**Next:** [Lesson 06 — The IoC container & beans](06-ioc-container-and-beans.md) — inside the container: bean definitions, the startup sequence, and the object you've been using without knowing it.

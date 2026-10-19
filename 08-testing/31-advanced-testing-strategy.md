# Lesson 31 — Advanced testing & CI strategy

> **Why this lesson exists:** The previous two lessons taught you to write tests. This one is about the *system* around them: enforcing architecture automatically, catching breaking API changes before consumers do, measuring whether your tests actually work, and killing flakiness — which is the thing that destroys a team's trust in its own suite. These are the topics that come up in senior and staff-level interviews.

**Time:** ~65 minutes · **Prereq:** Lessons 29–30

---

## 1. The idea in one sentence

> **A rule nobody can break by accident is worth more than a rule everybody agrees with: ArchUnit turns layering into a failing build, a spec snapshot turns a breaking change into a PR diff, and mutation testing turns "we have coverage" into "our assertions actually matter."**

The flip side is the same principle: a flaky test is a rule that *can't* be trusted, so retrying it silently converts your suite from a signal into a ritual.

---

## 2. ArchUnit — architecture as executable tests

From Lesson 13: architecture erodes because nothing enforces it. **ArchUnit turns your rules into JUnit tests.**

```xml
<dependency>
    <groupId>com.tngtech.archunit</groupId>
    <artifactId>archunit-junit5</artifactId>
    <version>1.3.0</version>
    <scope>test</scope>
</dependency>
```

```java
@AnalyzeClasses(packages = "com.vigil", importOptions = ImportOption.DoNotIncludeTests.class)
class ArchitectureTest {

    // ── Layering ──────────────────────────────────────────────────────────────
    @ArchTest
    static final ArchRule controllersMustNotAccessRepositories =
        noClasses().that().haveSimpleNameEndingWith("Controller")
            .should().dependOnClassesThat().haveSimpleNameEndingWith("Repository")
            .because("controllers must go through services so logic is reusable "
                   + "from schedulers and message consumers");

    @ArchTest
    static final ArchRule repositoriesMustOnlyBeUsedByServices =
        classes().that().haveSimpleNameEndingWith("Repository")
            .should().onlyBeAccessed().byClassesThat()
            .haveSimpleNameEndingWith("Service")
            .orShould().onlyBeAccessed().byClassesThat().resideInAPackage("..config..");

    // ── Feature boundaries (Lesson 13) ────────────────────────────────────────
    @ArchTest
    static final ArchRule commonMustNotDependOnFeatures =
        noClasses().that().resideInAPackage("..vigil.common..")
            .should().dependOnClassesThat().resideInAnyPackage(
                "..vigil.monitor..", "..vigil.incident..", "..vigil.billing..")
            .because("common must contain no feature knowledge");

    @ArchTest
    static final ArchRule featuresMustNotShareRepositories =
        slices().matching("com.vigil.(*)..").should().notDependOnEachOther()
            .ignoreDependency(alwaysTrue(), resideInAPackage("..common.."))
            .as("features communicate through public services or events, not shared repositories");

    // ── Correctness rules that catch real bugs ───────────────────────────────
    @ArchTest
    static final ArchRule noFieldInjection =
        noFields().should().beAnnotatedWith(Autowired.class)
            .because("constructor injection only — Lesson 07");

    @ArchTest
    static final ArchRule entitiesMustNotBeReturnedFromControllers =
        noMethods().that().areDeclaredInClassesThat()
            .areAnnotatedWith(RestController.class)
            .should().haveRawReturnType(describe("a JPA entity",
                    c -> c.isAnnotatedWith(Entity.class)))
            .because("returning entities leaks the schema and triggers lazy-loading failures");

    @ArchTest
    static final ArchRule transactionalMethodsMustBePublic =
        methods().that().areAnnotatedWith(Transactional.class)
            .should().bePublic()
            .because("proxy-based AOP cannot advise private/protected methods — Lesson 09");

    @ArchTest
    static final ArchRule noSystemOutOrPrintStackTrace =
        noClasses().should().callMethod(System.class, "currentTimeMillis")
            .orShould().accessField(System.class, "out")
            .orShould().callMethodWhere(target(name("printStackTrace")))
            .because("use a logger, and inject a Clock instead of reading system time");

    @ArchTest
    static final ArchRule noJavaUtilDate =
        noClasses().should().dependOnClassesThat()
            .belongToAnyOf(java.util.Date.class, java.util.Calendar.class,
                           java.text.SimpleDateFormat.class)
            .because("java.time only — SimpleDateFormat is not thread-safe (Lesson 02)");

    // ── Universal sanity ─────────────────────────────────────────────────────
    @ArchTest
    static final ArchRule noCycles =
        slices().matching("com.vigil.(*)..").should().beFreeOfCycles();

    @ArchTest
    static final ArchRule noClassesDependOnPlayground =
        noClasses().that().resideOutsideOfPackage("..playground..")
            .should().dependOnClassesThat().resideInAPackage("..playground..");
}
```

**Three of those rules deserve highlighting because they catch bugs, not just style:**

- **`transactionalMethodsMustBePublic`** catches the silent `@Transactional` no-op from Lesson 09 — a *correctness* bug that produces no error, automatically detected forever.
- **`entitiesMustNotBeReturnedFromControllers`** catches the schema-leak and lazy-loading class of bug from Lesson 16.
- **`noCycles`** catches the design decay that eventually forces `spring.main.allow-circular-references=true`.

**Why ArchUnit is such good value:** it's ~30 lines, it runs in your normal test suite, the failure messages name the exact offending class and line, and **the `.because(...)` text documents the rationale where a new team member will actually see it — in the failure message.** Compare that to a wiki page nobody reads.

**Bring this up unprompted in an architecture interview.** *"I enforce layering with ArchUnit tests rather than convention, so violations fail CI with an explanation instead of being caught in review or not at all."* Very few candidates mention it, and it signals that you've watched architecture erode and decided to do something about it.

**Practical notes:** cache the imported classes (`@AnalyzeClasses` does this per class), keep rules in one file so they're discoverable, and when you must allow an exception, use `.ignoreDependency(...)` with a comment rather than deleting the rule.

---

## 3. Contract testing

**The problem:** Vigil's API has consumers (a web dashboard, a mobile app, customer scripts). Your integration tests prove *your* code works. They don't prove you haven't broken a consumer.

```java
// Rename this field and every client breaks. Your tests still pass.
public record MonitorResponse(UUID id, String name, String url) {}
//                                     ^^^^ renamed to "title" → 200 OK, and clients see null
```

### Level 1 — snapshot the OpenAPI spec (cheap, do this)

```java
@Test
void openApiSpecHasNotChangedUnintentionally() throws Exception {
    String actual = mockMvc.perform(get("/v3/api-docs"))
            .andReturn().getResponse().getContentAsString();

    String committed = Files.readString(Path.of("src/test/resources/api-snapshot.json"));

    // Fails on ANY change. Reviewing the diff is the point.
    assertThatJson(actual).isEqualTo(committed);
}
```

**The value isn't the test passing — it's that the *diff* appears in the pull request.** A reviewer sees "field `name` removed, field `title` added" and asks the right question. **You've turned an invisible breaking change into a visible one.** This is the highest value-to-effort item in this lesson.

### Level 2 — consumer-driven contracts (Pact / Spring Cloud Contract)

The consumer declares what it needs; the provider's build verifies it still provides it.

```groovy
// Spring Cloud Contract — a contract lives in the PROVIDER's repo, written with the consumer
Contract.make {
    description "should return a monitor by id"
    request {
        method GET()
        url "/api/v1/monitors/7f3a2b1c-0000-0000-0000-000000000000"
        headers { header("Authorization", anyNonEmptyString()) }
    }
    response {
        status OK()
        headers { contentType(applicationJson()) }
        body([ id: anyUuid(), name: anyNonEmptyString(), status: anyOf("UP","DOWN","PAUSED") ])
    }
}
```

Spring Cloud Contract generates provider verification tests **and** a stub JAR the consumer can test against — so both sides test against the same agreed shape without a shared environment.

| | Pact | Spring Cloud Contract |
|---|---|---|
| Direction | Consumer-driven (contracts generated from consumer tests) | Contracts defined in the provider repo (Groovy/YAML) |
| Polyglot | ✅ Excellent (JS, Python, Go, Java...) | Java-centric |
| Broker | Pact Broker (versioning, can-i-deploy) | Stub JARs in your artifact repository |
| Spring integration | Good | Excellent |

**When it's worth the setup:** several services with independent deploy cycles and different teams. **When it's not:** a single API with one frontend deployed together — the OpenAPI snapshot plus integration tests gives you most of the benefit for a fraction of the effort.

**The mature answer:** *"For a monolith with a coupled frontend, an OpenAPI snapshot diffed in the PR is enough. Contract tests earn their keep when independent teams deploy independently and you need to know whether a change is safe to release — which is what Pact's `can-i-deploy` actually answers."*

---

## 4. Mutation testing — testing your tests

**The question nobody asks: do my tests actually detect bugs?** Coverage says a line ran. It doesn't say an assertion would have caught it being wrong.

**Mutation testing changes your code deliberately and checks whether tests fail.**

```xml
<plugin>
    <groupId>org.pitest</groupId>
    <artifactId>pitest-maven</artifactId>
    <version>1.17.0</version>
    <configuration>
        <targetClasses><param>com.vigil.*.domain.*</param>
                       <param>com.vigil.*.*Service</param></targetClasses>
        <excludedClasses><param>com.vigil.*.dto.*</param></excludedClasses>
        <mutationThreshold>75</mutationThreshold>
        <timeoutConstant>10000</timeoutConstant>
    </configuration>
    <dependencies>
        <dependency><groupId>org.pitest</groupId>
            <artifactId>pitest-junit5-plugin</artifactId><version>1.2.1</version></dependency>
    </dependencies>
</plugin>
```

```bash
./mvnw test-compile org.pitest:pitest-maven:mutationCoverage
```

**Example mutations PIT applies:**
```java
if (interval < 30)  →  if (interval <= 30)      // boundary
return uptime;      →  return null;             // return value
a + b               →  a - b                    // arithmetic
if (x)              →  if (!x)                  // negation
notifier.send(m);   →  (removed)                // void method call
```

**A "surviving mutant" means PIT broke your code and no test noticed.** That's a genuine gap.

```
>> Line Coverage: 92%
>> Mutation Score: 61%
SURVIVED  UptimeCalculator:47  changed conditional boundary  (< to <=)
SURVIVED  IncidentService:82   removed call to notifier.send()
```

**Read that second one carefully: you can delete the notification entirely and the suite passes.** So the notification is effectively untested — which is exactly the kind of thing that reaches production.

**The honest assessment:** mutation testing is **slow** (it reruns your suite once per mutant — minutes to hours). So:
- Run it **on your core domain logic only**, not the whole codebase
- Run it **nightly or weekly** in CI, not on every commit
- Treat the score as a diagnostic, not a gate — though a threshold on the domain package is reasonable
- Target ~70–80% mutation score on business logic; 100% is not worth chasing (equivalent mutants exist that no test can distinguish)

**Saying "coverage tells you what ran; mutation testing tells you whether your assertions matter" is a memorable and genuinely insightful line** — and mentioning PIT puts you ahead of the large majority of candidates who only know JaCoCo.

---

## 5. Flaky tests — the thing that destroys trust

**A flaky test is worse than no test.** Once people learn to re-run CI until it goes green, every real failure gets re-run too. **The suite stops being a signal.**

### The causes, in order of frequency

**1. Time.**
```java
// ❌ Fails at midnight, in another timezone, or on a fast machine
assertThat(monitor.getCreatedAt()).isEqualTo(Instant.now());
assertThat(LocalDate.now().getDayOfWeek()).isNotEqualTo(SUNDAY);

// ✅ Inject a Clock (Lesson 29), or assert a range
assertThat(monitor.getCreatedAt()).isCloseTo(Instant.now(), within(1, SECONDS));
```

**2. Order dependence / shared state.**
```java
// ❌ Passes alone, fails in a suite — a leftover row from another test
@Test void findAllReturnsThreeMonitors() { assertThat(repo.findAll()).hasSize(3); }

// ✅ Assert on YOUR data only
@Test void findAllReturnsThisTestsMonitors() {
    assertThat(repo.findByOrganizationId(myOrg.getId())).hasSize(3);
}
```
**Absolute counts over shared tables are the single most common flake.** Scope every assertion to data the test created.

**3. Sleeps.**
```java
// ❌ Slow AND flaky — too short on a loaded CI runner, too long always
service.triggerAsyncCheck(monitor);
Thread.sleep(2000);
assertThat(repo.count()).isEqualTo(1);

// ✅ Awaitility — polls until true, fails fast on timeout
await().atMost(5, SECONDS)
       .untilAsserted(() -> assertThat(repo.count()).isEqualTo(1));
```
**`Awaitility` should replace every `Thread.sleep` in your test suite.** It's faster on a fast machine and more reliable on a slow one — strictly better in both directions.

**4. Randomness without a seed.** `UUID.randomUUID()` for identity is fine; random *values* that affect assertions are not. If you use randomized data (property-based testing), log the seed so failures are reproducible.

**5. Concurrency and hash ordering.**
```java
assertThat(map.keySet()).containsExactly("a", "b");        // ❌ HashMap order isn't guaranteed
assertThat(map.keySet()).containsExactlyInAnyOrder("a", "b");  // ✅
```

**6. External dependencies.** A test hitting a real API fails when the network hiccups or someone's rate limit trips. **Use WireMock** (Lesson 30). No test should require the internet.

**7. Port conflicts.** Always `RANDOM_PORT` and `dynamicPort()`; never a hardcoded port.

### The policy that actually works

```java
@Test
@Disabled("Flaky — fails ~1/20 on CI. Tracked in VIGIL-482. Owner: @you. Fix by 2026-08-15.")
void occasionallyFailsUnderLoad() { }
```

**Quarantine, don't delete, and don't retry.**

**Why not retry?** Adding `@RepeatedTest` or a retry extension makes the symptom disappear and the cause permanent. **A flaky test is usually reporting a real race condition** — in your code, not just your test. Retrying it means you've silenced a bug report about production behaviour. That's the argument to make, and it's a strong one.

**The process:** quarantine with a ticket, an owner, and a date; track flake rate in CI; and treat a flaky test as a P2 bug rather than an annoyance.

---

## 6. Property-based testing (know it exists)

Instead of examples, state *properties* that must hold for all inputs, and let the framework hunt for counterexamples.

```java
@Property
void uptimePercentageIsAlwaysBetweenZeroAndOneHundred(
        @ForAll @Size(min = 1, max = 200) List<@From("checkResults") CheckResult> results) {
    var uptime = calculator.calculate(results);
    assertThat(uptime.percentage()).isBetween(BigDecimal.ZERO, new BigDecimal("100"));
}

@Property
void resolvingThenReopeningAnIncidentNeverLosesTheOriginalStartTime(
        @ForAll("incidents") Incident incident) {
    Instant original = incident.getStartedAt();
    incident.resolve(); incident.reopen();
    assertThat(incident.getStartedAt()).isEqualTo(original);
}
```

**jqwik** (JUnit 5 native) or **QuickTheories**. The genuinely valuable feature is **shrinking**: when it finds a failure with a 200-element list, it automatically reduces it to the smallest failing case — often a 2-element list that makes the bug obvious.

**Where it earns its place:** invariants (percentages in range, monotonic timestamps, round-trip serialization `parse(format(x)) == x`), state machines (no sequence of transitions reaches an invalid state), and parsers. **Not** general business logic, where example-based tests communicate intent better.

**Vigil's real use:** the uptime calculator and the incident state machine — both are invariant-heavy, and both are exactly where a subtle edge case would go unnoticed for months.

---

## 7. CI strategy

### Stage the pipeline by feedback speed

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  fast:                                   # ~2 minutes — must pass before anything else
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: '21', distribution: 'temurin', cache: 'maven' }
      - name: Compile + unit tests + ArchUnit
        run: ./mvnw -B test -Dgroups='!integration'
      - name: Static analysis
        run: ./mvnw -B spotless:check

  integration:                            # ~6 minutes
    needs: fast
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { java-version: '21', distribution: 'temurin', cache: 'maven' }
      - name: Integration tests (Testcontainers)
        run: ./mvnw -B verify -Dgroups=integration
      - uses: actions/upload-artifact@v4
        if: always()
        with: { name: test-reports, path: '**/target/surefire-reports/**' }

  security:
    needs: fast
    runs-on: ubuntu-latest
    steps:
      - name: Dependency vulnerability scan
        run: ./mvnw -B org.owasp:dependency-check-maven:check
      - name: Secret scan
        uses: gitleaks/gitleaks-action@v2

  nightly-mutation:                       # slow — scheduled, not per-commit
    if: github.event_name == 'schedule'
    runs-on: ubuntu-latest
    steps:
      - run: ./mvnw -B test-compile org.pitest:pitest-maven:mutationCoverage
```

**The principle: order stages by how fast they fail.** A compile error should be known in 90 seconds, not after a 12-minute Testcontainers run. **Fail fast, cheap first.**

### Maven Surefire vs Failsafe — a genuinely useful convention

```xml
<plugin>
    <artifactId>maven-surefire-plugin</artifactId>       <!-- runs during `test` -->
    <configuration>
        <excludes><exclude>**/*IT.java</exclude></excludes>
    </configuration>
</plugin>
<plugin>
    <artifactId>maven-failsafe-plugin</artifactId>       <!-- runs during `verify` -->
    <configuration>
        <includes><include>**/*IT.java</include></includes>
    </configuration>
    <executions>
        <execution><goals><goal>integration-test</goal><goal>verify</goal></goals></execution>
    </executions>
</plugin>
```

**`*Test` = fast unit tests (Surefire, `mvn test`). `*IT` = integration tests (Failsafe, `mvn verify`).** So `mvn test` is your 20-second local loop and `mvn verify` is the full check. **And Failsafe's key difference: it doesn't fail the build immediately, so `post-integration-test` cleanup still runs** — which matters when you're tearing down containers.

### Static analysis worth having

| Tool | Purpose |
|---|---|
| **Spotless** | Auto-format. **Ends all formatting arguments in reviews permanently.** |
| **SpotBugs** (+ **find-sec-bugs**) | Bytecode bug patterns, including security ones |
| **Error Prone** | Google's compile-time checker; catches real bugs (e.g. `equals` on incompatible types) |
| **SonarQube / SonarCloud** | Aggregate quality gate; free for public repos |
| **NullAway** | Compile-time null-safety using JSpecify annotations (Lesson 02) |
| **OWASP dependency-check / Dependabot** | Known CVEs in dependencies |
| **gitleaks** | Committed secrets (Lesson 12) |

**Start with Spotless + Dependabot + gitleaks.** Highest value, near-zero friction, and **Spotless on a public portfolio repo means every file looks deliberate** — which reviewers notice.

### The quality gate

```
Pull request must have:
  ✅ compiles
  ✅ all tests pass
  ✅ ArchUnit rules pass
  ✅ formatting clean (Spotless)
  ✅ no new high/critical CVEs
  ✅ no secrets detected
  ⚠️ coverage not decreased on changed files    ← better than an absolute threshold
```

**"Coverage must not decrease on changed files" is a much better gate than "coverage must be ≥80%."** The absolute threshold punishes people who touch legacy code and encourages assertion-free tests to satisfy the number. The delta version asks the only question that matters: *is this change tested?*

---

## 8. Build & break — Vigil's test infrastructure

**1. ⭐ ArchUnit rules.** Write at least eight from §2, including `transactionalMethodsMustBePublic` and `entitiesMustNotBeReturnedFromControllers`. **Then deliberately violate each one and read the failure message.** Note that ArchUnit names the exact class and location. **These 30 lines are the highest-leverage code in your test suite.**

**2. OpenAPI snapshot test.** Commit `api-snapshot.json`, write the diff test. Then rename a DTO field and **watch the test fail with a readable diff.** Update the snapshot deliberately and note that the change is now visible in your git history. **This is what "the API contract is reviewed" looks like in practice.**

**3. Replace every sleep with Awaitility.**
```xml
<dependency><groupId>org.awaitility</groupId><artifactId>awaitility</artifactId>
  <scope>test</scope></dependency>
```
Find every `Thread.sleep` in your tests and convert it. **Record the suite time before and after.**

**4. Run PIT on your domain.**
```bash
./mvnw test-compile org.pitest:pitest-maven:mutationCoverage
```
Open `target/pit-reports/index.html`. Compare your **line coverage** against your **mutation score**. Find one surviving mutant in `UptimeCalculator` or `IncidentService`, write the test that kills it, and re-run. **That moment — finding a real gap in tests you were proud of — is the point of this exercise.**

**5. Manufacture a flake, then fix it properly.** Write a test asserting `repo.count() == 3`, then add a second test that inserts a row. Run them in different orders (`junit.jupiter.testmethod.order.default`) and watch it fail intermittently. **Then fix it by scoping to your own organization rather than by adding a retry.** Write down why retrying would have been the wrong fix.

**6. Split unit and integration tests.** Rename integration tests to `*IT`, configure Surefire/Failsafe, and verify `./mvnw test` is fast (seconds) while `./mvnw verify` is thorough.

**7. The GitHub Actions pipeline** from §7 with staged jobs. Push a commit with a compile error and confirm you get feedback in ~90 seconds rather than after the integration stage.

**8. Add Spotless.**
```xml
<plugin><groupId>com.diffplug.spotless</groupId><artifactId>spotless-maven-plugin</artifactId>
  <configuration><java><palantirJavaFormat/><removeUnusedImports/></java></configuration>
</plugin>
```
Run `./mvnw spotless:apply` once, commit the result, then add `spotless:check` to CI.

**9. Add gitleaks and Dependabot.** Commit a fake AWS key to a scratch branch and confirm gitleaks catches it. **Then delete the branch** and reflect on how much easier that was than rotating a real credential.

**10. Optional: property-based test the uptime calculator** with jqwik. Assert the percentage is always in [0, 100], that a list of all-UP results always yields 100, and that adding a PAUSED result never *decreases* uptime. **See if the shrinker finds an edge case you missed.** It often does.

### Explain out loud (60s)

1. Why an ArchUnit rule beats a documented convention, in terms of when each is enforced.
2. What a spec snapshot catches that your integration tests can't.
3. Coverage vs mutation testing, in one sentence each.
4. Why retrying a flaky test is the wrong fix.
5. How you'd order CI stages, and why that order.

---

## 9. Production rules

| Do / Never | Why |
|---|---|
| **Enforce layering with ArchUnit, not convention** | A violation then fails CI with the offending class named and the rationale in the message, instead of relying on a reviewer noticing |
| **Put the rationale in ArchUnit's `.because(...)`** | It is the one place a new team member will actually read it — a wiki page explaining the rule is a page nobody opens |
| **Snapshot the OpenAPI spec and diff it in the PR** | The value isn't the passing test; it's that "field removed, field added" becomes visible to a reviewer before a consumer discovers it |
| **Reach for consumer-driven contracts only when teams deploy independently** | For one API and one frontend released together, the spec snapshot plus integration tests gives most of the benefit for a fraction of the setup |
| **Use mutation testing to find untested assertions** | Coverage says a line ran; a surviving mutant says you could delete that behaviour entirely and nothing would fail |
| **Run mutation testing on changed files, not the whole codebase** | It reruns the suite once per mutant, so a full run is minutes to hours and will simply stop being run |
| **Never retry a flaky test** | Flakiness usually reports a real race in your code; a retry annotation silences the bug report and makes the cause permanent |
| **Quarantine flaky tests with a ticket, an owner and a date** | Deleting loses the signal and ignoring erodes trust until every genuine failure gets re-run too |
| **Never assert absolute counts over a shared table** | It is the single most common flake; scope every assertion to data the test itself created |
| **Replace every `Thread.sleep` with Awaitility** | It is faster on a fast machine and more reliable on a slow one — strictly better in both directions |
| **Never let a test require the internet** | A real API call fails on a network hiccup or someone else's rate limit, and the failure tells you nothing about your code |
| **Stage CI by how fast each step fails** | A compile error should surface in ninety seconds, not after a twelve-minute Testcontainers run |
| **Split `*Test` (Surefire) from `*IT` (Failsafe)** | `mvn test` stays a twenty-second loop, and Failsafe defers failure so container teardown still runs |
| **Gate on "coverage must not decrease on changed files"** | An absolute threshold punishes anyone touching legacy code and rewards assertion-free tests; the delta asks the only useful question |

---

## 10. Interview traps

**Q1. How do you stop architecture from eroding?**
ArchUnit tests in the normal suite, so violations fail CI with a message explaining the rule. It covers layering (controllers must not touch repositories), feature boundaries, cycles, and correctness rules like "`@Transactional` methods must be public" — which catches the proxy no-op that produces no error. Cheaper and stricter than review or a wiki page.

**Q2. What's the difference between coverage and mutation testing?**
Coverage says which lines executed. Mutation testing (PIT) modifies your code — flipping conditionals, changing return values, removing void calls — and checks whether a test fails. A surviving mutant means a real bug went undetected. Coverage tells you what ran; mutation testing tells you whether your assertions matter. It's slow, so run it on core domain logic nightly rather than per-commit.

**Q3. How do you know you haven't broken an API consumer?**
Cheapest: commit the generated OpenAPI spec and diff it in CI, so a breaking change appears in the pull request. Stronger: consumer-driven contract tests (Pact) or Spring Cloud Contract, where the provider build verifies contracts and consumers test against generated stubs. Contract tests earn their setup cost when independent teams deploy independently.

**Q4. What do you do about flaky tests?**
Quarantine with `@Disabled` plus a ticket, an owner, and a date — never retry. **Retrying hides the cause, and a flaky test is often reporting a real race condition in the code, not the test.** Then fix the root cause: inject a `Clock` instead of reading time, scope assertions to data the test created, replace sleeps with Awaitility, use random ports, and stub external HTTP.

**Q5. What's the most common cause of test flakiness?**
Shared state — usually an absolute count over a shared table (`assertThat(repo.findAll()).hasSize(3)`), which passes alone and fails in a suite. Fix by scoping every assertion to data the test itself created, which also unlocks parallel execution.

**Q6. Why is `Thread.sleep` bad in tests, and what replaces it?**
It's simultaneously too long (always wasted time) and too short (fails on a loaded CI runner). Awaitility polls until a condition holds with a timeout — faster on fast machines and more reliable on slow ones, so strictly better in both directions.

**Q7. How do you structure a CI pipeline?**
Stages ordered by feedback speed: compile + unit + ArchUnit + formatting (~2 min) → integration tests with Testcontainers (~6 min) → security scans in parallel → mutation testing on a nightly schedule. Fail fast and cheap first; never make a developer wait 12 minutes to learn about a compile error.

**Q8. Surefire vs Failsafe?**
Surefire runs `*Test` during the `test` phase; Failsafe runs `*IT` during `integration-test`/`verify`. The split gives a fast local loop and a thorough CI check. Failsafe also defers build failure so `post-integration-test` cleanup still runs — which matters for container teardown.

**Q9. Should coverage be a build gate?**
An absolute threshold is a poor gate — it punishes touching legacy code and incentivizes assertion-free tests. Better: **coverage must not decrease on changed files**, which asks the only useful question: is this change tested? Complement it with mutation testing on the domain to check the tests are meaningful.

**Q10. What is property-based testing and when would you use it?**
You assert properties that must hold for all inputs and the framework searches for counterexamples, then **shrinks** a failing case to its minimum. Ideal for invariants (values in range, round-trip serialization), state machines, and parsers. Example-based tests remain better for communicating specific business rules.

**Q11. Which static analysis tools would you add first?**
Spotless (ends formatting debates and makes every file look deliberate), Dependabot/OWASP dependency-check (CVEs — the category behind Log4Shell), and gitleaks (committed secrets). All near-zero friction, all high value.

---

## 11. Retention check

1. Name two ArchUnit rules that catch *correctness* bugs rather than style.
2. Explain in one sentence what mutation testing tells you that coverage can't.
3. Why is retrying a flaky test the wrong fix?
4. Give the single most common cause of flakiness and its fix.
5. Why is "coverage must not decrease on changed files" better than "coverage ≥ 80%"?
6. What's the cheapest way to make a breaking API change visible?

---

## Module 8 complete ✅

You can now write fast unit tests, integration tests against real infrastructure, security tests that function as controls, and a CI pipeline that enforces architecture and catches breaking changes. **This module is where your portfolio project stops looking like a tutorial and starts looking like production software.**

**Next:** [Lesson 32 — Async, scheduling & events](../09-production/32-async-scheduling-events.md) — background work that survives running on five instances at once.

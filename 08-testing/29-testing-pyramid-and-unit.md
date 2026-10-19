# Lesson 29 — The testing pyramid & unit tests

> **Why this lesson exists:** Testing is the clearest single signal of engineering maturity. Two candidates with identical feature knowledge are separated instantly by how they talk about tests — whether they mock everything, whether they test behaviour or implementation, whether their tests survive a refactor. And for a portfolio project, a good test suite is more persuasive than another feature.

**Time:** ~70 minutes · **Prereq:** Modules 1–7

---

## 1. The idea in one sentence

> **A test suite is not a bug net, it's a licence to change things — so the only tests worth keeping are the ones that still pass after a refactor that changed nothing a user can see.**

That single criterion decides almost every argument in this lesson: how much to mock, whether to verify interactions, what a good test name says, and why a service that only delegates to a repository deserves an integration test rather than a unit test full of stubs.

---

## 2. Why test — the reason that actually matters

The usual answer is "to catch bugs." That's the *least* important reason.

**The real reason: tests are what make change safe.** Without them, every modification carries unbounded risk, so the codebase calcifies — people stop refactoring, work around problems instead of fixing them, and the system rots. **A test suite is not a bug net; it's a licence to change things.**

Which reframes everything: a test that breaks whenever you refactor is *actively harmful*, because it taxes the exact activity tests exist to enable. That's the standard by which you judge a test — **does it let me change the implementation freely while telling me if I broke behaviour?**

---

## 3. The pyramid, and the honest version of it

```
              ╱╲            E2E / UI            few (5–20)
             ╱  ╲           slow (minutes), brittle, high confidence
            ╱────╲
           ╱      ╲         Integration          some (50–200)
          ╱        ╲        real DB, real HTTP layer, seconds
         ╱──────────╲
        ╱            ╲      Unit                 many (500–5000)
       ╱______________╲     no Spring context, milliseconds
```

**Why the shape:** cost and speed. A unit test runs in <1ms; a `@SpringBootTest` with Testcontainers takes seconds. **A suite you won't run isn't a suite** — if the full run takes 40 minutes, developers stop running it locally and the feedback loop dies.

### The honest amendment: the "testing trophy"

There's a well-argued counter-position (Kent C. Dodds' "testing trophy") that for typical web applications the **integration** layer deserves the most investment, because:
- Unit tests with heavy mocking verify that your mocks behave as you told them to — not that the system works
- Most real bugs live at *boundaries*: SQL, serialization, transaction scope, HTTP mapping — precisely what unit tests mock away
- With Testcontainers, integration tests are now fast enough that the cost argument is weaker than it was in 2010

**The calibrated position, and the one to give in an interview:**

> *"I use the pyramid as a cost heuristic, not a quota. Pure business logic — pricing, state machines, uptime calculation — gets fast unit tests with no mocks. Anything crossing a boundary — repositories, controllers, security rules, transaction behaviour — gets integration tests, because that's where bugs actually are and where mocks prove nothing. A service class whose only job is to call a repository doesn't need a unit test with a mocked repository; it needs an integration test against a real database."*

That answer shows you've thought about it rather than repeating a diagram.

---

## 4. Unit tests — no Spring, no mocks where possible

```java
class UptimeCalculatorTest {

    private final UptimeCalculator calculator = new UptimeCalculator();   // ← just `new`

    @Test
    void uptimeIsOneHundredPercentWhenAllChecksPass() {
        var results = List.of(up(), up(), up());

        var uptime = calculator.calculate(results);

        assertThat(uptime.percentage()).isEqualByComparingTo("100.00");
    }

    @Test
    void uptimeExcludesPausedPeriodsFromTheDenominator() {
        var results = List.of(up(), down(), paused(), paused());

        var uptime = calculator.calculate(results);

        // 1 up / 2 counted checks = 50%, paused periods don't count against the SLA
        assertThat(uptime.percentage()).isEqualByComparingTo("50.00");
    }

    @Test
    void uptimeIsUnknownRatherThanZeroWhenThereAreNoChecks() {
        assertThat(calculator.calculate(List.of()).isKnown()).isFalse();
    }
}
```

**Notice what's absent: no `@SpringBootTest`, no `@Mock`, no annotations at all.** This runs in under a millisecond. And notice the third test — `0%` vs `unknown` for a monitor with no data is a real product decision, and encoding it in a test is how that decision survives.

**This is why constructor injection matters** (Lesson 07 §3): `new UptimeCalculator()` is only possible because dependencies are explicit.

### JUnit 5 essentials

```java
@Test
@DisplayName("uptime excludes paused periods from the SLA denominator")
@Disabled("flaky — see VIGIL-482")          // ★ always with a reason and a ticket
@Tag("slow")

@BeforeEach  void setUp() { }               // before each test
@AfterEach   void tearDown() { }
@BeforeAll   static void once() { }         // must be static (unless @TestInstance(PER_CLASS))
@AfterAll    static void cleanup() { }

@Nested                                      // group related tests, share setup
class WhenMonitorIsPaused { }

@ParameterizedTest
@ValueSource(ints = {29, 0, -1})
void rejectsIntervalsBelowThirtySeconds(int interval) { }

@ParameterizedTest
@CsvSource({ "100,0,100.00", "99,1,99.00", "0,100,0.00" })
void calculatesPercentage(int up, int down, String expected) { }

@ParameterizedTest
@EnumSource(MonitorStatus.class)
void everyStatusHasAColour(MonitorStatus status) {
    assertThat(status.colour()).isNotNull();     // ★ automatically covers new enum values
}

@ParameterizedTest
@MethodSource("edgeCases")
void handlesEdgeCases(List<CheckResult> input, String expected) { }
static Stream<Arguments> edgeCases() { return Stream.of(arguments(List.of(), "UNKNOWN")); }
```

**`@EnumSource` is quietly excellent:** add a new `MonitorStatus` next year and the test automatically covers it. That's a test that gets *better* without being edited.

**JUnit 4 → 5 differences** (you'll meet JUnit 4 in older codebases):

| JUnit 4 | JUnit 5 |
|---|---|
| `@Before` / `@After` | `@BeforeEach` / `@AfterEach` |
| `@BeforeClass` / `@AfterClass` | `@BeforeAll` / `@AfterAll` |
| `@Ignore` | `@Disabled` |
| `@RunWith` / `@Rule` | `@ExtendWith` (one unified extension model) |
| `@Test(expected = X.class)` | `assertThrows` / `assertThatThrownBy` |
| Test classes/methods must be `public` | Package-private is fine (and preferred) |

### AssertJ — use it, not JUnit assertions

```java
// ❌ JUnit assertions — poor failure messages, awkward argument order
assertEquals(expected, actual);
assertTrue(monitors.size() == 3);

// ✅ AssertJ — fluent, discoverable, excellent failure output
assertThat(monitor.getStatus()).isEqualTo(MonitorStatus.DOWN);
assertThat(monitors).hasSize(3)
                    .extracting(Monitor::getName)
                    .containsExactly("api", "web", "db");
assertThat(monitors).filteredOn(m -> m.getStatus().isFailing()).hasSize(1);
assertThat(response.uptime()).isEqualByComparingTo("99.95");     // ★ BigDecimal-safe
assertThat(monitor).usingRecursiveComparison()
                   .ignoringFields("id", "createdAt")
                   .isEqualTo(expected);

assertThatThrownBy(() -> service.create(invalidRequest))
        .isInstanceOf(ValidationException.class)
        .hasMessageContaining("interval");

assertThatCode(() -> service.create(validRequest)).doesNotThrowAnyException();

// Soft assertions — report ALL failures, not just the first
SoftAssertions.assertSoftly(softly -> {
    softly.assertThat(response.name()).isEqualTo("api");
    softly.assertThat(response.status()).isEqualTo(UP);
    softly.assertThat(response.uptime()).isEqualByComparingTo("100.00");
});
```

**Three AssertJ features worth internalizing:**
- **`isEqualByComparingTo`** for `BigDecimal` — `isEqualTo` compares scale, so `99.9` ≠ `99.90` and your test fails for no real reason (Lesson 02).
- **`usingRecursiveComparison().ignoringFields(...)`** — compare whole objects while ignoring generated IDs and timestamps. Removes enormous amounts of assertion boilerplate.
- **Soft assertions** — one run tells you everything that's wrong, instead of fix-rerun-fix-rerun.

AssertJ ships with `spring-boot-starter-test`, so there's no reason not to use it.

---

## 5. Mockito — and the discipline of using it sparingly

```java
@ExtendWith(MockitoExtension.class)
class IncidentServiceTest {

    @Mock  private IncidentRepository incidentRepository;
    @Mock  private NotificationDispatcher dispatcher;
    @Captor private ArgumentCaptor<Incident> incidentCaptor;

    private IncidentService service;

    @BeforeEach
    void setUp() {
        // ★ Constructor injection: explicit, and adding a dependency breaks COMPILATION
        service = new IncidentService(incidentRepository, dispatcher,
                                      Clock.fixed(Instant.parse("2026-07-30T12:00:00Z"), UTC));
    }

    @Test
    void opensAnIncidentAndNotifiesWhenAMonitorGoesDown() {
        var monitor = TestData.monitor().withStatus(UP).build();
        when(incidentRepository.findOpenByMonitorId(monitor.getId())).thenReturn(Optional.empty());
        when(incidentRepository.save(any())).thenAnswer(inv -> inv.getArgument(0));

        service.handleFailure(monitor, "connection refused");

        verify(incidentRepository).save(incidentCaptor.capture());
        assertThat(incidentCaptor.getValue().getStartedAt())
                .isEqualTo(Instant.parse("2026-07-30T12:00:00Z"));      // ★ fixed Clock
        verify(dispatcher).dispatch(any(IncidentOpenedEvent.class));
    }

    @Test
    void doesNotOpenASecondIncidentWhenOneIsAlreadyOpen() {
        var monitor = TestData.monitor().build();
        when(incidentRepository.findOpenByMonitorId(monitor.getId()))
                .thenReturn(Optional.of(TestData.openIncident(monitor)));

        service.handleFailure(monitor, "connection refused");

        verify(incidentRepository, never()).save(any());
        verifyNoInteractions(dispatcher);          // ★ assert the ABSENCE of a notification
    }
}
```

**Note `@InjectMocks` is absent, deliberately** (Lesson 07 §3): explicit construction means adding a dependency breaks compilation rather than silently injecting null.

**Note the injected `Clock`.** Any code calling `Instant.now()` directly is untestable for time-dependent behaviour. Inject a `Clock`, use `Clock.fixed(...)` in tests, and time becomes an input. **This is a small design habit with a large testing payoff** and a nice thing to demonstrate.

### Mockito API

```java
when(repo.findById(id)).thenReturn(Optional.of(monitor));
when(repo.save(any())).thenAnswer(inv -> inv.getArgument(0));      // echo the argument
when(repo.count()).thenThrow(new DataAccessResourceFailureException("db down"));
when(repo.findAll()).thenReturn(first).thenReturn(second);         // consecutive calls

verify(repo).save(monitor);
verify(repo, times(2)).save(any());
verify(repo, never()).delete(any());
verify(repo, atLeastOnce()).findById(id);
verifyNoInteractions(dispatcher);
verifyNoMoreInteractions(repo);                                    // ⚠️ brittle, avoid
InOrder order = inOrder(repo, dispatcher);
order.verify(repo).save(any()); order.verify(dispatcher).dispatch(any());

any(), anyString(), eq("x"), argThat(m -> m.getName().equals("api"))
```

### When NOT to mock — the part that shows judgment

**❌ Don't mock value objects or data.**
```java
@Mock Monitor monitor;                              // ❌ just build one
Monitor monitor = TestData.monitor().build();       // ✅
```

**❌ Don't mock types you don't own.**
Mocking `RestClient` or `EntityManager` means your test encodes *your belief* about how the library behaves. When that belief is wrong, the test passes and production fails. **Use the real thing (Testcontainers) or a fake server (WireMock) instead.** This is one of the most valuable testing principles there is.

**❌ Don't mock the thing you're testing.** Obvious, and it happens — usually via a spy on the class under test.

**❌ Don't over-verify.**
```java
// ❌ This test breaks when you refactor internals, without any behaviour changing
verify(repo).findById(id);
verify(repo).save(any());
verify(mapper).toResponse(any());
verify(auditLog).record(any());
verifyNoMoreInteractions(repo, mapper, auditLog);
```
**You've now tested the implementation, not the behaviour.** Rename a method, extract a helper, reorder two calls — the test fails though nothing broke. **Assert on outcomes; verify only interactions that *are* the outcome** (a notification sent, an event published, a row saved).

**❌ Don't mock so heavily that the test proves nothing.**
```java
// What does this actually verify? That Mockito returns what you told it to.
when(repo.findById(id)).thenReturn(Optional.of(monitor));
when(mapper.toResponse(monitor)).thenReturn(response);
assertThat(service.get(id)).isEqualTo(response);
```
**A service that only delegates doesn't need a unit test.** Test it via an integration test where the repository and mapper are real — then you're testing something.

### Mocks vs stubs vs fakes vs spies — a common interview question

| Term | What it is |
|---|---|
| **Dummy** | Passed but never used (just to satisfy a signature) |
| **Stub** | Returns canned answers. No assertions about calls. |
| **Mock** | A stub you also assert *interactions* on |
| **Spy** | Wraps a real object; some methods real, some stubbed. ⚠️ Usually a smell |
| **Fake** | A **working** lightweight implementation — in-memory repository, H2, a stub HTTP server |

**Fakes are underrated and often the best option:**
```java
class InMemoryMonitorRepository implements MonitorRepository {
    private final Map<UUID, Monitor> store = new ConcurrentHashMap<>();
    public Optional<Monitor> findById(UUID id) { return Optional.ofNullable(store.get(id)); }
    public Monitor save(Monitor m) { store.put(m.getId(), m); return m; }
}
```
30 lines, reusable everywhere, and it behaves *consistently* — no per-test `when(...)` setup, so tests read as behaviour rather than mock choreography. **When you find yourself writing the same five `when(...)` lines in twelve tests, write a fake instead.** That refactor is a genuinely senior move.

**On spies specifically:** `@Spy` on the class under test to stub one of its own methods almost always means the class does too much — extract the stubbed part into a collaborator. Naming that is a good code-review instinct.

---

## 6. Test quality — how to judge a test

### The FIRST properties

| | |
|---|---|
| **F**ast | Milliseconds. A slow suite doesn't get run. |
| **I**ndependent | Any order, any subset. No shared mutable state. |
| **R**epeatable | Same result every time. No clocks, no randomness, no network. |
| **S**elf-validating | Passes or fails. Never "check the console output." |
| **T**imely | Written with the code, not "later" (which means never). |

### Structure: Arrange–Act–Assert

```java
@Test
void acknowledgingAnIncidentStopsEscalation() {
    // Arrange
    var incident = TestData.openIncident().withEscalationLevel(1).build();

    // Act
    service.acknowledge(incident.getId(), "oncall@vigil.io");

    // Assert
    assertThat(incident.getStatus()).isEqualTo(ACKNOWLEDGED);
    verify(escalationScheduler).cancel(incident.getId());
}
```

**One Act per test.** If you have two, you have two tests.

### Naming — describe the behaviour

```java
// ❌ Tells you nothing when it fails in CI at 2am
@Test void test1() { }
@Test void testCreate() { }
@Test void testCreateMonitor() { }

// ✅ The failure message alone tells you what broke
@Test void rejectsIntervalBelowThirtySeconds() { }
@Test void doesNotOpenSecondIncidentWhenOneIsAlreadyOpen() { }
@Test void returns404WhenMonitorBelongsToAnotherOrganization() { }
```

**The test that most often catches real bugs is the one asserting what *doesn't* happen** — no duplicate incident, no notification for a paused monitor, no cross-tenant row in a list. Positive-path tests catch typos; negative-path tests catch design errors.

### Coverage — the honest position

**Coverage measures which lines executed, not whether behaviour is correct.** You can have 100% coverage and zero assertions.

```java
@Test
void coversEverythingAndTestsNothing() {
    service.createMonitor(request);        // 100% coverage of that method. No assertion.
}
```

**A reasonable stance:** 70–80% line coverage as a *smell detector* (a class at 20% deserves a look), with attention to *branch* coverage on business logic. **Don't chase 100%** — the last 20% is usually getters, generated code, and unreachable defensive branches, and mandating it produces assertion-free tests written to satisfy a gate.

**Better questions than "what's our coverage?":**
- Does every bug we've fixed have a regression test?
- Can we refactor confidently?
- Do tests fail for the right reasons? (**Mutation testing** answers this — Lesson 31.)

---

## 7. Test data — the thing that actually determines maintainability

**Test data setup is where test suites go to die.** Twenty tests each constructing a full `Monitor` with eight fields means one constructor change breaks all twenty.

**The Object Mother / Builder pattern:**

```java
public final class TestData {

    public static MonitorBuilder monitor() {
        return new MonitorBuilder()                     // ★ sensible defaults for EVERYTHING
                .withName("api-health")
                .withUrl("https://api.example.com/health")
                .withType(MonitorType.HTTP)
                .withStatus(MonitorStatus.UP)
                .withIntervalSeconds(60)
                .withOrganization(organization().build());
    }

    public static OrganizationBuilder organization() {
        return new OrganizationBuilder().withName("Acme").withPlan(Plan.PRO);
    }

    public static CheckResult upResult(Monitor m, long ms) { ... }
    public static CheckResult downResult(Monitor m, String error) { ... }
}
```

```java
// Each test states ONLY what matters to it. Everything else is a sensible default.
var monitor = TestData.monitor().withStatus(DOWN).build();
var paused  = TestData.monitor().withStatus(PAUSED).withIntervalSeconds(300).build();
```

**Why this is the single highest-leverage testing practice:**
1. **Tests express intent.** `withStatus(DOWN)` is visibly the thing under test; the other seven fields are noise you've removed.
2. **One place to change** when a required field is added.
3. **Tests stay short**, so they're read rather than skimmed.

**Rules:** defaults must produce a **valid** object; never share mutable instances between tests (return a fresh builder each call); and keep it in `src/test/java` under a `support` or `testdata` package.

---

## 8. Build & break — Vigil's unit tests

**1. Extract genuinely testable logic.** Pull `UptimeCalculator` out of any service:
```java
public class UptimeCalculator {
    public Uptime calculate(List<CheckResult> results) { ... }   // pure function
    public Duration totalDowntime(List<CheckResult> results) { ... }
    public long p95ResponseTime(List<CheckResult> results) { ... }
}
```
**Pure functions, no dependencies, no Spring.** Test them exhaustively with `@ParameterizedTest`. Include: empty list, all up, all down, mixed, paused periods excluded, a single check, and the p95 of an even- vs odd-length list (the off-by-one that percentile code always has).

**2. Test the incident state machine** — this is where the real business rules live:
```java
@Nested class WhenNoIncidentIsOpen {
    @Test void opensAnIncidentOnFailure() { }
    @Test void doesNothingOnSuccess() { }
}
@Nested class WhenAnIncidentIsOpen {
    @Test void doesNotOpenASecondIncident() { }
    @Test void resolvesTheIncidentOnRecovery() { }
    @Test void recordsRecoveryTimeForMttrReporting() { }
}
@Nested class WhenAnIncidentIsAcknowledged {
    @Test void doesNotReEscalate() { }
    @Test void stillResolvesOnRecovery() { }
}
```
**Use `@Nested` for the states.** The test class becomes readable documentation of the state machine — and that's a genuinely good thing to show an interviewer.

**3. Build `TestData` with builders** for `Monitor`, `Organization`, `CheckResult`, and `Incident`. Then go back and rewrite your earlier tests to use it. **Measure the line count before and after.**

**4. Inject a `Clock` everywhere** you currently call `Instant.now()`:
```java
@Bean Clock clock() { return Clock.systemUTC(); }
```
Then write a test proving that an incident's `startedAt` is exactly your fixed instant. **Now time is an input, and you can test "escalate after 5 minutes" without sleeping.**

**5. Write a fake repository.** `InMemoryMonitorRepository implements MonitorRepository`. Use it in a service test with **zero mocks**. Compare readability against the mocked version and decide which you prefer.

**6. Feel the over-verification problem.** Write a test with five `verify(...)` calls and `verifyNoMoreInteractions`. Then do a pure refactor of the service — extract a private method, reorder two independent calls. **Watch tests fail though behaviour is unchanged.** Then rewrite them to assert outcomes only, and repeat the refactor. **This experiment teaches the difference between testing behaviour and testing implementation better than any explanation.**

**7. Write the negative tests.** For each feature, one test asserting what must *not* happen: no notification for a paused monitor; no second incident; no state change on an invalid transition; no downtime counted while paused.

**8. Check coverage, then interrogate it.** `mvn verify` with JaCoCo:
```xml
<plugin><groupId>org.jacoco</groupId><artifactId>jacoco-maven-plugin</artifactId>
  <executions>
    <execution><goals><goal>prepare-agent</goal></goals></execution>
    <execution><id>report</id><phase>verify</phase><goals><goal>report</goal></goals></execution>
  </executions>
</plugin>
```
Open `target/site/jacoco/index.html`. **Find a class with high coverage and few assertions, and admit to yourself that the number is lying.** That's the lesson.

### Explain out loud (60s)

1. The real reason to test, in one sentence that isn't "to catch bugs".
2. The calibrated pyramid answer: what gets a unit test and what doesn't.
3. Mock vs stub vs fake vs spy, and when a fake beats all of them.
4. What over-verification costs you, demonstrated with a refactor.
5. Why 100% coverage is not the goal, and what you'd ask instead.

---

## 9. Production rules

| Do / Never | Why |
|---|---|
| **Treat the pyramid as a cost heuristic, not a quota** | Pure logic earns fast unit tests; anything crossing a boundary earns an integration test, because that's where the bugs are and where mocks prove nothing |
| **Test pure business logic with no Spring and no mocks** | A sub-millisecond test with explicit construction is the cheapest feedback you will ever get, and it only exists because of constructor injection |
| **Inject a `Clock` instead of calling `Instant.now()`** | Time becomes an input you can fix, which is the difference between testing time-dependent behaviour and hoping |
| **Assert on outcomes; verify only interactions that *are* the outcome** | Over-verification tests the implementation, so extracting a helper or reordering two calls fails a test although nothing broke |
| **Never mock a type you don't own, or a value object** | A mock of someone else's library encodes your belief about its behaviour, and the test keeps passing after that belief stops being true |
| **Prefer a fake to a pile of stubs** | An in-memory repository is a working implementation, so the test exercises real behaviour instead of asserting your own assumptions back at you |
| **Don't unit-test a service that only delegates** | Mocking the repository and the mapper leaves nothing under test — cover it with an integration test where both are real |
| **Avoid `@Spy` on the class under test** | Stubbing a class's own method almost always means it does two jobs; extract the stubbed part into a collaborator instead |
| **Write one negative test per feature** | Positive tests catch typos; "no notification for a paused monitor" and "no second incident" catch design errors |
| **Name a test after the behaviour, not the method** | The name is what someone reads when it fails at 2am, and a method name tells them nothing about what was expected |
| **Build an Object Mother or builders for test data** | Twenty tests each constructing an eight-field entity means one constructor change breaks all twenty |
| **Use coverage as a smell detector, never as a target** | It measures which lines ran, not whether behaviour is correct — mandating 100% reliably produces assertion-free tests |

---

## 10. Interview traps

**Q1. Why write tests?**
Primarily to make change safe. Without them every modification carries unbounded risk, so refactoring stops and the codebase calcifies. Catching bugs is a benefit; enabling change is the purpose. It follows that a test which breaks on every refactor is actively harmful.

**Q2. What's the testing pyramid, and do you agree with it?**
Many fast unit tests, fewer integration tests, very few E2E — driven by cost and speed. I use it as a heuristic, not a quota: pure business logic gets unit tests; anything crossing a boundary (SQL, HTTP, transactions, security) gets integration tests, because that's where bugs live and where mocks prove nothing. With Testcontainers, integration tests are cheap enough to justify a heavier middle layer than the classic shape suggests.

**Q3. When should you NOT mock?**
Value objects and data (build them). Types you don't own (your mock encodes your possibly-wrong belief about the library — use the real thing or a fake server). The class under test. And avoid mocking so much that the test only verifies Mockito returned what you configured.

**Q4. Mock vs stub vs fake?**
Stub returns canned values. Mock is a stub you also assert interactions on. Fake is a working lightweight implementation (in-memory repository, WireMock). Fakes are often best: consistent behaviour, no per-test setup, and tests read as behaviour rather than mock choreography.

**Q5. Why avoid `@InjectMocks`?**
It sets fields reflectively, so a newly added dependency is silently null and the test NPEs at runtime. With explicit constructor calls, adding a dependency breaks **compilation** of every test — immediate, precise feedback.

**Q6. What's wrong with over-verifying interactions?**
It tests implementation rather than behaviour, so pure refactors break tests without any behaviour change — taxing the very activity tests exist to enable. Assert outcomes; verify only interactions that *are* the outcome (a message sent, an event published).

**Q7. Is 100% coverage a good goal?**
No. Coverage measures execution, not correctness — an assertion-free test can give 100%. Use 70–80% as a smell detector with attention to branch coverage on business logic. Better questions: does every fixed bug have a regression test, and do tests fail for the right reasons (which mutation testing measures)?

**Q8. How do you test time-dependent logic?**
Inject a `Clock` and use `Clock.fixed(...)` in tests, so time is an input rather than an ambient dependency. Never `Thread.sleep` to wait for time to pass — it's slow and flaky. For scheduling, verify the scheduled instant rather than waiting for it.

**Q9. How do you keep test data maintainable?**
Builders/Object Mothers with valid defaults, so each test specifies only the field under test. One place to update when a required field appears, and tests read as intent instead of construction noise. It's the highest-leverage testing practice I know.

**Q10. What makes a good test name?**
It describes the behaviour and the condition, so the failure message alone explains what broke: `doesNotOpenSecondIncidentWhenOneIsAlreadyOpen`, not `testCreate`.

**Q11. Which tests catch the most real bugs?**
Negative ones — asserting what must *not* happen (no duplicate incident, no notification while paused, no cross-tenant row). Positive-path tests catch typos; negative tests catch design errors.

---

## 11. Retention check

1. Give the primary reason to write tests, and the consequence for how you judge a test.
2. Name three things you should never mock, and what to use instead for each.
3. Why is `@InjectMocks` worse than explicit construction?
4. Explain how over-verification makes a test harmful.
5. Why is a `Clock` bean worth the small ceremony?
6. Why do builders matter more than any other testing practice?

---

**Next:** [Lesson 30 — Spring integration testing & Testcontainers](30-spring-integration-testing.md) — testing against a real Postgres, fast, with slices.

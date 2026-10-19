# Lesson 30 — Spring integration testing & Testcontainers

> **Why this lesson exists:** Integration tests are where Spring bugs actually live — SQL that doesn't match your entity, a transaction that doesn't roll back, JSON that serializes wrong, a security rule that doesn't apply. This is also where slow test suites are born, so knowing about **test slices** and **context caching** is the difference between a 40-second suite and a 12-minute one.

**Time:** ~80 minutes · **Prereq:** Lesson 29, Module 6

---

## 1. The idea in one sentence

> **An integration test is only worth its seconds if it exercises the thing production will: the real database, a real HTTP round trip, the real security rules — and if it doesn't, it can pass while the code is broken.**

The two forces in this lesson pull against each other. Realism costs time, so test slices and the context cache exist to buy it back. Knowing which realism you're giving up in each slice is the whole skill.

---

## 2. `@SpringBootTest` and the context cache

```java
@SpringBootTest
class VigilApplicationTests {
    @Test
    void contextLoads() { }        // ← the single most valuable test in the project
}
```

**That empty test is not a joke.** It proves the entire object graph can be constructed: every bean resolves, every `@ConfigurationProperties` binds and validates, no circular dependencies, no missing beans. **A wiring error becomes a failed build instead of a failed deploy.** Every project should have it.

### Context caching — the most important performance fact in Spring testing

Starting a Spring context takes 2–10 seconds. **Spring's `TestContext` framework caches contexts and reuses them across test classes** — keyed on the *configuration*:

- the classes/locations loaded
- active profiles
- property sources (`@TestPropertySource`, `properties = {...}`)
- `@MockitoBean` / `@MockitoSpyBean` definitions (**each unique set = a new context**)
- `ContextCustomizer`s, `@DirtiesContext` usage

**Consequence: every distinct configuration costs another full context startup.**

```java
@SpringBootTest                                              // context A
class MonitorServiceTest { }

@SpringBootTest                                              // context A — REUSED ✅ (free)
class IncidentServiceTest { }

@SpringBootTest(properties = "vigil.checks.enabled=false")   // context B — NEW ⏱️
class DisabledChecksTest { }

@SpringBootTest
@MockitoBean(NotificationDispatcher.class)                   // context C — NEW ⏱️
class NotificationTest { }
```

**Practical rules that keep a suite fast:**

1. **Standardize configuration.** Put shared setup in an `AbstractIntegrationTest` base class so dozens of test classes share one context.
2. **Avoid one-off `properties = {...}`.** Each unique set forks a context. Group tests that need special config together.
3. **Minimize `@MockitoBean` variety.** Each distinct combination is a new context. Prefer real beans, or a shared set of mocks in the base class.
4. **`@DirtiesContext` is a last resort** — it evicts the cached context, so the *next* test class pays full startup. One careless `@DirtiesContext` can add minutes.

**"Why is our test suite slow?" is a genuine interview question, and "too many distinct Spring contexts, usually from scattered `properties` and `@MockitoBean` combinations" is an answer that immediately marks you as someone who has fixed it.**

Diagnose it:
```properties
logging.level.org.springframework.test.context.cache=DEBUG
```
It logs cache hits, misses, and size — you'll see exactly how many contexts you're building.

---

## 3. Test slices — load only what you need

Boot provides annotations that load a **subset** of auto-configuration. Much faster than `@SpringBootTest`.

| Annotation | Loads | For testing |
|---|---|---|
| `@WebMvcTest` | MVC layer, controllers, `@ControllerAdvice`, converters, filters. **No services, no repositories.** | Controllers, HTTP mapping, JSON, error handling |
| `@DataJpaTest` | JPA, repositories, `DataSource`, `TestEntityManager`. **Transactional + rolled back.** | Repositories, queries, mappings |
| `@JdbcTest` | `DataSource`, `JdbcTemplate` | `JdbcClient` code |
| `@JsonTest` | Jackson, `ObjectMapper` | Serialization contracts |
| `@RestClientTest` | `RestClient`/`RestTemplate` + `MockRestServiceServer` | Outbound HTTP clients |
| `@DataRedisTest` | Redis | Cache repositories |
| `@WebFluxTest` | WebFlux + `WebTestClient` | Reactive controllers |

### `@WebMvcTest` — controllers in isolation

```java
@WebMvcTest(MonitorController.class)                 // ★ name the controller — smaller context
@Import(SecurityConfig.class)                         // security isn't auto-included in the slice
class MonitorControllerTest {

    @Autowired  private MockMvc mockMvc;
    @Autowired  private ObjectMapper objectMapper;
    @MockitoBean private MonitorService monitorService;      // ★ the service is MOCKED

    @Test
    @WithMockUser(authorities = "monitors:read")
    void returnsMonitorWhenItExists() throws Exception {
        var response = new MonitorResponse(ID, "api", "https://x", HTTP, UP, 60, null, null, null, null);
        when(monitorService.get(ID)).thenReturn(response);

        mockMvc.perform(get("/api/v1/monitors/{id}", ID))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.id").value(ID.toString()))
               .andExpect(jsonPath("$.name").value("api"))
               .andExpect(jsonPath("$.status").value("UP"))
               .andExpect(jsonPath("$.passwordHash").doesNotExist());   // ★ leak check
    }

    @Test
    @WithMockUser(authorities = "monitors:write")
    void returns400WithProblemDetailWhenIntervalIsTooSmall() throws Exception {
        var request = new CreateMonitorRequest("api", "https://x", 5, HTTP, null);

        mockMvc.perform(post("/api/v1/monitors")
                        .contentType(APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(request)))
               .andExpect(status().isBadRequest())
               .andExpect(jsonPath("$.title").value("Validation failed"))
               .andExpect(jsonPath("$.errors[*].field").value(hasItem("intervalSeconds")));
    }

    @Test
    void returns401WhenUnauthenticated() throws Exception {
        mockMvc.perform(get("/api/v1/monitors/{id}", ID))
               .andExpect(status().isUnauthorized());
    }

    @Test
    @WithMockUser(authorities = "monitors:read")
    void returns403WhenDeletingWithoutDeletePermission() throws Exception {
        mockMvc.perform(delete("/api/v1/monitors/{id}", ID).with(csrf()))
               .andExpect(status().isForbidden());
    }
}
```

**`@WebMvcTest` tests exactly the layer you want:** URL mapping, argument binding, validation, serialization, status codes, error contract, and security rules — **without a database.** The service is mocked because it's not what's under test.

**Note the `$.passwordHash` doesNotExist assertion.** A test that asserts a field is *absent* is how you prevent a future field addition from leaking data. Cheap, and almost nobody writes them.

> `@MockitoBean` is the Boot 3.4+ replacement for `@MockBean` (deprecated). Same idea: replace a bean in the context with a Mockito mock. `@MockitoSpyBean` replaces `@SpyBean`.

**MockMvc essentials:**
```java
mockMvc.perform(get("/api/v1/monitors").param("status", "DOWN")
                .header("X-Correlation-Id", "abc")
                .with(csrf())                                  // when CSRF is enabled
                .accept(APPLICATION_JSON))
       .andDo(print())                                          // ★ dumps the full req/resp — debug gold
       .andExpect(status().isOk())
       .andExpect(header().exists("Location"))
       .andExpect(content().contentType(APPLICATION_JSON))
       .andExpect(jsonPath("$.content", hasSize(3)))
       .andExpect(jsonPath("$[?(@.status=='DOWN')]").exists())
       .andReturn();
```

**`.andDo(print())` is the debugging tool to remember** — it prints the resolved handler, the full request, and the full response body. It answers "why is this 400?" in one line.

> **Boot 3.4+:** `MockMvcTester` is a newer AssertJ-native API (`assertThat(mvc.get().uri(...)).hasStatus(OK)`). Nicer assertions, same underlying engine. Worth knowing it exists.

### `@DataJpaTest` — repositories against a real database

```java
@DataJpaTest
@Import(TestcontainersConfig.class)                   // real Postgres, not H2 — see §3
class MonitorRepositoryTest {

    @Autowired private MonitorRepository repository;
    @Autowired private TestEntityManager em;           // ★ direct persistence-context control

    @Test
    void findByIdAndOrgIdDoesNotReturnAnotherOrgsMonitor() {
        var orgA = em.persist(TestData.organization().withName("A").build());
        var orgB = em.persist(TestData.organization().withName("B").build());
        var monitorInB = em.persist(TestData.monitor().withOrganization(orgB).build());
        em.flush();
        em.clear();                                    // ★ force a real DB read, not a cache hit

        assertThat(repository.findByIdAndOrgId(monitorInB.getId(), orgA.getId())).isEmpty();
        assertThat(repository.findByIdAndOrgId(monitorInB.getId(), orgB.getId())).isPresent();
    }

    @Test
    void uniqueConstraintPreventsDuplicateNameWithinAnOrganization() {
        var org = em.persist(TestData.organization().build());
        em.persist(TestData.monitor().withOrganization(org).withName("api").build());
        em.flush();

        assertThatThrownBy(() -> {
            em.persist(TestData.monitor().withOrganization(org).withName("api").build());
            em.flush();                                // ★ flush REQUIRED to hit the constraint
        }).isInstanceOf(PersistenceException.class);
    }

    @Test
    void entityGraphAvoidsNPlusOne() {
        var org = em.persist(TestData.organization().build());
        for (int i = 0; i < 10; i++)
            em.persist(TestData.monitor().withOrganization(org).withName("m" + i).build());
        em.flush();
        em.clear();

        var stats = em.getEntityManager().unwrap(Session.class).getSessionFactory().getStatistics();
        stats.clear();

        var monitors = repository.findByOrganizationIdWithOrg(org.getId());
        monitors.forEach(m -> m.getOrganization().getName());       // would trigger N+1

        assertThat(stats.getPrepareStatementCount()).isEqualTo(1);   // ★ Lesson 24's regression test
    }
}
```

**Three techniques worth internalizing:**

1. **`em.flush(); em.clear();`** before asserting. Without `clear()`, `findById` hits the first-level cache and returns your in-memory object — **so your test passes even when the SQL is wrong.** This is the #1 reason a repository test gives false confidence. (Lesson 20.)
2. **`em.flush()` to trigger constraints.** Without it, the INSERT hasn't reached the database, so the constraint violation never happens and your test passes wrongly.
3. **The query-count assertion** is your N+1 regression guard, now in the right place.

**`@DataJpaTest` is `@Transactional` and rolls back after each test.** Convenient — but know the caveat:

```java
@DataJpaTest
class Trap {
    @Test
    void thisPassesButIsMisleading() {
        repository.save(monitor);
        assertThat(repository.findById(monitor.getId())).isPresent();
        // ⚠️ The INSERT may not have reached the DB at all — this could be a cache hit.
    }
}
```
**And the rollback means you can't test anything involving real commit semantics** — `@TransactionalEventListener(AFTER_COMMIT)`, `REQUIRES_NEW` isolation, or database triggers on commit. For those, use `@SpringBootTest` with explicit cleanup instead of rollback.

---

## 4. Testcontainers — real infrastructure, in tests

### Why not H2

Using H2 as a Postgres stand-in is the most common testing shortcut, and it produces a specific, nasty failure mode: **tests pass, production breaks.**

| Feature | H2 | Postgres |
|---|---|---|
| `jsonb` | ❌ (no real support) | ✅ |
| Window functions, CTEs | partial | ✅ |
| `FOR UPDATE SKIP LOCKED` | ❌ | ✅ |
| `percentile_cont`, `FILTER` | ❌ | ✅ |
| Partial / covering indexes | ❌ | ✅ |
| Actual query plans | different | the ones you'll get |
| Sequence and locking behaviour | different | |
| Row-Level Security | ❌ | ✅ |
| Error messages / SQLStates | different | |

**Every one of those is something Vigil uses.** An H2-based suite would be untestable for `jsonb` config, `SKIP LOCKED` job claiming, `percentile_cont` for p95, RLS for tenant isolation, and the partial index for the scheduler. **You'd be testing a different database than you ship on.**

**Testcontainers runs the real thing in Docker.** State this as: *"I use Testcontainers rather than H2 because half of what I need to test — jsonb, SKIP LOCKED, window functions, RLS, actual query plans, real constraint errors — doesn't exist or behaves differently in H2. Testing against a different database than production is testing the wrong thing."*

### The modern setup (Boot 3.1+ makes this trivial)

```xml
<dependency><groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-testcontainers</artifactId><scope>test</scope></dependency>
<dependency><groupId>org.testcontainers</groupId>
  <artifactId>postgresql</artifactId><scope>test</scope></dependency>
<dependency><groupId>org.testcontainers</groupId>
  <artifactId>junit-jupiter</artifactId><scope>test</scope></dependency>
```

```java
@TestConfiguration(proxyBeanMethods = false)
public class TestcontainersConfig {

    @Bean
    @ServiceConnection                              // ★ Boot 3.1+: auto-configures spring.datasource.*
    PostgreSQLContainer<?> postgres() {
        return new PostgreSQLContainer<>("postgres:16-alpine");
    }

    @Bean
    @ServiceConnection
    GenericContainer<?> redis() {
        return new GenericContainer<>("redis:7-alpine").withExposedPorts(6379);
    }
}
```

**`@ServiceConnection` is the feature that made Testcontainers effortless.** Before it you wrote `@DynamicPropertySource` blocks wiring URLs, usernames and passwords by hand. Now Boot detects the container type and configures the connection itself. **Knowing that `@DynamicPropertySource` was the old way is a nice currency signal.**

```java
// The old way, still needed for containers without a @ServiceConnection factory:
@DynamicPropertySource
static void props(DynamicPropertyRegistry registry) {
    registry.add("spring.datasource.url", postgres::getJdbcUrl);
    registry.add("vigil.slack.base-url", () -> "http://localhost:" + wiremock.getPort());
}
```

### Container reuse — the performance essential

**By default each context gets fresh containers**, which is slow. Two fixes:

**1. A singleton container in a shared base class** — the classic pattern:
```java
@SpringBootTest
@ActiveProfiles("test")
public abstract class AbstractIntegrationTest {

    static final PostgreSQLContainer<?> POSTGRES =
            new PostgreSQLContainer<>("postgres:16-alpine")
                    .withReuse(true);               // ★ survives across JVM runs

    static { POSTGRES.start(); }                    // ★ static init: started ONCE per JVM
                                                    //   JVM shutdown + Ryuk handle cleanup

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", POSTGRES::getJdbcUrl);
        r.add("spring.datasource.username", POSTGRES::getUsername);
        r.add("spring.datasource.password", POSTGRES::getPassword);
    }
}
```

**2. `withReuse(true)` plus `~/.testcontainers.properties`:**
```properties
testcontainers.reuse.enable=true
```
The container survives *between test runs*, so your second local run starts in milliseconds. **Enable this locally; leave it off in CI** (where you want a clean slate and the container dies with the runner anyway).

> **Fun fact:** Testcontainers starts a tiny sidecar container called **Ryuk** ("the reaper") that watches the Docker socket and destroys your test containers if the JVM dies without cleaning up. It's why a killed test run doesn't leave orphaned Postgres containers eating your laptop's RAM. `withReuse(true)` deliberately opts out of Ryuk for that container — which is why reused containers need manual cleanup occasionally.

### Test data isolation — the thing that causes flakiness

Shared containers mean shared state, and **tests that pass alone but fail in a suite** are the result.

| Strategy | How | Verdict |
|---|---|---|
| **`@Transactional` rollback** | Boot rolls back after each test | ✅ Fastest. ⚠️ Can't test commit semantics or multi-threaded behaviour |
| **Truncate between tests** | `TRUNCATE ... CASCADE` in `@AfterEach` | ✅ Reliable, works with commits. Slightly slower |
| **Unique data per test** | Every test creates its own org with a random slug | ✅ **Best — enables parallel execution** |
| `@DirtiesContext` | Rebuild the context | ❌ Very slow |

```java
// The truncate helper — reliable and cheap
@Component
public class DatabaseCleaner {
    private final JdbcClient jdbc;

    public void clean() {
        jdbc.sql("""
            TRUNCATE TABLE check_result, incident, monitor, membership,
                           refresh_token, app_user, organization
            RESTART IDENTITY CASCADE
            """).update();
    }
}
```

**The "unique data per test" strategy deserves emphasis:** if every test creates its own organization with a random slug and works only inside it, tests never collide and you can run them **in parallel** — which is often a bigger win than any single optimization. It also mirrors production, where tenants are genuinely isolated. **This is the strategy that scales.**

---

## 5. Full-stack tests with a real HTTP server

```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
class MonitorApiIntegrationTest extends AbstractIntegrationTest {

    @Autowired private TestRestTemplate rest;          // or WebTestClient / RestClient
    @LocalServerPort private int port;

    @Test
    void createThenFetchAMonitor() {
        var token = authHelper.loginAs("owner@acme.test");
        var headers = new HttpHeaders();
        headers.setBearerAuth(token);

        var create = new HttpEntity<>(
                new CreateMonitorRequest("api", "https://example.com", 60, HTTP, null), headers);

        var created = rest.postForEntity("/api/v1/monitors", create, MonitorResponse.class);

        assertThat(created.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(created.getHeaders().getLocation()).isNotNull();

        var fetched = rest.exchange(created.getHeaders().getLocation(), HttpMethod.GET,
                                    new HttpEntity<>(headers), MonitorResponse.class);
        assertThat(fetched.getBody().name()).isEqualTo("api");
    }
}
```

**`MOCK` vs `RANDOM_PORT` — the distinction interviewers ask about:**

| | `WebEnvironment.MOCK` (default) | `RANDOM_PORT` / `DEFINED_PORT` |
|---|---|---|
| Server | None — MockMvc simulates the servlet container | **Real embedded Tomcat** |
| Client | `MockMvc` | `TestRestTemplate` / `WebTestClient` / `RestClient` |
| Speed | Faster | Slower |
| Tests real HTTP | ❌ | ✅ real sockets, real serialization, real filters |
| Same thread as the test | ✅ so `@Transactional` rollback works | ❌ **different thread → rollback does NOT apply** |

**That last row is a genuine trap.** With `RANDOM_PORT`, the request runs on a Tomcat thread with its own transaction, so the test's `@Transactional` rollback has no effect on what the request committed. **You must clean up explicitly.** People spend hours on this.

---

## 6. Testing security

```java
@SpringBootTest
@AutoConfigureMockMvc
class SecurityIntegrationTest extends AbstractIntegrationTest {

    @Autowired private MockMvc mockMvc;

    @Test
    void unauthenticatedRequestIs401() throws Exception {
        mockMvc.perform(get("/api/v1/monitors")).andExpect(status().isUnauthorized());
    }

    @Test
    @WithMockUser(username = "u@x.io", authorities = {"monitors:read"})
    void readPermissionCannotDelete() throws Exception {
        mockMvc.perform(delete("/api/v1/monitors/{id}", UUID.randomUUID()))
               .andExpect(status().isForbidden());
    }

    @Test
    void crossTenantAccessReturns404() throws Exception {
        var orgA = fixtures.createOrgWithUser("a@x.io");
        var orgB = fixtures.createOrgWithUser("b@x.io");
        var monitorInB = fixtures.createMonitor(orgB);

        mockMvc.perform(get("/api/v1/monitors/{id}", monitorInB.getId())
                        .header(AUTHORIZATION, "Bearer " + authHelper.tokenFor("a@x.io")))
               .andExpect(status().isNotFound());       // ★ 404, not 403 — Lesson 28
    }

    @Test
    void listEndpointNeverReturnsAnotherTenantsRows() throws Exception {
        // ...create data in both orgs...
        mockMvc.perform(get("/api/v1/monitors").header(AUTHORIZATION, "Bearer " + tokenForA))
               .andExpect(jsonPath("$.content[*].id",
                          not(hasItem(monitorInB.getId().toString()))));
    }
}
```

**Test helpers:**
```java
@WithMockUser(username = "u", roles = {"ADMIN"})          // roles → adds ROLE_ prefix
@WithMockUser(authorities = {"monitors:write"})            // literal authorities
@WithAnonymousUser
@WithUserDetails("owner@acme.test")                        // loads via your UserDetailsService
// custom: @WithMockVigilUser(org = "acme", role = OWNER) via WithSecurityContextFactory

.with(csrf())                                              // add a CSRF token
.with(user("u").authorities(...))                          // per-request user
.with(jwt().jwt(j -> j.claim("org", orgId)))               // for a resource server
```

**The cross-tenant tests in this section are the most valuable tests in the whole project.** They're a security control expressed as code, and a reviewer who finds them draws immediate conclusions about how you work. **Write one per endpoint.**

---

## 7. Testing outbound HTTP with WireMock

Vigil's whole job is calling other people's URLs. You need to test timeouts, 500s, and slow responses **without depending on the internet.**

```xml
<dependency><groupId>org.wiremock</groupId>
  <artifactId>wiremock-standalone</artifactId><scope>test</scope></dependency>
```

```java
@SpringBootTest
class HttpCheckExecutorTest {

    static WireMockServer wireMock = new WireMockServer(options().dynamicPort());

    @BeforeAll static void start() { wireMock.start(); }
    @AfterAll  static void stop()  { wireMock.stop(); }
    @AfterEach void reset()        { wireMock.resetAll(); }

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("vigil.test.base-url", () -> "http://localhost:" + wireMock.port());
    }

    @Autowired private HttpCheckExecutor executor;

    @Test
    void reportsUpWhenTheTargetReturns200() {
        wireMock.stubFor(get("/health").willReturn(ok()));

        var outcome = executor.execute(monitorFor("/health", 200));

        assertThat(outcome.status()).isEqualTo(UP);
        assertThat(outcome.responseTimeMs()).isBetween(0L, 5000L);
    }

    @Test
    void reportsDownOnUnexpectedStatus() {
        wireMock.stubFor(get("/health").willReturn(serverError()));
        assertThat(executor.execute(monitorFor("/health", 200)).status()).isEqualTo(DOWN);
    }

    @Test
    void reportsDownAndDoesNotHangWhenTheTargetIsSlow() {
        wireMock.stubFor(get("/slow").willReturn(ok().withFixedDelay(30_000)));  // 30s

        long start = System.currentTimeMillis();
        var outcome = executor.execute(monitorFor("/slow", 200));
        long elapsed = System.currentTimeMillis() - start;

        assertThat(outcome.status()).isEqualTo(DOWN);
        assertThat(outcome.error()).containsIgnoringCase("timeout");
        assertThat(elapsed).isLessThan(15_000);      // ★★ PROVES the read timeout fires
    }

    @Test
    void doesNotFollowARedirectToAPrivateAddress() {
        wireMock.stubFor(get("/redirect")
                .willReturn(temporaryRedirect("http://169.254.169.254/latest/meta-data/")));

        assertThat(executor.execute(monitorFor("/redirect", 200)).error())
                .containsIgnoringCase("blocked");    // ★ SSRF defence, tested — Lesson 28
    }

    @Test
    void reportsDownOnConnectionRefused() {
        var outcome = executor.execute(monitorFor("http://localhost:1/nothing", 200));
        assertThat(outcome.status()).isEqualTo(DOWN);
    }
}
```

**The slow-response test is the most valuable one here.** Timeout configuration is trivially easy to get wrong (or to have silently overridden), and **an unenforced timeout is exactly what causes thread-pool exhaustion and cascading outages** (Lessons 14, 19, 23). Almost nobody tests that their timeouts actually fire. **Doing so — and being able to say you do — is a standout answer.**

**The redirect-to-metadata test is the other standout:** it's a security control (SSRF) verified by an automated test. Naming it in an interview lands hard.

`MockRestServiceServer` (via `@RestClientTest`) is the lighter-weight alternative — but it *replaces* the HTTP layer, so it can't test real timeouts, TLS, or connection failures. **WireMock runs a real server, so it can.** That distinction is the reason to prefer it here.

---

## 8. Build & break — Vigil's integration test suite

**1. The base class.** `AbstractIntegrationTest` with a singleton Postgres container, `@ActiveProfiles("test")`, and a `DatabaseCleaner` in `@AfterEach`. **Every integration test extends it, so they all share one context.**

**2. The context-loads test.** `@SpringBootTest` with an empty `@Test`. Then deliberately break wiring (remove a `@Component` a service depends on) and confirm it fails. **That test earns its keep on day one.**

**3. `@DataJpaTest` for repositories** against Testcontainers Postgres:
- The tenant-scoped query returns nothing for the wrong org
- The unique constraint fires (remember `em.flush()`)
- `jsonb` config round-trips — **and query it with `config->>'key'` to prove it's real JSON, which H2 couldn't do**
- `FOR UPDATE SKIP LOCKED` gives two threads disjoint batches
- The `@EntityGraph` query count is exactly 1

**4. ⭐ Prove the `em.clear()` trap.** Write a repository test asserting a saved entity can be found. Make it pass. Then **deliberately break the entity's column mapping** (rename a `@Column` to something nonexistent) and confirm the test **still passes** — because it never hit the database. Then add `em.flush(); em.clear();` and watch it correctly fail. **This is the most important exercise in the lesson: you've just discovered that a whole category of repository test can be worthless.**

**5. `@WebMvcTest` for controllers:** success, validation 400 with the `ProblemDetail` shape, 404, 401, 403, and a `$.passwordHash doesNotExist` leak check. Confirm the tests run in **milliseconds** — no database involved.

**6. Full-stack tests** with `RANDOM_PORT` and `TestRestTemplate` for the create→fetch→update→delete lifecycle. **Then discover the transaction trap:** annotate the test `@Transactional`, create a monitor via HTTP, and confirm the row **survives** after the test. Explain why to yourself.

**7. The security suite** from §5, including cross-tenant 404 tests for every endpoint. **Then delete a tenant filter from one repository method and confirm the test catches it.**

**8. WireMock tests** from §6, including the slow-response timeout proof and the SSRF redirect block.

**9. Make the suite fast.** Turn on `logging.level.org.springframework.test.context.cache=DEBUG` and count your contexts. Then reduce the count by consolidating `properties` and `@MockitoBean` usage into the base class. **Record the before/after suite time** — that number is a great thing to mention.

**10. Enable parallel execution.** `src/test/resources/junit-platform.properties`:
```properties
junit.jupiter.execution.parallel.enabled=true
junit.jupiter.execution.parallel.mode.default=concurrent
junit.jupiter.execution.parallel.mode.classes.default=concurrent
```
Run the suite. **Things will break** — that's the point: it surfaces every hidden shared-state dependency. Fix them by giving each test its own organization with a random slug. **Record the speedup.**

### Explain out loud (60s)

1. How the context cache works, and the two habits that quietly multiply your contexts.
2. Five things H2 can't test that your schema actually uses.
3. Why `em.flush()` and `em.clear()` are the difference between a real repository test and a worthless one.
4. The `RANDOM_PORT` plus `@Transactional` trap, in terms of threads.
5. Why testing your timeouts matters more than testing the happy path of an outbound call.

---

## 9. Production rules

| Do / Never | Why |
|---|---|
| **Keep one `@SpringBootTest` with an empty test body** | It proves the whole object graph constructs — every bean resolves, every property binds — so a wiring error fails the build instead of the deploy |
| **Consolidate `properties` and `@MockitoBean` into a shared base class** | The context cache keys on configuration, so every distinct combination is another full startup; this is the usual answer to "why is our suite slow?" |
| **Use Testcontainers, not H2** | `jsonb`, `SKIP LOCKED`, window functions, RLS, real query plans and real SQLStates either don't exist or behave differently — H2 tests a database you don't ship |
| **Use `@ServiceConnection` rather than hand-wiring `@DynamicPropertySource`** | Boot detects the container and configures the connection itself, which removes the most error-prone part of the setup |
| **Always `em.flush()` and `em.clear()` in repository tests** | Without it the assertion reads from the persistence context, so the test passes even when the column mapping is wrong and no SQL ever ran |
| **Prefer unique data per test over shared fixtures** | A random org per test means tests never collide, which is what makes parallel execution possible and mirrors real tenant isolation |
| **Know that `@Transactional` rollback does not apply with `RANDOM_PORT`** | The request runs on a Tomcat thread with its own transaction, so anything it commits survives your test — clean up explicitly |
| **Don't use `@DataJpaTest` for commit semantics** | It rolls back, so `AFTER_COMMIT` listeners, `REQUIRES_NEW` isolation and commit triggers can't be observed at all |
| **Assert that sensitive fields are absent from responses** | `$.passwordHash` doesNotExist is what stops a future field addition from quietly publishing a secret |
| **Write a cross-tenant test per endpoint** | It is a security control expressed as code, and it is the single most valuable set of tests in a multi-tenant project |
| **Test that your timeouts actually fire** | An unenforced timeout is what causes thread-pool exhaustion and cascading outages, and nothing else in your suite would notice |
| **Use `.andDo(print())` before guessing at a 400** | It prints the resolved handler, the request and the whole response body — usually the entire answer |

---

## 10. Interview traps

**Q1. What does `@SpringBootTest` do, and what's the cost?**
Loads the full application context (all auto-configuration, all beans) — 2–10 seconds. Use it for full-stack behaviour; use slices (`@WebMvcTest`, `@DataJpaTest`) when you only need one layer. An empty `contextLoads` test is genuinely valuable: it proves the whole object graph and all configuration binding is valid.

**Q2. How does Spring cache test contexts, and why does it matter?**
Contexts are cached and reused across test classes, keyed on the configuration: classes loaded, profiles, property sources, `@MockitoBean` definitions, customizers. **Every distinct configuration builds another context.** Scattered `properties = {...}` and varied `@MockitoBean` sets are the usual cause of slow suites. `@DirtiesContext` evicts the cache and should be a last resort.

**Q3. Why not use H2 for tests?**
It's a different database. No real `jsonb`, no `FOR UPDATE SKIP LOCKED`, no `percentile_cont`/`FILTER`, no partial indexes, no RLS, different query plans, different error codes and SQLStates. Tests pass and production breaks. Testcontainers runs the actual Postgres version you deploy.

**Q4. What is Testcontainers and what does `@ServiceConnection` do?**
A library that starts real infrastructure in Docker for tests. `@ServiceConnection` (Boot 3.1+) makes Boot auto-configure connection properties from a container bean, replacing hand-written `@DynamicPropertySource` blocks.

**Q5. How do you keep Testcontainers-based tests fast?**
A singleton container started in a static initializer in a shared base class (so it starts once per JVM), `withReuse(true)` locally so it survives between runs, one shared Spring context via that base class, and data isolation by transaction rollback, truncation, or per-test unique tenants.

**Q6. `@WebMvcTest` vs `@SpringBootTest`?**
`@WebMvcTest` loads only the MVC layer — controllers, advice, converters, filters — with services mocked via `@MockitoBean` and no database. Fast, and ideal for HTTP mapping, validation, serialization, error contracts and security rules. `@SpringBootTest` loads everything for true end-to-end behaviour.

**Q7. What's the trap with `@SpringBootTest(webEnvironment = RANDOM_PORT)` and `@Transactional`?**
The request executes on a Tomcat thread with its own transaction, so the test's rollback doesn't undo what the request committed. Data persists between tests. You must clean up explicitly (truncate) rather than relying on rollback.

**Q8. Why must you call `em.flush()` and `em.clear()` in repository tests?**
`flush()` sends the SQL, so constraints and generated values actually happen. `clear()` empties the persistence context, so the subsequent query hits the **database** rather than the first-level cache. Without `clear()`, a repository test can pass even when the SQL or mapping is wrong.

**Q9. How do you test security rules?**
Slice or full-stack tests with `@WithMockUser`/`@WithUserDetails`/`.with(jwt())`, asserting 401 for unauthenticated, 403 for insufficient authority, and **404 for cross-tenant access**. The cross-tenant tests are the highest-value tests in a multi-tenant application — one per endpoint.

**Q10. How do you test outbound HTTP calls?**
WireMock — a real HTTP server you control — so you can test 200s, 500s, malformed bodies, connection refusals, redirects, and **fixed delays that prove your read timeout actually fires**. `MockRestServiceServer` is lighter but replaces the HTTP layer, so it can't test real timeouts or connection failures.

**Q11. Why is testing that your timeouts fire so important?**
An unenforced timeout means a slow dependency holds threads and connections indefinitely, exhausting the pool and turning someone else's outage into yours. Timeout config is easy to get wrong or silently override, and almost nobody tests it — so a `withFixedDelay` test asserting elapsed time is a genuinely differentiating practice.

**Q12. `@MockBean` or `@MockitoBean`?**
`@MockitoBean` — `@MockBean` was deprecated in Boot 3.4. Same purpose: replace a context bean with a Mockito mock (and note each distinct set forks a new cached context).

---

## 11. Retention check

1. Name four things that fork a new cached Spring context.
2. Give five Postgres features H2 can't test, and tie each to something Vigil does.
3. Why can a repository test pass with a broken column mapping, and what's the fix?
4. What breaks about `@Transactional` rollback under `RANDOM_PORT`, and why?
5. Which single WireMock test would you write first, and what production failure does it prevent?
6. What's the strongest test-data isolation strategy, and what does it unlock?

---

**Next:** [Lesson 31 — Advanced testing & CI strategy](31-advanced-testing-strategy.md) — ArchUnit, contract tests, mutation testing, and eliminating flakiness.

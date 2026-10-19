# Lesson 12 — Configuration, profiles & secrets

> **Why this lesson exists:** Config is where "works on my machine" is born. It's also a favourite interview area because it has a precise, memorizable answer (the precedence order) *and* a judgment component (profiles vs properties, secrets handling) that separates people who've shipped from people who haven't.

**Time:** ~60 minutes · **Prereq:** Lesson 11

---

## 1. The idea in one sentence

> **One build artifact must run in every environment, differing only in what the environment hands it — so config is everything that varies between deployments, and the test of whether you've got it right is whether you could open-source the repository today without leaking a single credential.**

Both halves of this lesson fall out of that. Precedence exists so the deployment can override the jar; profiles are dangerous precisely when they make environments run *different code* instead of the same code with different values.

---

## 2. The 12-Factor principle behind all of this

> **Config is everything that varies between deployments.** It must live in the *environment*, not the code.

The test: **could you open-source your repository right now without leaking a single credential?** If not, your config is wrong.

This isn't dogma — it's what makes one build artifact promotable. The exact `vigil.jar` that passed tests in CI should run in staging and production with **no rebuild**, differing only in environment. If you have to rebuild per environment, you're not testing what you ship. That argument is the *why* behind everything below, and it's the framing that impresses in interviews.

---

## 3. Property sources and precedence

Spring's `Environment` merges many sources into one lookup. **Precedence, highest wins** (this is the memorizable list):

```
 1. Devtools global settings (~/.config/spring-boot)         [dev only]
 2. @TestPropertySource / @SpringBootTest(properties=...)     [tests only]
 3. Command-line arguments               --server.port=9090
 4. SPRING_APPLICATION_JSON              (inline JSON in an env var)
 5. ServletConfig / ServletContext init params
 6. JNDI attributes
 7. Java System properties               -Dserver.port=9090
 8. OS environment variables             SERVER_PORT=9090
 9. Profile-specific OUTSIDE the jar     ./config/application-prod.yml
10. Profile-specific INSIDE the jar      application-prod.yml
11. application.properties/yml OUTSIDE the jar    ./application.yml
12. application.properties/yml INSIDE the jar     src/main/resources/application.yml
13. @PropertySource on @Configuration
14. SpringApplication.setDefaultProperties()
```

**What you actually need to remember for interviews and for real work:**

> **Command-line args > environment variables > profile-specific file > base file.**
> **Outside the JAR beats inside the JAR.**
> **Profile-specific beats non-profile-specific.**

Two rules explain the whole table: *more specific wins*, and *closer to the deployment wins*.

**The practical consequence, and it's the one that matters:**
```bash
# You can override ANY property at deploy time without rebuilding.
java -jar vigil.jar --spring.datasource.url=jdbc:postgresql://prod-db:5432/vigil

# Or, more typically in containers:
docker run -e SPRING_DATASOURCE_URL=jdbc:postgresql://prod-db:5432/vigil vigil:1.0
```

### Relaxed binding — one property, many spellings

```
vigil.notification.slack.webhook-url      ← kebab-case: canonical, use this in files
vigil.notification.slack.webhookUrl       ← camelCase
vigil.notification.slack.webhook_url
VIGIL_NOTIFICATION_SLACK_WEBHOOKURL       ← env var form
VIGIL_NOTIFICATION_SLACK_WEBHOOK_URL      ← env var form (the one you'll use)
```

All bind to the same target. **The env-var rule: uppercase, and replace `.` and `-` with `_`.** This is essential in Docker/Kubernetes, where env vars are the primary config channel and dots aren't always legal in env var names.

Lists and maps in env vars:
```bash
VIGIL_ALLOWED_ORIGINS_0=https://app.vigil.io
VIGIL_ALLOWED_ORIGINS_1=https://status.vigil.io
VIGIL_LIMITS_FREE=10                     # → vigil.limits.free  (a Map<String,Integer>)
```

**Important limitation:** relaxed binding applies to `@ConfigurationProperties`, **not** to `@Value`. `@Value("${vigil.webhookUrl}")` needs an exact match. That asymmetry surprises people and is a good trap question.

---

## 4. `application.yml` — structure and the multi-document trick

YAML vs properties is mostly taste. YAML wins for nesting and lists; properties wins for being un-breakable by whitespace. Most teams use YAML.

```yaml
spring:
  application:
    name: vigil
  datasource:
    url: jdbc:postgresql://localhost:5432/vigil
    username: vigil
    password: ${DB_PASSWORD}                # ← from env; NEVER a literal here
    hikari:
      maximum-pool-size: 10
  jpa:
    hibernate:
      ddl-auto: validate                    # ← 'validate' in prod. Never 'update'. (Lesson 24)
    open-in-view: false                     # ← turn this OFF. Always. (Lesson 20)
  threads:
    virtual:
      enabled: true                         # Java 21+

server:
  port: 8080
  shutdown: graceful
  error:
    include-message: always
    include-stacktrace: never               # never leak stack traces to clients

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      probes:
        enabled: true                       # /health/liveness and /health/readiness

logging:
  level:
    root: INFO
    com.vigil: DEBUG

vigil:                                       # ← your own namespace
  checks:
    default-interval: 60s
    timeout: 10s
    max-concurrent: 50
  notification:
    type: SLACK
```

**Two lines in there are load-bearing production advice** and worth flagging now so you recognize them later: `ddl-auto: validate` (never let Hibernate alter your production schema) and `open-in-view: false` (the default is `true` and it's a well-known performance trap — Lesson 20).

### Multi-document YAML — profiles in one file

```yaml
# Shared by all profiles
spring:
  application:
    name: vigil
vigil:
  checks:
    default-interval: 60s

---
spring:
  config:
    activate:
      on-profile: dev
  datasource:
    url: jdbc:postgresql://localhost:5432/vigil_dev
  jpa:
    show-sql: true
logging:
  level:
    com.vigil: DEBUG

---
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: ${DB_URL}
    password: ${DB_PASSWORD}
logging:
  level:
    root: WARN
    com.vigil: INFO
```

`---` separates documents; `spring.config.activate.on-profile` gates each one. (The old `spring.profiles: dev` key is deprecated — recognizing that dates a codebase.)

Also useful:
```yaml
spring:
  config:
    activate:
      on-cloud-platform: kubernetes         # only when running on K8s
    import:
      - optional:file:./local.yml           # 'optional:' = don't fail if absent
      - configtree:/run/secrets/            # ← reads K8s/Docker secret files!
```

**`configtree:` is genuinely useful and little known**: Kubernetes mounts secrets as files (`/run/secrets/db-password` containing the value), and `configtree` binds each file to a property named after it. That's how you consume K8s secrets without a sidecar or an SDK. Mentioning it in a "how do you handle secrets in Kubernetes?" answer is a strong signal.

---

## 5. Reading config: `@Value` vs `@ConfigurationProperties`

### `@Value` — for one-off scalars

```java
@Service
public class CheckService {
    @Value("${vigil.checks.timeout:10s}")            // ':' provides a default
    private Duration timeout;

    // Better — constructor injection, so it's final and testable:
    private final int maxConcurrent;
    public CheckService(@Value("${vigil.checks.max-concurrent}") int maxConcurrent) {
        this.maxConcurrent = maxConcurrent;
    }
}
```

`@Value` also supports SpEL:
```java
@Value("#{systemProperties['user.timezone']}")
@Value("#{${vigil.checks.timeout-seconds} * 1000}")
@Value("#{'${vigil.allowed-origins}'.split(',')}")
```
**Use SpEL in `@Value` sparingly** — it's a string evaluated at runtime, so a mistake is a startup failure at best and a subtle wrong value at worst.

### `@ConfigurationProperties` — the professional default

```java
@ConfigurationProperties(prefix = "vigil.checks")
@Validated
public record CheckProperties(
        @DefaultValue("60s")  Duration defaultInterval,
        @DefaultValue("10s")  Duration timeout,
        @Min(1) @Max(500) @DefaultValue("50") int maxConcurrent,
        @NotEmpty List<String> allowedSchemes,
        @Valid Retry retry
) {
    public record Retry(
            @DefaultValue("3") @Min(0) int maxAttempts,
            @DefaultValue("2s") Duration backoff
    ) {}
}
```

Register it:
```java
@SpringBootApplication
@ConfigurationPropertiesScan          // scans for @ConfigurationProperties classes
public class VigilApplication { }
```
(or `@EnableConfigurationProperties(CheckProperties.class)` on a config class.)

Inject it like any bean:
```java
@Service
@RequiredArgsConstructor
public class CheckService {
    private final CheckProperties props;      // typed, validated, autocompleted
}
```

### Why `@ConfigurationProperties` wins — six reasons

| | `@Value` | `@ConfigurationProperties` |
|---|---|---|
| Type safety | String → coerced, per field | Whole object, typed |
| Relaxed binding | ❌ exact key only | ✅ all forms including env vars |
| Validation | ❌ | ✅ `@Validated` + Bean Validation → **fails at startup** |
| Nested/complex structure | ❌ painful | ✅ nested records, `List`, `Map` |
| IDE autocomplete | partial | ✅ with the config processor |
| Testability | needs a context or reflection | plain constructor call |
| Appears in `/actuator/configprops` | ❌ | ✅ |
| Metadata/docs | ❌ | ✅ Javadoc → IDE tooltip |

**The validation point is the strongest argument.** `@Validated` turns a typo into a startup failure with a precise message:

```
Binding to target CheckProperties failed:
    Property: vigil.checks.maxConcurrent
    Value: "1000"
    Reason: must be less than or equal to 500
```

That's found at deploy time, not at 3am. **"Fail fast on invalid configuration" is a genuinely senior instinct** — a service that refuses to start is vastly better than one that starts with a nonsense value.

Also note the automatic `Duration`/`DataSize` conversion — `60s`, `5m`, `2h`, `500ms`, `10MB` all bind. That's a small quality-of-life win people miss and end up parsing strings by hand.

### Binding rules gotcha

Records (and constructor binding generally) are immutable and require all values at construction — hence `@DefaultValue`. Class-based binding with setters is mutable and rebindable (needed for `@RefreshScope` with Spring Cloud Config). **Use records unless you need runtime refresh.**

---

## 6. Profiles

```bash
--spring.profiles.active=prod
SPRING_PROFILES_ACTIVE=prod,kubernetes
-Dspring.profiles.active=dev
```

```java
@Profile("dev")            @Profile("!prod")         @Profile("dev | test")
@Profile({"dev","test"})   // OR
```

Also:
```yaml
spring:
  profiles:
    group:
      prod: [prod-db, prod-cache, metrics]     # one profile activating several
```

### The single most important opinion in this lesson

**Do not use profiles as your primary configuration mechanism.**

```java
// ❌ The trap
@Service
public class PaymentService {
    @Profile("prod") public void charge() { realGateway.charge(); }
    @Profile("!prod") public void charge() { log.info("pretend charge"); }
}
```

Now the code path that runs in production **is the one you never test**. Every environment runs different logic, so "it worked in staging" means nothing. This is how the worst class of production bug is born.

**✅ The right approach: one code path, different values.**
```yaml
# same code everywhere; only configuration differs
vigil:
  payments:
    gateway-url: ${PAYMENT_GATEWAY_URL}
    mode: ${PAYMENT_MODE:sandbox}
```

**Where profiles genuinely belong** (structurally different *wiring*, not different logic):
- `test` — swap in an embedded/Testcontainers datasource
- `local` — a `NoOpNotificationChannel` so you don't spam a real Slack
- `kubernetes` / `cloud` — enable platform-specific beans (probes, config sources)
- Choosing between real implementations of a genuinely pluggable port (S3 vs local filesystem storage)

**And the rule that follows:** if the *number* of beans differs between environments, a profile is reasonable. If only a *value* differs, use a property. Being able to state that distinction crisply is worth real points.

**Two more traps:**
1. **No active profile ≠ no profile.** Spring activates `default`, so `@Profile("default")` beans load. Surprising when you meant "always."
2. **`spring.profiles.active` in `application.yml` is fragile.** It's overridden by any environment-supplied value, and hardcoding `prod` in a committed file has caused real incidents. Set profiles from the environment.

---

## 7. Secrets — the part that gets people fired

### Never do this

```yaml
spring:
  datasource:
    password: SuperSecret123        # ❌ now in git, forever, in every clone and fork
```

**Git history is permanent.** Deleting the line doesn't remove it — you need history rewriting *and* credential rotation, because you must assume it's compromised. Automated scanners crawl public GitHub for exactly this within minutes of a push.

### The progression, worst to best

**Level 0 — committed plaintext.** Never.

**Level 1 — env vars + a gitignored local file.**
```yaml
spring:
  datasource:
    password: ${DB_PASSWORD}       # fails loudly at startup if unset — good
```
```
# .gitignore
application-local.yml
.env
```
Fine for local development and small deployments. Weakness: env vars are visible in `/proc/<pid>/environ`, in `docker inspect`, in crash dumps, and in accidental `System.getenv()` logging.

**Level 2 — mounted secret files.** Kubernetes Secrets or Docker secrets mounted as files, read via `configtree:` (§3). Better than env vars: file permissions apply, and they don't leak into process listings or child processes.

**Level 3 — a secret manager.** HashiCorp Vault, AWS Secrets Manager, Azure Key Vault, GCP Secret Manager.
```yaml
spring:
  config:
    import: "vault://secret/vigil"      # spring-cloud-vault
```
Buys you: central audit trail, automatic rotation, fine-grained access policy, and short-lived dynamic credentials (Vault can issue a per-pod database user valid for one hour). **This is the answer to give when asked "how would you handle secrets in production?"** — and mention *rotation*, because that's the part people forget.

**Level 4 — no long-lived secrets at all.** Workload identity: the platform attests the workload's identity (IAM roles for service accounts, Kubernetes projected tokens, cloud instance metadata) and the app exchanges that for short-lived credentials. **The best secret is one that doesn't exist.** Saying this shows you're thinking about the actual threat model rather than reciting tools.

### Encrypted-in-repo, as a pragmatic middle ground

`git-crypt`, `SOPS`, `sealed-secrets`, or Jasypt (`ENC(...)` values in properties). Reasonable when a full secret manager is overkill — but you still have to protect the *decryption* key, so it moves the problem rather than solving it. Say that if you propose it.

### Vigil's approach (and what you'll implement)
- Local: `application-local.yml`, gitignored, with a committed `application-local.yml.example` template
- CI: GitHub Actions encrypted secrets
- Production: env vars from the platform's secret store, or `configtree:` from mounted files
- **Zero secrets in the repo, verified by a `gitleaks` pre-commit hook**

That last item is worth doing for real. A pre-commit secret scanner has saved a lot of careers.

---

## 8. Spring Cloud Config — centralized configuration (know the concept)

With 30 microservices, "change the log level everywhere" becomes a deployment project. A **config server** centralizes it:

```
                     ┌──────────────────┐
                     │  Config Server   │──── git repo (versioned, reviewable, auditable)
                     └────────┬─────────┘
        ┌─────────────────────┼─────────────────────┐
   monitor-svc          notification-svc        api-gateway
   (fetches config at startup, and on refresh)
```

```yaml
spring:
  config:
    import: "optional:configserver:http://config-server:8888"
  application:
    name: vigil-monitor        # ← the server serves {name}-{profile}.yml
```

`@RefreshScope` beans can be rebuilt on `POST /actuator/refresh` without a restart.

**The honest assessment, which is what interviewers want:** config servers add a component that every service depends on at startup — a single point of failure needing its own HA story. Kubernetes-native teams increasingly prefer **ConfigMaps + Secrets** (declarative, versioned in git via GitOps, no runtime dependency) and accept a pod restart for config changes. Spring Cloud Config is most valuable in non-Kubernetes environments, or where live refresh without restart is genuinely required.

*"I'd use ConfigMaps on Kubernetes and reserve Spring Cloud Config for dynamic refresh needs or non-K8s deployments"* is a well-calibrated answer.

---

## 9. Build & break

**1. Prove the precedence order.** Set `vigil.demo.value` in four places at once and predict the winner before checking:
```yaml
# application.yml
vigil.demo.value: from-yml
```
```yaml
# application-dev.yml
vigil.demo.value: from-dev-profile
```
```bash
set VIGIL_DEMO_VALUE=from-env          # Windows: use `set` in the run config's env
java -jar target/vigil.jar --spring.profiles.active=dev --vigil.demo.value=from-cli
```
Expose it via an endpoint or log it at startup. Remove the CLI arg → env wins. Remove env → dev profile wins. Remove the profile → base yml. **Run all four steps.** You'll never mis-answer the precedence question again.

**2. Typed, validated properties.** Build `CheckProperties` from §4 with `@Validated`, `@Min`, `@NotEmpty`, and a nested `Retry` record. Then set `max-concurrent=1000` and read the startup failure. **That failure message is the whole point.**

**3. Env-var relaxed binding.** Set `VIGIL_CHECKS_MAX_CONCURRENT=25` and verify it binds. Then try to inject the same value with `@Value("${vigil.checks.maxConcurrent}")` and watch it fail — **that's the `@Value` vs `@ConfigurationProperties` difference, felt rather than read.**

**4. Add config metadata.** Add `spring-boot-configuration-processor`, put Javadoc on each `CheckProperties` component, rebuild, and confirm IntelliJ autocompletes `vigil.checks.` *with your documentation as the tooltip*.

**5. Profiles, done properly.**
- `application-dev.yml`: `show-sql: true`, DEBUG logging, an embedded/local DB
- `application-prod.yml`: WARN logging, `${DB_URL}`, `ddl-auto: validate`
- A `@Profile("!prod")` `NoOpNotificationChannel` and a `@Profile("prod")` real one
- Start with each profile and verify via `/actuator/beans` which channel exists

**6. Secrets hygiene.** Move the DB password to `${DB_PASSWORD}` with no default. Start the app without it and read the failure — **fail-fast on a missing secret is correct behaviour, not a bug.** Create `application-local.yml` with the real value, add it to `.gitignore`, and commit `application-local.yml.example` as documentation.

**7. Inspect `/actuator/configprops` and `/actuator/env`.** Find `vigil.checks` and note the property **source** for each value. When someone says "the config isn't taking effect," this endpoint is your answer.

### Explain out loud (60s)

1. Why one artifact must run everywhere, and what that implies about config.
2. The precedence order, compressed into two rules.
3. Four reasons `@ConfigurationProperties` beats `@Value`.
4. The profiles-vs-properties rule, in one sentence.
5. The secrets progression from committed plaintext to no long-lived secret at all.

---

## 10. Production rules

| Do / Never | Why |
|---|---|
| **Remember precedence as two rules: more specific wins, and closer to the deployment wins** | Command-line > env var > profile file > base file, and outside the jar beats inside — that pair reconstructs the whole table |
| **Prefer `@ConfigurationProperties` over `@Value`** | It gives you typed binding, relaxed binding, `@Validated` startup failure, nested structure and a `/actuator/configprops` entry; `@Value` gives you a string |
| **Never rely on relaxed binding with `@Value`** | Relaxed binding is a `@ConfigurationProperties` feature — `@Value("${vigil.webhookUrl}")` needs the key spelled exactly that way |
| **Use `@Validated` with `@Min`/`@NotEmpty` on your properties** | It converts a typo'd or missing value into a startup failure with a precise message, instead of a wrong value discovered in production |
| **Never commit a secret, even briefly** | Git history is permanent across every clone and fork, scanners crawl public pushes within minutes, and the only real fix is rotation |
| **Reference secrets as `${DB_PASSWORD}` with no default** | A missing value then fails loudly at startup rather than silently starting with an empty password |
| **Prefer mounted secret files (`configtree:`) over environment variables** | File permissions apply, and files don't leak into `/proc/<pid>/environ`, `docker inspect`, crash dumps or child processes |
| **Say "rotation" when you say "secret manager"** | Central storage without rotation only moves the secret; short-lived dynamic credentials are what actually shrinks the blast radius |
| **Never let a profile change business logic** | The production path then becomes the one you never test, and "it worked in staging" stops meaning anything |
| **Use a profile when the *number of beans* differs, a property when only a *value* differs** | That one line decides every profiles-vs-properties argument you'll have in a code review |
| **Set the active profile from the environment, never in a committed `application.yml`** | A hardcoded `prod` in a committed file has caused real incidents, and any environment value silently overrides it anyway |
| **Remember that no active profile means the `default` profile is active** | `@Profile("default")` beans load when you meant "always", which is a surprising way to get an extra bean in production |
| **Run a secret scanner (`gitleaks`) as a pre-commit hook** | It is the only control in this table that catches the mistake *before* it becomes permanent |

---

## 11. Interview traps

**Q1. What's the property precedence order?**
Command-line args > `SPRING_APPLICATION_JSON` > system properties > OS env vars > profile-specific outside the JAR > profile-specific inside > `application.yml` outside > inside > `@PropertySource` > defaults. Two rules explain it: more specific wins, closer to the deployment wins.

**Q2. `@Value` vs `@ConfigurationProperties`?**
`@Value` injects a single value (supports SpEL, exact key only, no validation). `@ConfigurationProperties` binds a whole typed object with relaxed binding, Bean Validation, nested structures, IDE metadata, and `/actuator/configprops` visibility. Use `@ConfigurationProperties` by default; `@Value` for one-offs.

**Q3. What is relaxed binding, and where doesn't it apply?**
`vigil.checks.max-concurrent` also binds from `maxConcurrent`, `max_concurrent`, and `VIGIL_CHECKS_MAX_CONCURRENT`. Applies to `@ConfigurationProperties` only — **not** to `@Value`, which needs an exact key.

**Q4. How do you set config in a container?**
Environment variables with the uppercase/underscore form, or mounted secret files consumed via `spring.config.import=configtree:/run/secrets/`. Never rebuild the image per environment — one artifact, environment-supplied config.

**Q5. How do you handle secrets?**
Never in the repo. Minimum: env vars from the platform's secret store, gitignored local files, and a secret-scanning pre-commit hook. Better: mounted secret files (`configtree:`). Best: a secret manager (Vault / AWS Secrets Manager) with **rotation** and short-lived dynamic credentials, or workload identity so no long-lived secret exists at all.

**Q6. When should you use profiles, and when shouldn't you?**
Use them when the **set of beans** differs structurally (embedded vs real DB, no-op vs real notification channel, platform-specific beans). Don't use them for values, and never for business logic — that makes production the only untested code path. Values belong in properties so every environment runs identical code.

**Q7. What happens if no profile is active?**
The `default` profile is active, so `@Profile("default")` beans load. Not the same as "no profile," which surprises people who expect those beans to be skipped.

**Q8. How do you validate configuration at startup?**
`@ConfigurationProperties` + `@Validated` with Bean Validation annotations (`@NotBlank`, `@Min`, `@Valid` for nesting). Binding failures abort startup with the property name, offending value, and reason.

**Q9. What is Spring Cloud Config and would you use it?**
A server that serves versioned configuration (usually from git) to services at startup, with `@RefreshScope` for live updates. Worth it for non-Kubernetes fleets or genuine no-restart refresh requirements; on Kubernetes, ConfigMaps/Secrets via GitOps are usually simpler and avoid a startup-time single point of failure.

**Q10. How does Spring convert `60s` into a `Duration`?**
Built-in converters registered in the `Environment` handle `Duration` (`60s`, `5m`, `2h`, `500ms`), `DataSize` (`10MB`), enums, `List`/`Map`, and `Period`. `@DurationUnit`/`@DataSizeUnit` set the unit for bare numbers.

**Q11. Records or classes for `@ConfigurationProperties`?**
Records for immutable constructor binding (the modern default) with `@DefaultValue` for optional components. Classes with setters when you need mutability — chiefly `@RefreshScope` with a config server.

---

## 12. Retention check

1. State the top four levels of the precedence order.
2. Give the exact env-var spelling for `vigil.notification.slack.webhook-url`.
3. Give three reasons `@ConfigurationProperties` beats `@Value`, including the one about startup.
4. Explain why `@Profile` on business logic is dangerous, in one sentence.
5. Someone has a DB password committed to git and deletes the line. What do you tell them?
6. What's `configtree:` for, and why is it better than env vars for secrets?

---

**Next:** [Lesson 13 — Project structure like a senior engineer](13-project-structure.md) — package layout, layering, and the architecture arguments you'll be expected to hold your own in.

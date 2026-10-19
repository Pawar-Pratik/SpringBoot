# Lesson 11 — Auto-configuration demystified

> **Why this lesson exists:** Auto-configuration is *the* Spring Boot feature, and it's where "magic" accusations come from. You already have every prerequisite: annotations (Lesson 04), meta-annotations, bean definitions vs instances (Lesson 06), and `@Conditional` (Lesson 08). This lesson just assembles them. By the end you'll be able to trace why *any* bean exists in *any* Spring Boot app, and you'll have written your own starter — which is the thing that makes an interviewer sit up.

**Time:** ~70 minutes · **Prereq:** Lessons 04, 06, 08

---

## 1. The idea in one sentence

> **Auto-configuration is `@Conditional` applied with discipline: a starter ships no code at all — it only puts classes on the classpath so that "if this library is present and you haven't configured it yourself, here is a sensible default" becomes true.**

The second half matters as much as the first. The import selector is *deferred*, so it runs after every bean of yours is registered — which is the only reason `@ConditionalOnMissingBean` can reliably mean "the user didn't define one."

---

## 2. Decompose `@SpringBootApplication`

```java
@SpringBootApplication
public class VigilApplication {
    public static void main(String[] args) {
        SpringApplication.run(VigilApplication.class, args);
    }
}
```

Three lines that do a startling amount. Open the annotation:

```java
@Target(TYPE) @Retention(RUNTIME)
@SpringBootConfiguration               // = @Configuration (+ marks it as THE primary config)
@EnableAutoConfiguration               // ← the whole lesson is this one
@ComponentScan(excludeFilters = { ... })
public @interface SpringBootApplication { }
```

- **`@SpringBootConfiguration`** — a `@Configuration` variant. There must be exactly one per app; tests use it to locate the context (`@SpringBootTest` searches up the package tree for it — which is why a misplaced main class breaks tests in a confusing way).
- **`@ComponentScan`** — scans this package and below (Lesson 06).
- **`@EnableAutoConfiguration`** — everything else.

And `@EnableAutoConfiguration` itself:

```java
@Target(TYPE) @Retention(RUNTIME)
@AutoConfigurationPackage                              // registers this package for other features
@Import(AutoConfigurationImportSelector.class)         // ★ THE MECHANISM
public @interface EnableAutoConfiguration { }
```

**It's an `@Import` of one class.** That's the entire trick. Everything Spring Boot does flows through `AutoConfigurationImportSelector`.

---

## 3. What `AutoConfigurationImportSelector` does

It's a `DeferredImportSelector` — "deferred" meaning **it runs last**, after all your own configuration classes are processed. That ordering is what makes `@ConditionalOnMissingBean` work (Lesson 08 §3).

Its algorithm:

```
1. Scan EVERY jar on the classpath for:
      META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
   (a plain text file: one fully-qualified class name per line)

2. Concatenate them → ~150 candidate auto-configuration class names

3. Remove exclusions (@SpringBootApplication(exclude=...), spring.autoconfigure.exclude=...)

4. FILTER using AutoConfigurationImportFilter — a fast pre-pass that reads each candidate's
   @ConditionalOnClass/@ConditionalOnBean metadata from a pre-computed index
   (spring-autoconfigure-metadata.properties) WITHOUT loading the classes.
   This is a startup optimization: it discards ~70% of candidates cheaply.

5. Sort by @AutoConfigureBefore / @AutoConfigureAfter / @AutoConfigureOrder

6. Import the survivors as @Configuration classes → each one's @Conditional
   annotations are now evaluated in full, per @Bean method

7. Fire an AutoConfigurationImportEvent (this is what the /actuator/conditions report reads)
```

**Step 4 is the interesting engineering.** Naively evaluating 150 configuration classes would mean loading hundreds of classes just to throw them away. Boot ships a pre-generated metadata index so it can reject candidates by *name* first. This is one of the concrete reasons Boot startup isn't much worse than plain Spring, and mentioning it demonstrates you've actually looked.

> **Fun fact / history:** Before Boot 2.7 this file was `META-INF/spring.factories`, with a giant `org.springframework.boot.autoconfigure.EnableAutoConfiguration=\` key and hundreds of backslash-continued lines. Boot 2.7 introduced the `AutoConfiguration.imports` file; Boot 3 removed `spring.factories` support for auto-configuration entirely. **If you see `spring.factories` in a tutorial, it's pre-2022 material** — a useful dating tool. (`spring.factories` still exists for *other* extension points, like `ApplicationListener` registration.)

---

## 4. Read a real auto-configuration class

Here's a faithful sketch of `DataSourceAutoConfiguration`, which is representative of all of them:

```java
@AutoConfiguration(before = SqlInitializationAutoConfiguration.class)
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })   // is JDBC even present?
@ConditionalOnMissingBean(type = "io.r2dbc.spi.ConnectionFactory")      // not if we're reactive
@EnableConfigurationProperties(DataSourceProperties.class)             // bind spring.datasource.*
@Import({ DataSourcePoolMetadataProvidersConfiguration.class, ... })
public class DataSourceAutoConfiguration {

    @Configuration(proxyBeanMethods = false)
    @Conditional(PooledDataSourceCondition.class)
    @ConditionalOnMissingBean({ DataSource.class, XADataSource.class })  // ★ user override hook
    @Import({ Hikari.class, Tomcat.class, Dbcp2.class, OracleUcp.class, Generic.class })
    protected static class PooledDataSourceConfiguration { }

    @Configuration(proxyBeanMethods = false)
    static class Hikari {
        @Bean
        @ConditionalOnMissingBean(DataSource.class)
        @ConditionalOnClass(HikariDataSource.class)          // is Hikari on the classpath?
        HikariDataSource dataSource(DataSourceProperties properties) {
            HikariDataSource ds = createDataSource(properties, HikariDataSource.class);
            if (StringUtils.hasText(properties.getName())) ds.setPoolName(properties.getName());
            return ds;
        }
    }
}
```

**Read that structure and you've read all of Spring Boot.** The recurring pattern is:

```
@ConditionalOnClass(SomeLibrary.class)        "is the library present?"
@ConditionalOnMissingBean(SomeType.class)     "did the user configure their own?"
@EnableConfigurationProperties(Props.class)   "bind the namespaced properties"
@Bean                                          "create a sensible default"
```

Which decodes to a single sentence:

> **"If the library is on the classpath and you haven't configured it yourself, I'll create a reasonable default, configurable via properties."**

That's Spring Boot. Not magic — a very disciplined application of `@Conditional`.

**And now you can answer the deeper questions:**

- *"How does adding one dependency configure my whole database layer?"* — the starter puts Hibernate + Hikari + the driver on the classpath; `@ConditionalOnClass` conditions that were previously false become true; the auto-configurations activate.
- *"Why does my own `DataSource` bean win?"* — `@ConditionalOnMissingBean`, evaluated after your config because the import selector is deferred.
- *"Why doesn't Boot configure two web servers?"* — each server's auto-config is `@ConditionalOnMissingBean(ServletWebServerFactory.class)` and they're ordered, so the first match wins. Excluding Tomcat and adding Undertow just changes which condition passes.

---

## 5. Debugging auto-configuration — the skills that matter

This is the practical payoff. **When a senior engineer sees unexpected Spring behaviour, they run these. Learn them now.**

### Tool 1 — the condition evaluation report

```properties
# application.properties
debug=true
```
or `java -jar vigil.jar --debug`

```
============================
CONDITIONS EVALUATION REPORT
============================

Positive matches:
-----------------
   DataSourceAutoConfiguration matched:
      - @ConditionalOnClass found required classes 'javax.sql.DataSource',
        'org.springframework.jdbc.datasource.embedded.EmbeddedDatabaseType'

   DataSourceAutoConfiguration#dataSource matched:
      - @ConditionalOnMissingBean (types: javax.sql.DataSource) did not find any beans

Negative matches:
-----------------
   RedisAutoConfiguration:
      Did not match:
         - @ConditionalOnClass did not find required class
           'org.springframework.data.redis.core.RedisOperations'

   KafkaAutoConfiguration:
      Did not match:
         - @ConditionalOnClass did not find required class 'org.apache.kafka.clients.CommonClientConfigs'

Exclusions:
-----------
   None

Unconditional classes:
----------------------
   org.springframework.boot.autoconfigure.context.ConfigurationPropertiesAutoConfiguration
```

**The "Negative matches" section is the one you'll actually use.** "Why isn't my Redis cache working?" → search for `RedisAutoConfiguration` in the negative matches and it tells you the missing class or bean, exactly. That turns a 40-minute mystery into a 20-second lookup.

### Tool 2 — Actuator's `/actuator/conditions`

Same report as JSON, at runtime, in any environment:

```properties
management.endpoints.web.exposure.include=conditions,beans,configprops,env,mappings,health
```

### Tool 3 — `/actuator/beans`

Every bean, its type, scope, `resource` (which configuration class created it!), and dependencies. **The `resource` field is the answer to "where did this bean come from?"**

### Tool 4 — `/actuator/configprops`

Every `@ConfigurationProperties` object with its resolved values and the property source each came from. The fastest way to answer "is my config actually being picked up?"

### Tool 5 — excluding auto-configurations

```java
@SpringBootApplication(exclude = { DataSourceAutoConfiguration.class, SecurityAutoConfiguration.class })
```
```properties
spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

Common real reasons: you manage a `DataSource` yourself; you're writing a non-web module that dragged in web transitively; you want to fully own security config. **Excluding is a legitimate tool, not a defeat** — but if you're excluding many, question the dependency that pulled them in.

---

## 6. Starters — what's actually in one

Open `spring-boot-starter-web`'s POM and you find… **no code**. A starter is a POM with dependencies and nothing else:

```xml
<dependencies>
  <dependency> spring-boot-starter </dependency>          <!-- core + logging + autoconfigure -->
  <dependency> spring-boot-starter-json </dependency>     <!-- Jackson -->
  <dependency> spring-boot-starter-tomcat </dependency>   <!-- embedded server -->
  <dependency> spring-web </dependency>
  <dependency> spring-webmvc </dependency>
</dependencies>
```

**Starters are curated dependency bundles.** Their power is entirely indirect: by putting classes on the classpath, they flip `@ConditionalOnClass` conditions from false to true, which activates auto-configurations that were already sitting there inert.

Say that sentence in an interview — *"a starter contains no code; it's a dependency bundle whose only job is to make classpath conditions true"* — and you've demonstrated real understanding.

**Naming convention (matters if you publish one):**
- Official Spring: `spring-boot-starter-*`
- Third-party: `*-spring-boot-starter` (e.g. `mybatis-spring-boot-starter`)

Reversing this is considered impolite because it implies official status. Interviewers occasionally ask.

**Starters worth knowing:**

| Starter | Brings |
|---|---|
| `web` | Spring MVC, embedded Tomcat, Jackson, validation |
| `webflux` | Reactive stack, Netty |
| `data-jpa` | Hibernate, Spring Data JPA, HikariCP, transactions |
| `data-redis` | Lettuce client, Spring Data Redis |
| `security` | Spring Security filter chain |
| `oauth2-resource-server` | JWT validation |
| `validation` | Hibernate Validator (**not** included in `web` since Boot 2.3 — a common surprise) |
| `actuator` | Health, metrics, info endpoints |
| `test` | JUnit 5, AssertJ, Mockito, MockMvc, Testcontainers support |
| `aop` | AspectJ weaver for pointcut parsing |
| `mail`, `quartz`, `batch`, `amqp`, `cache`, `thymeleaf` | as named |

> **The `validation` gotcha is real:** it was removed from `starter-web` in Boot 2.3, so upgrading silently disabled `@Valid` for a lot of applications — requests that should have been rejected with 400 started reaching controllers with invalid data. If you ever want a concrete "a dependency change caused a production bug" story, that's a good one.

---

## 7. Build your own starter

**This is the exercise that separates you from the pack.** Most Spring developers have never written a starter, and it's the clearest possible demonstration that you understand the mechanism.

We'll build `vigil-notification-spring-boot-starter` — auto-configures a notification channel based on properties, with a sensible default and full user override.

### Project layout

Real starters are split in two (this is the official convention):
```
vigil-notification-spring-boot-autoconfigure/    ← the code + conditions
vigil-notification-spring-boot-starter/          ← an empty POM depending on the above
```
For learning, one module is fine. In an interview, **mention the split and why** (so users can depend on the autoconfigure module without the opinionated transitive dependencies).

### 1. Properties class

```java
package com.vigil.notification.autoconfigure;

@ConfigurationProperties(prefix = "vigil.notification")
@Validated
public record NotificationProperties(
        @DefaultValue("true")  boolean enabled,
        @DefaultValue("LOG")   ChannelType type,
        @Valid Slack slack,
        @DefaultValue("5s")    Duration timeout
) {
    public enum ChannelType { LOG, SLACK, WEBHOOK }

    public record Slack(String webhookUrl, @DefaultValue("#alerts") String channel) {}
}
```

Records + `@DefaultValue` is the modern immutable-binding style. Note `@Validated` — invalid config now fails at **startup** with a clear message rather than at first use.

### 2. The auto-configuration

```java
package com.vigil.notification.autoconfigure;

@AutoConfiguration
@ConditionalOnClass(NotificationChannel.class)
@ConditionalOnProperty(prefix = "vigil.notification", name = "enabled",
                       matchIfMissing = true)
@EnableConfigurationProperties(NotificationProperties.class)
public class NotificationAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean                                       // ★ user override hook
    @ConditionalOnProperty(prefix = "vigil.notification", name = "type", havingValue = "LOG",
                           matchIfMissing = true)
    NotificationChannel logNotificationChannel() {
        return new LoggingNotificationChannel();
    }

    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnProperty(prefix = "vigil.notification", name = "type", havingValue = "SLACK")
    NotificationChannel slackNotificationChannel(NotificationProperties props,
                                                 RestClient.Builder builder) {
        Assert.hasText(props.slack().webhookUrl(),
                "vigil.notification.slack.webhook-url is required when type=SLACK");
        return new SlackNotificationChannel(builder.build(), props);
    }

    @Bean
    @ConditionalOnMissingBean
    NotificationDispatcher notificationDispatcher(List<NotificationChannel> channels) {
        return new NotificationDispatcher(channels);
    }
}
```

`@AutoConfiguration` (Boot 2.7+) is the modern replacement for `@Configuration` in this role — it implies `proxyBeanMethods = false` and carries `before`/`after` ordering attributes.

### 3. Register it

`src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`:
```
com.vigil.notification.autoconfigure.NotificationAutoConfiguration
```

**That's it.** One file, one line. That file is the entire registration mechanism for every one of the ~150 auto-configurations in Spring Boot.

### 4. Add IDE autocomplete for your properties (the professional touch)

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
    <optional>true</optional>
</dependency>
```

This annotation processor generates `META-INF/spring-configuration-metadata.json` at compile time, so **your** properties get autocomplete and inline docs in IntelliJ, exactly like Spring's own. Genuinely impressive in a demo, and a five-second addition. Add Javadoc to your properties record and it becomes the tooltip.

### 5. Use it

```properties
vigil.notification.type=SLACK
vigil.notification.slack.webhook-url=https://hooks.slack.com/services/xxx
vigil.notification.timeout=10s
```

Consumers just add the dependency and set properties. And if they define their own `NotificationChannel` bean, `@ConditionalOnMissingBean` steps aside — the override contract works exactly like Spring's own.

**When would you actually build a starter in industry?** Shared infrastructure across many services in one company: a standard security config, a company-wide logging/tracing setup, a shared client for an internal API, a multi-tenancy module. This is extremely common in organizations with 20+ services, and it's a great thing to have on your resume.

---

## 8. Anti-patterns and cautions

**❌ Fighting auto-configuration instead of understanding it.** Symptom: a config class full of `@Bean` overrides copied from Stack Overflow. Fix: read the negative-matches report and find out *why* the default didn't apply.

**❌ Excluding auto-configurations to silence errors.** If `DataSourceAutoConfiguration` fails, the real problem is a missing URL or driver, not the auto-configuration. Excluding it hides the bug until something else breaks.

**❌ `@ConditionalOnMissingBean` in your own application config.** Ordering between your own `@Configuration` classes isn't guaranteed, so the outcome is non-deterministic (Lesson 08). It belongs in auto-configuration classes only.

**❌ Assuming a starter implies auto-configuration.** They're separate: `spring-boot-starter-aop` is *just* dependencies; the auto-config lives in `spring-boot-autoconfigure`. A third-party starter might bundle both or neither.

**⚠️ Auto-configuration and testing.** `@SpringBootTest` loads *everything*, which is slow and can require infrastructure. Test slices (`@WebMvcTest`, `@DataJpaTest`) load a curated subset. Understanding that slices are just filtered auto-configuration makes Module 8 much easier.

---

## 9. Build & break

**1. Read your own condition report.** `debug=true`, restart, and answer in writing:
- How many positive matches? Negative?
- Why did `RedisAutoConfiguration` not match?
- Which auto-configuration created your `ObjectMapper`?
- Which one created the embedded Tomcat?

**2. Trace one bean end to end.** Pick `ObjectMapper`. Find it in `/actuator/beans`, note the `resource` field, then find `JacksonAutoConfiguration` in the report. Now **define your own `ObjectMapper` bean** and re-check: the negative matches now show Boot's backing off due to `@ConditionalOnMissingBean`. **You've just watched the override contract work.** (Better still, customize it the intended way — a `Jackson2ObjectMapperBuilderCustomizer` bean — and note that this *adds* to Boot's config instead of replacing it. That distinction between "override" and "customize" is a real senior/junior divider.)

**3. Break it on purpose.** Exclude `JacksonAutoConfiguration` and hit an endpoint returning an object. Read the failure. Then exclude `DataSourceAutoConfiguration` while `data-jpa` is on the classpath and read *that* failure. **Both errors are informative — practise reading them.**

**4. ⭐ Build the starter.** Follow §6 fully, in the same project for simplicity (`com.vigil.starter.notification`). Then:
- Verify the default `LOG` channel is created with no properties set.
- Switch `type=SLACK`, verify the Slack channel is created instead.
- Define your **own** `NotificationChannel` bean in the app and verify *both* auto-configured beans back off. Confirm via `/actuator/beans`.
- Add `spring-boot-configuration-processor`, rebuild, and confirm IntelliJ autocompletes `vigil.notification.` in `application.properties`. **That moment is very satisfying.**
- Set `enabled=false` and confirm nothing is created.
- Set `type=SLACK` without a webhook URL and confirm it **fails at startup** with your message. Fail-fast, by design.

**5. Find the imports file.** In `~/.m2/repository/org/springframework/boot/spring-boot-autoconfigure/<version>/`, open the JAR and read `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`. Count the lines. **That plain text file is Spring Boot's entire registry.** Seeing it makes the whole thing feel small and knowable — which is the point of this lesson.

### Explain out loud (60s)

1. What `@SpringBootApplication` decomposes into, and which part does the work.
2. What a starter actually contains, and how something with no code changes your app.
3. Why the import selector is deferred, and what depends on that ordering.
4. The four-annotation pattern every auto-configuration class follows.
5. The first two commands you run when a bean you expected isn't there.

---

## 10. Production rules

| Do / Never | Why |
|---|---|
| **Read the "Negative matches" section of `debug=true` before theorising** | It names the exact missing class or bean that stopped an auto-configuration, turning a 40-minute mystery into a 20-second lookup |
| **Use `/actuator/beans`' `resource` field to answer "where did this bean come from?"** | It names the configuration class that created it, which no amount of grepping your own code will tell you |
| **Exclude an auto-configuration deliberately, never habitually** | Excluding one because you own that concern is fine; excluding several means a dependency dragged in a whole stack you didn't want |
| **Name a third-party starter `*-spring-boot-starter`** | `spring-boot-starter-*` is reserved for the Spring team, and taking it makes your artifact look official when it isn't |
| **Put `@ConditionalOnMissingBean` on every bean a starter contributes** | It is what makes your defaults overridable, which is the entire contract users expect from a starter |
| **Register auto-configurations in `AutoConfiguration.imports`, never `spring.factories`** | Boot 3 removed `spring.factories` support for auto-configuration outright — a tutorial using it predates 2022 |
| **Declare auto-config classes with `@AutoConfiguration` and `proxyBeanMethods = false`** | Skipping CGLIB generation across hundreds of config classes is measurable startup time, and parameters make it safe |
| **Bind starter settings with `@ConfigurationProperties` under your own namespace** | It gives users typed, validated, documented config that shows up in `/actuator/configprops` instead of scattered `@Value` strings |
| **Never put business beans in an auto-configuration** | Auto-config is for infrastructure defaults; business logic that appears conditionally is behaviour nobody can trace |
| **Keep the `@SpringBootApplication` class in the root package** | `@SpringBootTest` searches up the package tree for `@SpringBootConfiguration`, so a misplaced main class breaks tests in a confusing way |

---

## 11. Interview traps

**Q1. How does auto-configuration work?** *(Give the full chain — this is the flagship answer.)*
`@SpringBootApplication` includes `@EnableAutoConfiguration`, which `@Import`s `AutoConfigurationImportSelector` — a `DeferredImportSelector`, so it runs after all user configuration. It reads `META-INF/spring/…AutoConfiguration.imports` from every JAR on the classpath (~150 candidates), pre-filters them using a metadata index without loading classes, applies exclusions, sorts by `@AutoConfigureBefore/After`, and imports the survivors. Each imported class's `@Conditional` annotations then decide which beans are actually created — typically `@ConditionalOnClass` ("is the library present?") plus `@ConditionalOnMissingBean` ("did the user define their own?").

**Q2. Why is the import selector "deferred"?**
So auto-configuration is processed **last**. That's what makes `@ConditionalOnMissingBean` meaningful — all user-defined beans are registered by then, so "did the user provide one?" is answerable. Reverse the order and user overrides would stop working.

**Q3. What is a starter? Does it contain code?**
A curated dependency bundle — typically **no code at all**, just a POM. Its effect is indirect: it puts classes on the classpath, flipping `@ConditionalOnClass` conditions true and thereby activating auto-configurations that already exist.

**Q4. How do you override an auto-configured bean?**
Define your own bean of that type — `@ConditionalOnMissingBean` makes Boot back off. Or exclude the auto-configuration entirely. Or, better for many cases, use the provided **customizer** hook (e.g. `Jackson2ObjectMapperBuilderCustomizer`, `WebMvcConfigurer`) which augments Boot's configuration rather than replacing it.

**Q5. How do you find out why an auto-configuration didn't apply?**
`--debug` / `debug=true` for the condition evaluation report, or `/actuator/conditions` at runtime. Read the **negative matches** — each entry states the exact unmet condition.

**Q6. How would you write your own starter?**
An `@AutoConfiguration` class with `@ConditionalOnClass`, `@ConditionalOnProperty`, `@ConditionalOnMissingBean` and `@EnableConfigurationProperties`; register it in `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`; add `spring-boot-configuration-processor` for IDE metadata; split into `-autoconfigure` and `-starter` modules; name it `xxx-spring-boot-starter` (third-party convention).

**Q7. `spring.factories` vs `AutoConfiguration.imports`?**
`spring.factories` was the pre-2.7 registration mechanism (one key with hundreds of continued lines). Boot 2.7 introduced the dedicated `AutoConfiguration.imports` file; Boot 3 dropped `spring.factories` support for auto-configuration. `spring.factories` still exists for other extension points. Seeing it used for auto-config dates material as pre-2022.

**Q8. Does auto-configuration slow startup, and how does Boot mitigate it?**
There's a cost to evaluating ~150 candidates, mitigated by a pre-computed metadata index that rejects candidates by name without loading classes, and by ASM-based annotation reading. Further mitigations: excluding unused auto-configurations, lazy initialization, AOT processing, CDS, and native images.

**Q9. `@Configuration` vs `@AutoConfiguration`?**
`@AutoConfiguration` (2.7+) is for auto-configuration classes specifically: it implies `proxyBeanMethods = false` and adds `before`/`after`/`beforeName`/`afterName` ordering attributes. Use `@Configuration` for application config, `@AutoConfiguration` for starter code.

**Q10. If two auto-configurations both want to create a `DataSource`, what happens?**
Ordering plus `@ConditionalOnMissingBean` resolves it: they're sorted, the first to match creates the bean, and the rest back off because a bean now exists. This is how Boot supports Hikari/Tomcat/DBCP2 pools with a documented preference order.

---

## 12. Retention check

1. Trace, step by step, how adding `spring-boot-starter-data-jpa` results in an `EntityManagerFactory` bean.
2. Why would `@ConditionalOnMissingBean` be broken if the import selector weren't deferred?
3. What single file registers an auto-configuration, and where does it live?
4. Someone says "Spring Boot is too magical." Give a four-sentence rebuttal naming specific tooling.
5. What's the difference between *overriding* an auto-configured bean and *customizing* it? Which is usually better?
6. Why does a starter contain no code?

---

**Next:** [Lesson 12 — Configuration, profiles & secrets](12-configuration-and-profiles.md) — externalized config, the precedence order you'll be quizzed on, and how to handle secrets without embarrassing yourself.

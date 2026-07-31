# Lesson 06 — The IoC container & beans

> **Why this lesson exists:** The container is the thing you're actually programming when you write Spring. Most developers use it for years while thinking of it as a black box. You're going to open it: bean definitions, the startup sequence, the difference between the `ApplicationContext` and the `BeanFactory`, and the five ways a bean can come into existence. This is the lesson that makes you able to *debug* Spring rather than guess at it.

**Time:** ~75 minutes · **Prereq:** Module 1 (especially Lesson 04)

---

## 1. What a bean actually is

> **A bean is an object whose lifecycle is managed by the Spring container.**

That's the whole definition. Not a special type, not an interface you implement, no base class. A `MonitorService` created by Spring is a bean; the identical `MonitorService` created with `new` is just an object.

What "managed" buys you:
- Spring creates it, and decides *when*
- Spring injects its dependencies
- Spring may **wrap it in a proxy** (transactions, caching, security, metrics)
- Spring calls its lifecycle hooks (`@PostConstruct`, `@PreDestroy`)
- Spring decides how many exist (scope) and destroys them on shutdown

> **Terminology note worth knowing:** the word comes from **JavaBeans** (1996) — a convention for reusable components with a no-arg constructor and getters/setters. Spring's "bean" is much looser: any object in the container. A Spring bean does *not* need getters/setters or a no-arg constructor. This trips people up because the old JavaBean rules still show up in places that *do* require them — JPA entities and Jackson deserialization, for example. So: same word, different contracts, depending on who's reading your object.

---

## 2. Bean definitions vs bean instances — the distinction most people miss

This is the key insight of the lesson.

```
   @Service                        BeanDefinition                    Instance
   class MonitorService  ──scan──▶  { class: MonitorService,   ──▶   MonitorService@4f2a1b
                                      scope: singleton,               (possibly wrapped in
                                      lazyInit: false,                 a proxy)
                                      constructorArgs: [...],
                                      initMethod: null,
                                      primary: false }

        SOURCE OF TRUTH              THE RECIPE                     THE CAKE
```

Spring's startup has **two distinct phases**, and knowing they're separate explains a lot of behaviour:

**Phase 1 — collect bean *definitions*** (recipes). Nothing is instantiated. Definitions come from component scanning, `@Bean` methods, auto-configuration, and XML. They can still be *modified* at this point.

**Phase 2 — instantiate the singletons.** Spring walks the definitions, resolves dependencies, creates objects, applies post-processors (which may replace an object with a proxy), and runs lifecycle callbacks.

**Why this matters practically:**

| Behaviour | Because definitions and instances are separate phases |
|---|---|
| A later `@Bean` can **override** an earlier one | Definitions are registered into a map — same name replaces (and Boot disables this by default, see §6) |
| `@ConditionalOnMissingBean` works at all | Conditions are evaluated during definition registration, so it can ask "was a definition already registered?" |
| `BeanFactoryPostProcessor` can rewrite config | It runs between the two phases, on definitions |
| `BeanPostProcessor` can return a **proxy** | It runs during phase 2, on instances |
| Your `@Bean` method might never be called | If its definition loses a conditional or gets overridden |

> **Interview gold:** *"What's the difference between `BeanFactoryPostProcessor` and `BeanPostProcessor`?"* — `BeanFactoryPostProcessor` operates on **bean definitions** before any bean is instantiated (e.g. `PropertySourcesPlaceholderConfigurer` resolving `${...}`). `BeanPostProcessor` operates on **bean instances** during creation, and can return a *different* object — which is exactly how AOP proxies get created (`AnnotationAwareAspectJAutoProxyCreator` is a `BeanPostProcessor`). Very few candidates can state this cleanly.

---

## 3. The container types: `BeanFactory` vs `ApplicationContext`

```
BeanFactory                         ← the bare container: getBean(), lazy instantiation
   ▲
   │ extends
ApplicationContext                  ← BeanFactory + enterprise features:
   ▲                                   · eager singleton instantiation
   │                                   · event publishing (ApplicationEventPublisher)
   │                                   · i18n message resolution (MessageSource)
   │                                   · resource loading (ResourceLoader)
   │                                   · automatic BeanPostProcessor registration
   │                                   · Environment / property source access
   │
   ├── AnnotationConfigApplicationContext            (plain Java app)
   ├── ClassPathXmlApplicationContext                (legacy XML)
   ├── AnnotationConfigServletWebServerApplicationContext   ← what Spring Boot web apps use
   └── AnnotationConfigReactiveWebServerApplicationContext  (WebFlux)
```

**You will always use `ApplicationContext`.** `BeanFactory` exists as the minimal SPI, used internally and in extremely memory-constrained scenarios. The interview-relevant differences:

| | `BeanFactory` | `ApplicationContext` |
|---|---|---|
| Instantiation | Lazy — on `getBean()` | **Eager** — all singletons at startup |
| `BeanPostProcessor` | Must register manually | Auto-detected |
| Events | ❌ | ✅ |
| i18n, resources | ❌ | ✅ |
| Annotation config | Limited | Full |

**Why eager instantiation is the right default:** it means configuration errors — a missing bean, a bad database URL, an ambiguous dependency — surface at **startup**, not at 2am when the first user hits that code path. **Fail fast is a deliberate design choice, and it's the correct answer to "why doesn't Spring create beans lazily?"**

```java
// Getting the context, if you ever need it
@Component
class ContextPeeker implements ApplicationContextAware {   // or just inject ApplicationContext
    public void setApplicationContext(ApplicationContext ctx) {
        System.out.println("beans: " + ctx.getBeanDefinitionCount());
    }
}
```

Needing `ApplicationContext` in business code is usually a smell — it means you're doing **service locator** lookup instead of injection, which reintroduces the coupling DI removed. Legitimate uses: dynamically choosing an implementation by name at runtime, framework-level code, and tests.

---

## 4. The five ways to create a bean

### Way 1 — `@Component` and its stereotypes (your default)

```java
@Component      // generic
@Service        // business logic layer
@Repository     // data access layer
@Controller     // web layer (returns views)
@RestController // web layer (returns response bodies)
@Configuration  // provides @Bean methods
```

All are meta-annotated with `@Component`, so they're all found by the same scan (Lesson 04). **Functionally, are they identical?** Almost — but not quite, and this is a favourite interview question:

| Stereotype | Extra behaviour beyond `@Component` |
|---|---|
| `@Service` | **None.** Purely semantic. |
| `@Repository` | **Real behaviour:** enables *exception translation* — a `BeanPostProcessor` wraps the bean in a proxy that converts vendor-specific `SQLException`s / Hibernate exceptions into Spring's portable `DataAccessException` hierarchy. |
| `@Controller` | Marks the class for `RequestMappingHandlerMapping` to scan for `@RequestMapping` methods. |
| `@RestController` | `@Controller` + `@ResponseBody` on every method. |
| `@Configuration` | Class is CGLIB-proxied so inter-`@Bean` calls return the singleton (see below — important!). |

**So the honest answer is:** `@Service` and `@Component` are functionally identical today, and you use `@Service` for readability and to communicate intent. `@Repository`, `@Controller`, and `@Configuration` all add real behaviour. Saying "they're all the same, just for readability" is the *almost*-right answer that a good interviewer will push back on.

**Component scanning: what gets scanned?**
`@SpringBootApplication` implies `@ComponentScan` **of its own package and all sub-packages.** So:

```
com.vigil                    ← VigilApplication.java lives here
├── monitor                  ✅ scanned
├── incident                 ✅ scanned
└── notification             ✅ scanned

com.othercompany.util        ❌ NOT scanned — a top-3 "why isn't my bean found?" cause
```

**This is why your main class must live in the root package.** If you must scan elsewhere: `@ComponentScan(basePackages = {"com.vigil", "com.shared.lib"})`. Prefer `basePackageClasses = SharedLibMarker.class` — it's refactor-safe, since renaming a package won't silently break a string.

### Way 2 — `@Bean` methods in a `@Configuration` class (for third-party objects)

You can't annotate a class you don't own. So you produce it yourself:

```java
@Configuration
public class HttpConfig {

    @Bean                                    // bean name = method name = "checkHttpClient"
    public RestClient checkHttpClient(CheckProperties props) {   // params are INJECTED
        return RestClient.builder()
                .requestFactory(clientRequestFactory(props))
                .defaultHeader("User-Agent", "Vigil/1.0")
                .build();
    }

    @Bean
    ClientHttpRequestFactory clientRequestFactory(CheckProperties props) {
        var settings = ClientHttpRequestFactorySettings.DEFAULTS
                .withConnectTimeout(props.connectTimeout())
                .withReadTimeout(props.readTimeout());
        return ClientHttpRequestFactories.get(settings);
    }

    @Bean(name = "auditExecutor", destroyMethod = "shutdown")
    ExecutorService auditExecutor() {
        return Executors.newFixedThreadPool(4);
    }
}
```

**Key facts:**
- Bean name defaults to the **method name**. `@Bean("customName")` overrides.
- Method **parameters are dependencies** — Spring resolves them like constructor args.
- `@Configuration` classes are themselves beans.
- `initMethod`/`destroyMethod` let you hook lifecycle on third-party classes.

**`@Component` vs `@Bean` — when to use which:**

| | `@Component` | `@Bean` |
|---|---|---|
| Applies to | A class **you own** | A method returning **any** object |
| Discovery | Component scan | Explicit declaration |
| Use for | Your services/repos/controllers | Third-party classes, conditional creation, multiple instances of one type, objects needing construction logic |

### The `@Configuration` proxy — a genuinely surprising mechanism

```java
@Configuration
public class DataConfig {
    @Bean DataSource dataSource() { return new HikariDataSource(...); }

    @Bean JdbcTemplate jdbcTemplate() {
        return new JdbcTemplate(dataSource());     // ← calling a @Bean method directly!
    }

    @Bean TransactionManager txManager() {
        return new DataSourceTransactionManager(dataSource());   // ← again!
    }
}
```

**How many `DataSource` objects (and therefore connection pools) are created?**

**One.** Because `@Configuration` classes are **CGLIB-proxied**: the proxy intercepts calls to `@Bean` methods and returns the already-created singleton instead of executing the method body again. This is called **"full" configuration mode** (`proxyBeanMethods = true`, the default).

If you set `@Configuration(proxyBeanMethods = false)` — "lite" mode — no proxy is created, and `dataSource()` becomes a plain method call, giving you **three separate connection pools**. Silently. In production.

```java
// Lite mode is CORRECT and faster, but you must pass dependencies as parameters:
@Configuration(proxyBeanMethods = false)
public class DataConfig {
    @Bean DataSource dataSource() { return new HikariDataSource(...); }
    @Bean JdbcTemplate jdbcTemplate(DataSource ds) { return new JdbcTemplate(ds); }   // ✅ injected
}
```

> **Real-world note:** Spring's own auto-configuration classes almost all use `proxyBeanMethods = false`, because avoiding CGLIB proxy generation for hundreds of config classes measurably improves startup time. **Best practice: always take dependencies as method parameters rather than calling `@Bean` methods, and then `proxyBeanMethods = false` is safe.** This is a strong senior-level answer.

### Way 3 — `@Import` and `@Enable*` annotations

```java
@Configuration
@Import({SecurityConfig.class, ObservabilityConfig.class})
public class AppConfig { }
```

Every `@EnableXxx` annotation (`@EnableScheduling`, `@EnableCaching`, `@EnableAsync`, `@EnableWebSecurity`) is just a meta-annotation carrying an `@Import` of a configuration class. Look at the source of `@EnableScheduling` — it's three lines. **Demystifying `@Enable*` is a nice interview moment:** "it's an `@Import` of a config class that registers the relevant post-processors."

`ImportSelector` and `ImportBeanDefinitionRegistrar` are the programmatic versions — they decide at runtime which configurations to import. **`@EnableAutoConfiguration` uses an `ImportSelector`.** That's the whole trick behind Boot (Lesson 11).

### Way 4 — `FactoryBean` (rare, but you'll meet it)

A bean whose job is to produce another bean, when construction is too complex for a `@Bean` method.

```java
public class EncryptedDataSourceFactoryBean implements FactoryBean<DataSource> {
    public DataSource getObject() { /* decrypt creds, build DataSource */ }
    public Class<?> getObjectType() { return DataSource.class; }
    public boolean isSingleton() { return true; }
}
```

**The gotcha:** `ctx.getBean("myFactory")` returns the *produced* object; `ctx.getBean("&myFactory")` (with `&`) returns the factory itself. Spring Data's repository infrastructure uses `FactoryBean`s heavily — that's how a bean appears for an interface you never implemented.

### Way 5 — programmatic registration

```java
@Component
class DynamicRegistrar implements BeanDefinitionRegistryPostProcessor {
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
        for (String channel : List.of("slack", "email", "webhook")) {
            var def = BeanDefinitionBuilder.genericBeanDefinition(GenericChannel.class)
                    .addConstructorArgValue(channel).getBeanDefinition();
            registry.registerBeanDefinition(channel + "Channel", def);
        }
    }
}
```

Used by library authors and for plugin architectures. You'll rarely write it; recognizing it explains how libraries create beans "out of nowhere."

---

## 5. The full startup sequence (the answer that impresses)

Practise narrating this. It's the single highest-value answer in Spring interviews.

```
 1. SpringApplication.run(VigilApplication.class, args)

 2. Deduce application type from the classpath:
       spring-webmvc present?     → SERVLET   → AnnotationConfigServletWebServerApplicationContext
       only spring-webflux?       → REACTIVE
       neither?                   → NONE (plain app / batch job)

 3. Load ApplicationContextInitializers and ApplicationListeners
    from META-INF/spring.factories

 4. Prepare the Environment: merge property sources in precedence order
    (CLI args > env vars > application-{profile}.yml > application.yml > defaults)   [Lesson 12]
    → fire ApplicationEnvironmentPreparedEvent  (this is when Spring Cloud Config would fetch)

 5. Print the banner. Create the ApplicationContext.

 6. ─── DEFINITION PHASE ────────────────────────────────────────────
    a. Register the primary source (@SpringBootApplication class)
    b. ConfigurationClassPostProcessor (a BeanFactoryPostProcessor) runs:
         · component-scans com.vigil.** — reads annotations via ASM
         · processes @Import / @Bean / @Configuration recursively
         · LAST: processes @EnableAutoConfiguration's ImportSelector →
              reads META-INF/spring/...AutoConfiguration.imports from every JAR,
              evaluates @Conditional on each,
              keeps the survivors                                      [Lesson 11]
    c. Other BeanFactoryPostProcessors run (property placeholder resolution, etc.)
    → the BeanFactory now holds ALL bean definitions. Zero business beans instantiated.

 7. Register BeanPostProcessors (including the AOP auto-proxy creator,
    AutowiredAnnotationBeanPostProcessor, CommonAnnotationBeanPostProcessor)

 8. Initialize MessageSource, ApplicationEventMulticaster

 9. onRefresh(): create the embedded web server (Tomcat) — but don't accept traffic yet

10. ─── INSTANTIATION PHASE ─────────────────────────────────────────
    finishBeanFactoryInitialization(): for every non-lazy singleton, in dependency order:
       a. resolve constructor args (recursively creating dependencies)
       b. instantiate                                    ← reflection: ctor.newInstance()
       c. populate fields (@Autowired fields, @Value)
       d. BeanPostProcessor.postProcessBeforeInitialization()
       e. @PostConstruct → InitializingBean.afterPropertiesSet() → custom initMethod
       f. BeanPostProcessor.postProcessAfterInitialization()   ← **PROXIES CREATED HERE**
       → bean is ready, cached in the singleton map

11. finishRefresh(): start Lifecycle beans, publish ContextRefreshedEvent,
    Tomcat starts accepting connections

12. Run ApplicationRunner / CommandLineRunner beans

13. Publish ApplicationReadyEvent  ← the correct hook for "app is fully up" work

14. Register the JVM shutdown hook → on SIGTERM: stop accepting traffic,
    publish ContextClosedEvent, call @PreDestroy / DisposableBean, close pools
```

**Three facts from this sequence worth memorizing:**

1. **Auto-configuration is processed *last*** among configuration classes. That ordering is *why* `@ConditionalOnMissingBean` works — by the time auto-config is evaluated, all your own beans are already defined, so "did the user define one?" is answerable.
2. **Proxies are created at step 10f**, after initialization. So inside `@PostConstruct`, `this` is the raw object — meaning **`@Transactional` and `@Async` don't work in `@PostConstruct`** (self-invocation, again). Use `ApplicationReadyEvent` for work that needs proxied behaviour.
3. **The web server starts before beans finish**? No — it's *created* at step 9 and *starts accepting traffic* at step 11. So a request can never hit a half-initialized context. Good to know when someone claims otherwise.

---

## 6. Bean naming, overriding, and the errors you'll actually see

**Default names:** the class name with a lowercased first letter (`MonitorService` → `monitorService`), or the method name for `@Bean`. Special case: if the first *two* letters are both uppercase, the name is unchanged (`URLShortener` → `URLShortener`) — an obscure rule that occasionally explains a mystery.

**Bean overriding is DISABLED by default in Spring Boot 2.1+:**

```
BeanDefinitionOverrideException: Invalid bean definition with name 'monitorService'
... There is already [Generic bean: class [com.vigil.MonitorService]] bound.
```

Two definitions claimed the same name. Fix the duplication (usually a class both `@Component`-annotated *and* declared as a `@Bean`, or two scanned packages overlapping). You *can* set `spring.main.allow-bean-definition-overriding=true`, but that's re-enabling a footgun — the default was changed precisely because silent overriding caused baffling bugs.

**The other three errors, and their real meanings:**

```
NoSuchBeanDefinitionException: No qualifying bean of type 'com.vigil.NotificationChannel'
→ nothing defined that bean. Causes: forgot @Component; class outside the scanned
  package; a @Conditional excluded it; you're in a test slice that doesn't load it.
```

```
NoUniqueBeanDefinitionException: expected single matching bean but found 2: slackChannel,emailChannel
→ ambiguous by type. Fix with @Primary, @Qualifier, or inject List<NotificationChannel>. [Lesson 07]
```

```
BeanCurrentlyInCreationException / "circular reference"
→ A needs B, B needs A, both constructor-injected. [Lesson 07]
```

### Your debugging toolkit (learn these now — they replace guessing)

```properties
# application.properties — prints the full auto-configuration report
debug=true
```

```bash
# Actuator endpoints (add spring-boot-starter-actuator)
management.endpoints.web.exposure.include=beans,conditions,configprops,env,mappings,health
```
- `/actuator/beans` — **every bean, its type, scope, and dependencies.** The definitive answer to "does this bean exist and what got injected?"
- `/actuator/conditions` — which auto-configurations matched and, crucially, **which didn't and why**
- `/actuator/configprops` — all `@ConfigurationProperties` with resolved values
- `/actuator/mappings` — every URL → handler method

```java
// Or just print it yourself during development
Arrays.stream(ctx.getBeanDefinitionNames()).sorted().forEach(System.out::println);
```

**A senior developer's instinct when Spring behaves unexpectedly is to look at `/actuator/conditions` and `/actuator/beans`, not to add annotations at random.** Internalize that.

---

## 7. Build this

Work in your `vigil` project under `com.vigil.playground.container`.

**1. Count your beans.** Add to `VigilApplication`:
```java
public static void main(String[] args) {
    var ctx = SpringApplication.run(VigilApplication.class, args);
    System.out.println("Total beans: " + ctx.getBeanDefinitionCount());
    Arrays.stream(ctx.getBeanDefinitionNames()).sorted()
          .filter(n -> !n.startsWith("org.springframework"))
          .forEach(System.out::println);
}
```
Run it. You wrote maybe 3 classes; you have 150+ beans. **Where did they come from?** Auto-configuration. Now `debug=true` in `application.properties` and re-run — read the "Positive matches" section. Find `DispatcherServletAutoConfiguration` and `JacksonAutoConfiguration`. This is the moment auto-config stops being magic.

**2. Prove the `@Configuration` proxy.** Create:
```java
@Configuration
public class ProxyDemoConfig {
    @Bean String sharedThing() {
        System.out.println("### CREATING sharedThing");
        return "shared-" + UUID.randomUUID();
    }
    @Bean String consumerA() { return "A uses " + sharedThing(); }
    @Bean String consumerB() { return "B uses " + sharedThing(); }
}
```
Wait — three `String` beans will collide on type later, so name them distinctly or use a small wrapper record. Run it: **`### CREATING sharedThing` prints ONCE**, and A and B reference the same UUID.

Now change to `@Configuration(proxyBeanMethods = false)` and re-run. **It prints three times with three different UUIDs.** Imagine that object was a `DataSource`. **Do this experiment — it's a five-minute investment for a permanent understanding.**

**3. See the stereotype difference.** Create a `@Repository` that throws a raw `SQLException`-ish failure and a `@Component` doing the same, and compare what escapes. (Full version comes in Module 6; for now, just note in `/actuator/beans` that the `@Repository` may be proxied.)

**4. Break it on purpose — the three big errors.** Reproduce each and read the message in full:

- **NoSuchBean:** create `@Service class A { A(B b){} }` with `class B` *not* annotated.
- **NoUniqueBean:** create an interface with two `@Component` implementations, inject the interface.
- **Override:** annotate a class `@Component` *and* declare it as a `@Bean` with the same name.
- **Outside scan path:** create `com.othercompany.Thing` with `@Component`, inject it into a `com.vigil` service. Watch it not be found. Then fix it with `@ComponentScan(basePackageClasses = Thing.class)` and see it work.

**Spring's error messages are unusually good.** They often contain the exact fix. Reading them properly is a skill, and most developers don't have it.

**5. Explore Actuator.** Hit `/actuator/beans` and search for `monitorService`. Note the `dependencies` array — that's your object graph, rendered.

---

## 8. Interview traps

**Q1. What is a Spring bean?**
An object whose lifecycle the container manages: created, injected, possibly proxied, lifecycle-callbacked, and destroyed by Spring. Not a special type — no interface, no base class, no JavaBean conventions required.

**Q2. `BeanFactory` vs `ApplicationContext`?**
`BeanFactory` is the minimal container (lazy `getBean`). `ApplicationContext` extends it with eager singleton instantiation, events, i18n, resource loading, automatic `BeanPostProcessor` detection, and `Environment` access. You always use `ApplicationContext`; eager instantiation is deliberate so misconfiguration fails at startup.

**Q3. `@Component` vs `@Bean`?**
`@Component` marks a class you own for component scanning. `@Bean` is a method in a `@Configuration` class that returns an object — required for third-party classes, conditional creation, or multiple instances of one type. `@Component` is discovered; `@Bean` is declared.

**Q4. Are `@Service`, `@Repository` and `@Component` the same?**
Not quite. `@Service` adds nothing functional (semantic only). `@Repository` enables **exception translation** to Spring's `DataAccessException` hierarchy via a post-processor proxy. `@Controller` makes the class a request-handler target. So: `@Service` ≈ `@Component`; the others carry real behaviour.

**Q5. What does `proxyBeanMethods = false` change, and why does Spring's own auto-config use it?**
It disables the CGLIB proxy on the `@Configuration` class, so calling one `@Bean` method from another executes the method again instead of returning the singleton — potentially creating duplicate objects (e.g. multiple connection pools). Spring's auto-configurations use it because skipping proxy generation for hundreds of config classes measurably improves startup, and they always pass dependencies as method parameters, which is safe.

**Q6. `BeanFactoryPostProcessor` vs `BeanPostProcessor`?**
`BeanFactoryPostProcessor` runs **before** instantiation and modifies bean *definitions* (e.g. resolving `${...}` placeholders). `BeanPostProcessor` runs **during** instantiation on bean *instances*, and can return a substitute object — which is how AOP proxies are created.

**Q7. Why aren't your beans being found?**
Almost always: the class is outside the `@SpringBootApplication` package tree (component scan covers that package and below only); or it's missing a stereotype annotation; or a `@Conditional` excluded it; or you're in a test slice (`@WebMvcTest`) that loads only part of the context.

**Q8. Why is bean overriding disabled by default?**
Because silently replacing a bean definition caused bugs that were extremely hard to trace — behaviour depended on scan/registration order. Since Boot 2.1 a duplicate name throws `BeanDefinitionOverrideException`, forcing you to resolve the ambiguity explicitly.

**Q9. When are AOP proxies created, and what does that imply for `@PostConstruct`?**
In `BeanPostProcessor.postProcessAfterInitialization`, i.e. *after* initialization callbacks. So inside `@PostConstruct` you hold the raw target — `@Transactional`, `@Async` and `@Cacheable` on self-calls there won't apply. Use `ApplicationReadyEvent` (or an `ApplicationRunner`) for startup work that needs proxied behaviour.

**Q10. What is a `FactoryBean` and how do you get the factory itself?**
A bean that produces another bean, for complex construction. `getBean("name")` returns the product; `getBean("&name")` returns the factory. Spring Data uses `FactoryBean`s to create repository implementations for your interfaces.

**Q11. Walk me through Spring Boot startup.**
Give the sequence from §5, and make sure you hit: application-type deduction from the classpath, environment/property-source preparation, the definition phase (component scan → `@Import`/`@Bean` → auto-config's `ImportSelector` evaluated **last** so `@ConditionalOnMissingBean` works), then the instantiation phase with post-processors and proxy creation, then web server start, then `ApplicationReadyEvent`.

---

## 9. Retention check

1. What's the difference between a bean definition and a bean instance, and name two behaviours that only make sense once you know they're separate phases?
2. You add `@Configuration(proxyBeanMethods = false)` to a class where one `@Bean` method calls another. What breaks, and how would you notice in production?
3. Which stereotype annotation adds real behaviour beyond `@Component`, and what behaviour?
4. Why is auto-configuration processed last, and which annotation depends on that ordering?
5. A colleague says "my `@Autowired` service is null in `@PostConstruct`" — actually, worse: "`@Transactional` doesn't work in `@PostConstruct`." Explain why, using step 10f.
6. Which two Actuator endpoints do you hit first when a bean isn't behaving?

---

**Next:** [Lesson 07 — Dependency injection, deeply](07-dependency-injection-deep.md) — constructor vs field injection (with the real argument, not the slogan), how ambiguity is resolved, and how to fix circular dependencies properly.

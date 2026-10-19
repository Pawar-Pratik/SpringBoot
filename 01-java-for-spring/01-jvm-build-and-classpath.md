# Lesson 01 — The JVM, the classpath & build tools

> **Why this lesson exists:** 90% of "Spring Boot won't start" problems are not Spring problems. They're classpath problems, dependency-version problems, or "I don't know what a JAR is" problems. Engineers who understand this layer debug in 2 minutes what others debug in 2 hours.

**Time:** ~50 minutes · **Prereq:** you can program in some language

---

## 1. The idea in one sentence

> **Java resolves almost nothing at compile time that matters to Spring: your dependencies are found by name, at runtime, from an ordered search path — so most "Spring Boot won't start" failures are the classpath answering a lookup with the wrong class, or with no class at all.**

Everything in this lesson is a consequence of that sentence. The JIT, the fat JAR's custom classloader, Maven's nearest-wins rule and Boot's BOM all exist because name resolution happens late, and late resolution is only safe if something curates what's on the path.

---

## 2. The mental model: Java's execution pipeline

Coming from Python/JS, the biggest shift is that Java has a **compile step** and runs on a **virtual machine**.

```
Foo.java        →  [ javac ]  →  Foo.class        →  [ java ]  →  running program
(source text)                   (JVM bytecode)                    (JIT-compiled to
                                                                   machine code)
```

Three things to internalize:

**1. `javac` produces bytecode, not machine code.** `Foo.class` contains instructions for an imaginary CPU — the Java Virtual Machine. This is why "write once, run anywhere" works: the same `.class` file runs on Windows, Linux, ARM Macs. The JVM is the thing that's platform-specific.

**2. The JVM compiles *again*, at runtime.** The JIT (Just-In-Time) compiler watches which methods run often ("hot" methods) and compiles those to native machine code, aggressively optimized using information only available at runtime.

> **Fun fact:** This is why a Java server is *slow for the first few seconds and then fast*. The JIT hasn't warmed up. Serious benchmarks discard the first N iterations for exactly this reason. It's also why Java, despite being "interpreted," can beat C in some long-running workloads — the JIT knows your actual runtime data distribution, and a static compiler doesn't. It can inline a virtual call because it observed that in practice only one implementation is ever used — then de-optimize if a second one shows up.

**3. Everything is loaded lazily by a ClassLoader.** A class isn't loaded until it's first needed. This becomes *extremely* relevant to Spring: `NoClassDefFoundError` at minute 40 of uptime is a normal Java experience.

### Compile time vs runtime — the distinction you must never blur

| | Compile time (`javac`) | Runtime (`java`) |
|---|---|---|
| Sees | Your source + the **compile classpath** | Loaded classes + the **runtime classpath** |
| Catches | Type errors, syntax, missing methods | Missing classes, wrong versions, nulls, config errors |
| Spring's role | Almost none | **Everything** |

**Spring does virtually all of its work at runtime.** It doesn't generate code you compile against (mostly). It starts up, reads annotations via reflection, and assembles objects. This is why:

- A wiring mistake compiles fine and explodes on startup.
- Your IDE can't autocomplete a bean name.
- Renaming a bean via a string in an annotation breaks nothing at compile time and everything at boot.

Hold this thought. It explains the *entire* character of Spring development.

---

## 3. The classpath — the single most important runtime concept

The **classpath** is the list of places the JVM searches for classes. That's it. It's a search path, like `PATH` for executables or `sys.path` in Python.

```bash
java -cp "app.jar;lib/spring-core.jar;lib/jackson.jar" com.vigil.Application
```

(On Windows the separator is `;`. On Linux/Mac it's `:`. A cross-platform bug generator since 1996.)

When your code says `new ObjectMapper()`, the JVM asks the ClassLoader: *"find `com/fasterxml/jackson/databind/ObjectMapper.class`."* The loader walks the classpath entries in order and uses the **first match**.

### The three errors this produces, and what each one means

| Error | Meaning | Typical cause |
|---|---|---|
| `ClassNotFoundException` | Class wasn't found when looked up **by name at runtime** (reflection) | Missing dependency; Spring trying to load an optional class |
| `NoClassDefFoundError` | Compiler saw it, runtime can't find it | `provided` scope dependency missing at runtime; jar not shipped |
| `NoSuchMethodError` | Class found, but the method signature isn't there | **Version conflict** — you compiled against v2, you're running v1 |

`NoSuchMethodError` is the one that teaches you humility. It almost always means two libraries dragged in different versions of a third library, and the wrong one won the classpath. Which brings us to dependency management.

> **Interview gold:** *"What's the difference between `ClassNotFoundException` and `NoClassDefFoundError`?"* is a classic filter question. Answer: `ClassNotFoundException` is a checked exception from an *explicit* dynamic lookup (`Class.forName`). `NoClassDefFoundError` is an `Error` thrown by the JVM when a class that was present at compile time is absent (or failed to initialize) at runtime. Bonus points: `NoClassDefFoundError` can also mean the class *was* found but its static initializer threw — a nasty case where the real error is buried in an earlier log line.

---

## 4. JAR, fat JAR, and how Spring Boot runs

A **JAR** is a ZIP file with a manifest. Genuinely — rename `app.jar` to `app.zip` and open it.

```
app.jar
├── META-INF/MANIFEST.MF     ← metadata; can declare Main-Class
└── com/vigil/Application.class
```

Classic Java deployment was: one thin JAR of your code + a `lib/` folder of 80 dependency JARs + a shell script assembling the classpath. Or worse: a WAR file you deployed *into* an application server (Tomcat, JBoss, WebSphere) that someone else administered.

**Spring Boot's original killer feature was killing that.** It produces an **executable fat JAR** containing your code *and* all dependencies *and* an embedded web server:

```
vigil.jar
├── META-INF/MANIFEST.MF                ← Main-Class: JarLauncher
├── org/springframework/boot/loader/...  ← Boot's own launcher
└── BOOT-INF/
    ├── classes/                        ← YOUR code
    └── lib/                            ← every dependency JAR, nested
```

```bash
java -jar vigil.jar
```

One command. No app server to install. No classpath script. This is the thing that made Spring Boot spread like wildfire in 2014 and it's why "microservices in Java" became practical at all.

> **Fun fact:** The JVM cannot natively load a JAR-inside-a-JAR. So Spring Boot ships its own `LaunchedClassLoader`: the manifest's `Main-Class` is Boot's `JarLauncher`, which builds a classloader that can read nested JARs, then reflectively invokes *your* `main`. Your `Start-Class` is in the manifest too. Look at your own manifest after Lesson 11 — seeing this makes "it's just a JAR" click permanently.

> **Boot 4 delta:** The nested-JAR layout and loader were reworked in Boot 3.2+ (the "classic" loader is legacy). The concept is unchanged; only internals moved.

---

## 5. Maven: the build tool you'll use

Java's ecosystem has two build tools. You'll learn Maven, because ~70% of enterprise Spring codebases use it and it's declarative enough that you can *read* any project instantly.

| | **Maven** | **Gradle** |
|---|---|---|
| Config | XML (`pom.xml`), declarative | Groovy/Kotlin DSL, programmatic |
| Learning | Rigid, therefore predictable | Flexible, therefore variable |
| Speed | Slower | Faster (incremental, build cache, daemon) |
| Reading someone else's | Easy — always the same shape | Depends how clever they were |
| Industry | Enterprise, banks, most Spring tutorials | Android (mandatory), newer startups |

**Learn Maven, be able to read Gradle.** Interviews rarely test build-tool syntax; they test whether you understand dependency *scopes* and *conflict resolution*, which are conceptually the same in both.

### Anatomy of a `pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" ...>
    <modelVersion>4.0.0</modelVersion>

    <!-- Inherit dependency versions + plugin config from Spring Boot -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.5.0</version>
        <relativePath/>
    </parent>

    <!-- GAV: the coordinates that uniquely identify YOUR artifact -->
    <groupId>com.vigil</groupId>          <!-- reverse-domain namespace -->
    <artifactId>vigil</artifactId>        <!-- project name -->
    <version>0.0.1-SNAPSHOT</version>     <!-- SNAPSHOT = mutable dev version -->

    <properties>
        <java.version>21</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <!-- NO VERSION: inherited from the parent's dependencyManagement -->
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

**GAV coordinates** — `groupId:artifactId:version` — are how every Java dependency on Earth is addressed. `org.springframework.boot:spring-boot-starter-web:3.5.0` is a globally unique address in Maven Central.

**Why no versions on the dependencies?** `spring-boot-starter-parent` contains a `<dependencyManagement>` block declaring tested-compatible versions for ~450 libraries. This is the *second* great Spring Boot innovation and it's massively underrated: before it, "Spring version 4.1 + Hibernate 5.2 + Jackson 2.6 + Tomcat 8.0" compatibility was a research project you did yourself, every upgrade. Boot's curated BOM ("Bill of Materials") eliminated an entire genre of suffering.

> **Interview trap:** *"How does Spring Boot manage dependency versions?"* Weak answer: "the parent POM." Strong answer: "`spring-boot-starter-parent` inherits from `spring-boot-dependencies`, which is a BOM — a POM with a `dependencyManagement` section pinning versions for the whole ecosystem. That means declaring a dependency without a version is safe, and overriding one version is a deliberate act you do by setting the property, e.g. `<postgresql.version>`. If you can't use the parent — common in companies with a corporate parent POM — you import the BOM in `dependencyManagement` with `<type>pom</type><scope>import</scope>` instead."

### Dependency scopes

| Scope | On compile classpath? | On runtime classpath? | Shipped in fat JAR? | Use |
|---|---|---|---|---|
| `compile` (default) | ✅ | ✅ | ✅ | Normal libraries |
| `runtime` | ❌ | ✅ | ✅ | JDBC drivers — you code to the JDBC API, not the driver |
| `provided` | ✅ | ❌ | ❌ | Container supplies it (servlet API in a WAR); Lombok |
| `test` | test only | test only | ❌ | JUnit, Mockito, Testcontainers |
| `import` | n/a | n/a | n/a | Only for BOMs in `dependencyManagement` |

The pattern worth understanding: **`runtime` scope for drivers is a deliberate design signal.** By keeping the Postgres driver off your compile classpath, Maven makes it *impossible* for you to accidentally `import org.postgresql.*` in business code. The build tool enforces the abstraction. That's good engineering, and noticing it is a senior-level observation.

### The commands you'll actually use

```bash
./mvnw clean package        # wipe target/, compile, test, build the JAR
./mvnw spring-boot:run      # run the app without building a JAR
./mvnw test                 # tests only
./mvnw dependency:tree      # THE debugging command — learn it now
./mvnw clean install -DskipTests   # install to local ~/.m2 repo, skip tests
```

`mvnw` is the **Maven wrapper** — a script committed to your repo that downloads the exact Maven version the project needs. Always use `./mvnw`, never `mvn`, so every developer and your CI server use identical builds. (Gradle has `gradlew`.)

### Maven's lifecycle phases (in order)

`validate → compile → test → package → verify → install → deploy`

Running a phase runs **everything before it**. So `mvn package` compiles and tests first. This is why `mvn clean package` is the standard incantation — `clean` isn't part of the default lifecycle, so you name it explicitly.

---

## 6. Transitive dependencies & conflict resolution

You declare `spring-boot-starter-web`. You get ~35 JARs. Because dependencies have dependencies — **transitively**.

```
spring-boot-starter-web
├── spring-boot-starter (→ spring-core, spring-context, logging...)
├── spring-boot-starter-json
│   └── jackson-databind
│       ├── jackson-core
│       └── jackson-annotations
├── spring-boot-starter-tomcat
│   └── tomcat-embed-core
├── spring-web
└── spring-webmvc
```

Now the problem: **library A needs Jackson 2.15, library B needs Jackson 2.17.** Only one version can be on the classpath (Java has no per-module versioning — unlike Node's `node_modules` nesting).

**Maven's rule: "nearest definition wins."** The version at the shallowest depth in the dependency tree is used. Ties are broken by declaration order.

This rule is *not* "highest version wins" (that's Gradle's default), and that difference bites people. Maven can therefore pick an **older** version and give you `NoSuchMethodError`.

**How to fix conflicts:**

```bash
./mvnw dependency:tree -Dincludes=com.fasterxml.jackson.core:jackson-databind
```

Then either:

```xml
<!-- 1. Pin the version explicitly (declare it directly = depth 1 = nearest) -->
<dependency>
  <groupId>com.fasterxml.jackson.core</groupId>
  <artifactId>jackson-databind</artifactId>
  <version>2.17.2</version>
</dependency>

<!-- 2. Or exclude the bad transitive path -->
<dependency>
  <groupId>com.example</groupId>
  <artifactId>legacy-lib</artifactId>
  <exclusions>
    <exclusion>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-databind</artifactId>
    </exclusion>
  </exclusions>
</dependency>
```

> **This is "JAR hell" / "dependency hell."** Java's answer is discipline (BOMs, `dependency:tree`, `mvn dependency:analyze`) rather than a technical fix. Spring Boot's BOM is the single biggest mitigation in the ecosystem — which is a large part of *why* Spring Boot won.

> **Fun fact:** Java 9's module system (JPMS, `module-info.java`) was designed partly to solve this. It largely failed to be adopted in application code — the ecosystem was too invested in the classpath, and Spring/Hibernate rely on deep reflection that modules restrict. You will almost never write `module-info.java` in a Spring app. Know it exists, know why it's absent; that's a genuinely impressive thing to be able to explain.

---

## 7. `mvnw`, `.m2` and where JARs actually live

```
~/.m2/
├── settings.xml       ← credentials, mirrors, corporate proxy config
└── repository/
    └── org/springframework/boot/spring-boot-starter-web/3.5.0/
        ├── spring-boot-starter-web-3.5.0.jar
        └── spring-boot-starter-web-3.5.0.pom
```

Maven downloads once, caches forever, in a directory laid out exactly like the GAV coordinates. Two consequences you'll hit:

- **`mvn install` puts YOUR artifact here**, so another local project can depend on it. Useful for multi-repo work.
- **Corrupt cache = weird errors.** `Could not resolve` or checksum failures with no obvious cause → delete the offending folder under `~/.m2/repository` and rebuild. Nuking all of `.m2/repository` is a legitimate, if slow, last resort. (Windows: `C:\Users\<you>\.m2`.)

In companies, `settings.xml` points at an internal **Nexus/Artifactory** mirror. All dependencies come from there — for caching, license auditing, and so a supply-chain attack on Maven Central doesn't reach production directly. Knowing this term exists puts you ahead of most juniors.

---

## 8. Build & break: your first Spring Boot project

We'll create the skeleton that becomes Vigil.

**Step 1 — Generate it.** Go to [start.spring.io](https://start.spring.io) and set:

| Field | Value |
|---|---|
| Project | Maven |
| Language | Java |
| Spring Boot | 3.5.x (latest 3.5 release, not SNAPSHOT/M) |
| Group | `com.vigil` |
| Artifact | `vigil` |
| Packaging | Jar |
| Java | 21 |
| Dependencies | **Spring Web**, **Spring Boot Actuator** |

Download, unzip into `E:\Learnings\spring boot\vigil`, open in IntelliJ.

> `start.spring.io` is not a toy — it's the standard way professionals start Spring projects, and it's an API. `curl https://start.spring.io/starter.zip -d dependencies=web -d javaVersion=21 -o app.zip` works. IntelliJ's New Project wizard calls the same service.

**Step 2 — Look at what you got.**

```
vigil/
├── .mvn/wrapper/           ← the wrapper's own jar + properties
├── mvnw / mvnw.cmd         ← use these, not `mvn`
├── pom.xml
└── src/
    ├── main/
    │   ├── java/com/vigil/VigilApplication.java
    │   └── resources/
    │       ├── application.properties
    │       ├── static/       ← served as-is at /
    │       └── templates/    ← server-rendered views
    └── test/
        └── java/com/vigil/VigilApplicationTests.java
```

`src/main/java` and `src/test/java` is the **Maven Standard Directory Layout**. Never fight it. Every Java tool on Earth assumes it.

**Step 3 — Run it three ways** (do all three, they teach different things):

```bash
./mvnw spring-boot:run
```

```bash
./mvnw clean package
java -jar target/vigil-0.0.1-SNAPSHOT.jar
```

Third way: click ▶ next to `main` in IntelliJ.

You should see the ASCII banner and `Tomcat started on port(s): 8080`. Hit `http://localhost:8080/actuator/health` — you'll get `{"status":"UP"}`.

**Step 4 — Inspect your own fat JAR.**

```bash
./mvnw clean package
```
```bash
unzip -l target/vigil-0.0.1-SNAPSHOT.jar | head -40
```

(No `unzip` on Windows? PowerShell: `Expand-Archive target\vigil-0.0.1-SNAPSHOT.jar -DestinationPath target\unpacked`, then browse it. Or just rename a copy to `.zip` and double-click.)

Find `BOOT-INF/lib/` and count the JARs. Open `META-INF/MANIFEST.MF` in a text editor and read `Main-Class` and `Start-Class`. **Do this — it makes "executable JAR" concrete rather than magical.**

**Step 5 — Read your dependency tree.**

```bash
./mvnw dependency:tree
```

You declared 2 dependencies. Count what you got. Find `tomcat-embed-core` — that's your web server, inside your JAR. Find `jackson-databind` — that's how objects become JSON.

**Step 6 — Break it on purpose.**

Add this to `pom.xml` and rebuild:

```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.9.0</version>
</dependency>
```

Run the app and hit an endpoint that returns JSON. Read the exception. You just manufactured a version conflict — "nearest wins" chose your ancient 2.9.0 over Boot's curated version. **Remove it afterward.** You now have a real, felt understanding of why the BOM exists.

### Explain out loud (60s)

1. The two compilation steps in a Java program's life, and who performs each.
2. What the classpath is, in one sentence, and why first-match-wins makes duplicate classes dangerous.
3. `ClassNotFoundException` vs `NoClassDefFoundError` vs `NoSuchMethodError` — the discriminator for each.
4. Why the JVM can't load a JAR inside a JAR, and what Boot does about it.
5. Maven's conflict rule, and why it can hand you an *older* library than you asked for.

---

## 9. Production rules

| Do / Never | Why |
|---|---|
| **Always `./mvnw`, never `mvn`** | The wrapper pins the exact Maven version in the repo, so your machine and CI produce the same build instead of "works on mine" |
| **Never write a `<version>` on a Boot-managed dependency** | The BOM already pins a tested-compatible version; a hand-written one silently un-pins that library from the curated set |
| **Override a managed version via the property (`<jackson.version>`), not by re-declaring the dependency** | The property moves every artifact in that group together; re-declaring moves one and leaves its siblings behind — which is exactly how `NoSuchMethodError` is born |
| **JDBC drivers get `runtime` scope** | Keeping driver classes off the compile classpath makes it impossible to `import org.postgresql.*` in business code — the build tool enforces the abstraction |
| **Run `dependency:tree` before theorising about a version error** | Nearest-definition-wins means the answer is always visible in the tree; guessing costs hours, reading it costs a minute |
| **Fix conflicts by declaring at depth 1 or excluding the bad path** | Maven resolves by *depth*, not by highest version, so a direct declaration is the only reliable way to win |
| **Never `mvn install` in CI** | It writes to a shared local repository, making builds order-dependent and letting a stale artifact pass a build that should fail; use `package` or `verify` |
| **On `NoClassDefFoundError`, scan earlier log lines before adding a dependency** | It also fires when a class *was* found but its static initializer threw — the real exception is buried further up |
| **Never rename or restructure `src/main/java`** | Every Java tool on Earth assumes the Maven Standard Directory Layout; fighting it breaks tooling you haven't met yet |

---

## 10. Interview traps

Cover the answers. Answer out loud first.

**Q1. What is the classpath, and how does the JVM use it?**
An ordered list of locations (directories and JARs) the ClassLoader searches for `.class` files, resolved by fully-qualified name, first match wins. Consequences: duplicate classes are silently shadowed, and ordering can determine behaviour.

**Q2. Difference between `ClassNotFoundException`, `NoClassDefFoundError`, and `NoSuchMethodError`?**
See the table in §2. The key discriminator: `ClassNotFoundException` = explicit dynamic lookup failed (checked exception). `NoClassDefFoundError` = linkage failure for a class the compiler resolved (an `Error`); can also mask a failed static initializer. `NoSuchMethodError` = class present, signature absent → version mismatch.

**Q3. How does Spring Boot's executable JAR work? Why isn't it a normal JAR?**
Fat JAR with dependencies nested under `BOOT-INF/lib`. The JVM can't load nested JARs, so the manifest's `Main-Class` is Boot's `JarLauncher`, which installs a custom classloader capable of reading nested entries, then reflectively invokes the real `Start-Class`.

**Q4. Fat JAR vs WAR — when would you still use a WAR?**
Fat JAR: self-contained, embedded server, one process per app — the default for cloud/containers/microservices. WAR: deployed into an external servlet container; still used where an ops team mandates a shared app server, or licensing/compliance requires it. Boot supports both; WAR requires extending `SpringBootServletInitializer` and marking the embedded server `provided`.

**Q5. How does Maven resolve a version conflict?**
Nearest-definition-wins (shallowest depth), ties broken by declaration order. **Not** highest-version-wins — that's Gradle. Fix by declaring directly (depth 1), excluding, or using `dependencyManagement` to force a version tree-wide. Diagnose with `dependency:tree`.

**Q6. What's `dependencyManagement` vs `dependencies`?**
`dependencies` = actually on the classpath. `dependencyManagement` = *if* something (including a transitive path) pulls this in, use this version — declares nothing by itself. Boot's parent uses it to pin ~450 versions.

**Q7. `provided` vs `runtime` scope?**
`provided`: needed to compile, supplied by the environment at runtime, not packaged (servlet API in a WAR, Lombok). `runtime`: not needed to compile, required to run, packaged (JDBC drivers). The JDBC case is intentional — it prevents you importing driver classes in business code.

**Q8. Why is Java "slow to start but fast later"?**
Class loading + JIT warm-up. The interpreter runs first; the JIT profiles hot methods and compiles them to optimized native code (with speculative optimizations it can de-optimize). Relevant to Spring: startup time matters for serverless and autoscaling, which is why CDS/AOT/GraalVM native images exist (Lesson 36).

**Q9. What does `mvn clean install` do that `mvn clean package` doesn't?**
`install` runs `package` and then copies the artifact + POM into your local `~/.m2/repository`, making it resolvable by other local builds. In CI you usually want `verify` or `package`; `install` polluting a shared cache is a known CI smell.

---

## 11. Retention check

Say these out loud without looking:

1. What are the two compilation steps in a Java program's life, and who performs each?
2. Why can't the JVM load a JAR inside a JAR, and what does Spring Boot do about it?
3. You get `NoSuchMethodError: com.fasterxml.jackson.databind.ObjectMapper.someMethod`. Give your first three debugging actions.
4. Why do you not write a `<version>` for `spring-boot-starter-web`?
5. Why is the Postgres driver `runtime` scope instead of `compile`?

---

**Next:** [Lesson 02 — Java essentials for Spring](02-java-essentials.md) — the language features you'll use in every file, with no filler.

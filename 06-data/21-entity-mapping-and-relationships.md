# Lesson 21 — Entity mapping & relationships

> **Why this lesson exists:** Relationships are where JPA gets genuinely hard. Bidirectional associations that don't update, `equals`/`hashCode` that break `Set`s, `CascadeType.ALL` that deletes half your database, enum ordinals that corrupt on a code change. Each of these has caused real production incidents. This lesson is the field guide.

**Time:** ~85 minutes · **Prereq:** Lesson 20

---

## 1. The idea in one sentence

> **An object graph and a set of rows are two models of the same facts, and Hibernate reconciles them only through the *owning* side — so every trap in this lesson is a place where the Java object says one thing and the foreign key says another.**

Update the wrong side and nothing persists. Base equality on a value that changes and the object vanishes from its own `Set`. Cascade in the wrong direction and deleting a monitor deletes the customer. Same question each time: which model is authoritative here?

---

## 2. Anatomy of an entity

```java
@Entity
@Table(name = "monitor",
       indexes = {
           @Index(name = "idx_monitor_org_status", columnList = "org_id, status"),
           @Index(name = "idx_monitor_next_check", columnList = "next_check_at")
       },
       uniqueConstraints = @UniqueConstraint(name = "uq_monitor_org_name",
                                             columnNames = {"org_id", "name"}))
@Getter @Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)      // ★ JPA requires it; keep it non-public
public class Monitor {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "monitor_seq")
    @SequenceGenerator(name = "monitor_seq", sequenceName = "monitor_id_seq", allocationSize = 50)
    private Long id;

    @Column(nullable = false, length = 100)
    private String name;

    @Column(nullable = false, columnDefinition = "text")
    private String url;

    @Enumerated(EnumType.STRING)                        // ★ NEVER the default ORDINAL
    @Column(nullable = false, length = 20)
    private MonitorStatus status = MonitorStatus.PENDING;

    @Column(name = "interval_seconds", nullable = false)
    private int intervalSeconds;

    @Column(name = "last_checked_at")
    private Instant lastCheckedAt;                       // ★ Instant, not LocalDateTime

    @Version                                            // ★ optimistic locking, nearly free
    private long version;

    @CreatedDate  @Column(updatable = false)
    private Instant createdAt;

    @LastModifiedDate
    private Instant updatedAt;

    @Transient                                          // not persisted
    private transient BigDecimal cachedUptime;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "org_id", nullable = false,
                foreignKey = @ForeignKey(name = "fk_monitor_org"))
    private Organization organization;
}
```

### The requirements JPA imposes

| Requirement | Why | Consequence |
|---|---|---|
| **A no-arg constructor** | Hibernate instantiates reflectively before populating | **Records can't be entities** |
| **Not `final`** (class or methods) | Proxy = generated subclass (Lesson 09) | `final` silently disables lazy loading |
| **An `@Id`** | Identity | |
| **Mutable fields** | Dirty checking sets them | No immutable entities |

**Which answers "why not records for entities?"** — no no-arg constructor, `final` class, immutable fields. All three are incompatible with Hibernate's proxying and dirty-checking model. **Records are perfect for DTOs and projections, wrong for entities.** Common interview question.

**Keep the no-arg constructor `protected`.** JPA needs it; your code shouldn't use it. `@NoArgsConstructor(access = PROTECTED)` documents that intent.

### Field vs property access

Hibernate uses **field access** if `@Id` is on a field, **property access** if it's on a getter. Field access is the norm and it matters: with property access, Hibernate calls your getters, so any logic in them (lazy initialization, computed values, side effects) runs during loading and dirty checking — a real source of surprising behaviour.

### `@Enumerated` — the one that has caused real outages

```java
@Enumerated(EnumType.ORDINAL)     // ❌ THE DEFAULT. Stores 0, 1, 2...
@Enumerated(EnumType.STRING)      // ✅ Stores "UP", "DOWN", "DEGRADED"
```

```java
// Version 1
enum MonitorStatus { UP, DOWN, PAUSED }        // UP=0, DOWN=1, PAUSED=2

// Version 2 — a "harmless" addition, alphabetically placed
enum MonitorStatus { DEGRADED, UP, DOWN, PAUSED }   // DEGRADED=0, UP=1, DOWN=2, PAUSED=3
```

**Every existing row now means something different.** Rows storing `0` were `UP` and are now `DEGRADED`. There is no error, no migration, no warning — your entire historical dataset is silently reinterpreted. **Always `EnumType.STRING`.** The extra bytes are irrelevant; the failure mode is catastrophic and undetectable.

> **A subtlety worth knowing:** `STRING` still breaks if you *rename* an enum constant. If you need full decoupling, use an `@Converter` mapping enum ↔ a stable code (`"UP"` → `"U"`), which is what some teams do for long-lived schemas. Mentioning this shows real depth.

### `@Version` — optimistic locking for free

```java
@Version
private long version;
```

Hibernate now includes the version in every UPDATE:
```sql
UPDATE monitor SET name = ?, version = 6 WHERE id = ? AND version = 5
```
Zero rows updated → someone else changed it → `OptimisticLockingFailureException`.

**Add `@Version` to every entity users can concurrently edit.** It costs one column and prevents lost updates — the classic "two admins edit the same monitor, the second silently overwrites the first" bug. Return **409 Conflict** to the client so they can re-read and retry (Lesson 16). Deep dive in Lesson 23.

### Other column mappings you'll want

```java
@Column(precision = 10, scale = 2)  BigDecimal amount;   // NUMERIC(10,2) — money

@Lob                                 String hugeText;
@Basic(fetch = FetchType.LAZY)       byte[] blob;         // don't load blobs by default

@JdbcTypeCode(SqlTypes.JSON)                              // Hibernate 6 — first-class JSON!
@Column(columnDefinition = "jsonb")
private Map<String, String> expectedHeaders;

@Convert(converter = EncryptedStringConverter.class)      // transparent encryption at rest
private String apiKeySecret;

@Embedded
private Address address;                                  // flattened into the same table

@ElementCollection
@CollectionTable(name = "monitor_tag", joinColumns = @JoinColumn(name = "monitor_id"))
@Column(name = "tag")
private Set<String> tags = new HashSet<>();               // a table of simple values
```

**`@JdbcTypeCode(SqlTypes.JSON)` is a genuine Hibernate 6 quality-of-life win.** Before it you needed a custom `UserType` or a third-party library to map `jsonb`. **Vigil uses it** for per-monitor configuration (expected headers, keyword assertions) — a schema-less blob where a rigid column set would be wrong.

**`AttributeConverter` for encryption at rest** is worth knowing:
```java
@Converter
public class EncryptedStringConverter implements AttributeConverter<String, String> {
    public String convertToDatabaseColumn(String attr)  { return crypto.encrypt(attr); }
    public String convertToEntityAttribute(String db)   { return crypto.decrypt(db); }
}
```
Transparent field-level encryption. Caveat to state: **you can no longer query or index that column meaningfully** — encrypted values don't sort or match. That trade-off is the interesting part.

---

## 3. Relationships

### `@ManyToOne` — the one you'll use most

```java
@Entity
public class Monitor {
    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "org_id", nullable = false)
    private Organization organization;                    // FK lives on THIS table
}
```

**The owning side holds the foreign key.** For `@ManyToOne` that's always this entity's table. Simple, efficient, no join table. **Always `LAZY`** (Lesson 20).

### `@OneToMany` — and the two ways to do it

```java
// Unidirectional @OneToMany — ⚠️ generates a JOIN TABLE by default. Almost never what you want.
@OneToMany
private List<CheckResult> results;        // → creates monitor_check_result join table 😖

// ✅ Fix with @JoinColumn — FK on the child table, no join table
@OneToMany
@JoinColumn(name = "monitor_id")
private List<CheckResult> results;

// ✅✅ Bidirectional — the standard approach
@OneToMany(mappedBy = "monitor", cascade = CascadeType.ALL, orphanRemoval = true)
private List<CheckResult> results = new ArrayList<>();
```

**`mappedBy` is the critical keyword.** It says: *"the other side owns this relationship; the FK is `CheckResult.monitor`."* Without it, Hibernate assumes this side owns it and creates a join table.

### THE BIDIRECTIONAL TRAP — a top-3 JPA interview question

```java
@Entity
public class Monitor {
    @OneToMany(mappedBy = "monitor", cascade = ALL, orphanRemoval = true)
    private List<CheckResult> results = new ArrayList<>();
}

@Entity
public class CheckResult {
    @ManyToOne(fetch = LAZY)
    @JoinColumn(name = "monitor_id")
    private Monitor monitor;                  // ★ THIS is the owning side (holds the FK)
}
```

```java
// ❌ THE BUG
@Transactional
public void addResult(Monitor monitor, CheckResult result) {
    monitor.getResults().add(result);         // updated the INVERSE side only
}
// No INSERT of monitor_id. The FK is null (or the insert fails on a NOT NULL constraint).
// The in-memory object graph looks correct. The database disagrees.
```

**Why:** `mappedBy` means the collection is the **inverse** side, and **Hibernate only persists changes made to the owning side.** Adding to the collection doesn't touch `CheckResult.monitor`, so nothing about the FK changed.

**The fix — a helper method on the parent, always:**

```java
@Entity
public class Monitor {
    @OneToMany(mappedBy = "monitor", cascade = ALL, orphanRemoval = true)
    private List<CheckResult> results = new ArrayList<>();

    public void addResult(CheckResult result) {
        results.add(result);              // keep the object graph consistent
        result.setMonitor(this);           // ★ AND set the owning side — this is what persists
    }

    public void removeResult(CheckResult result) {
        results.remove(result);
        result.setMonitor(null);           // ★ required for orphanRemoval to fire
    }
}
```

**Always write these helpers when you have a bidirectional relationship.** It's not optional style — it's the only way to keep memory and database in agreement. Not doing it is the single most common bidirectional bug.

**Second trap in the same area — the stale in-memory graph:**
```java
@Transactional
public void demo(Monitor m, CheckResult r) {
    r.setMonitor(m);
    resultRepo.save(r);                    // ✅ persists correctly
    log.info("{}", m.getResults().size()); // ❌ still 0 — the collection wasn't updated
}
```
Setting only the owning side persists correctly but leaves the parent's collection stale until the context is cleared and reloaded. Hence: **update both sides, always.**

### `@OneToOne`

```java
@Entity
public class Monitor {
    @OneToOne(mappedBy = "monitor", cascade = ALL, orphanRemoval = true,
              fetch = FetchType.LAZY, optional = false)
    private MonitorConfig config;
}

@Entity
public class MonitorConfig {
    @OneToOne(fetch = LAZY) @JoinColumn(name = "monitor_id", unique = true)
    private Monitor monitor;                  // owning side
}
```

**The lazy `@OneToOne` gotcha, and it's a good one:** a nullable `@OneToOne` on the *inverse* side **cannot be lazy**. Hibernate must query to know whether to give you a proxy or `null` — so it queries eagerly regardless of your annotation. Workarounds: make it `optional = false`, put the FK on this side, or use bytecode enhancement. **Knowing this explains a lot of "my lazy association isn't lazy" confusion.**

**Design advice: prefer `@ManyToOne` over `@OneToOne`** even for 1:1 relationships, with a unique constraint on the FK. It avoids the laziness problems entirely and is easier to evolve if the cardinality changes.

### `@ManyToMany` — and why to avoid it

```java
// ❌ The naive version
@ManyToMany
@JoinTable(name = "user_organization",
           joinColumns = @JoinColumn(name = "user_id"),
           inverseJoinColumns = @JoinColumn(name = "org_id"))
private Set<Organization> organizations = new HashSet<>();
```

**The problem:** the join table can hold *only* the two FKs. The moment you need a `role`, a `joined_at`, or an `invited_by` — and you always do — you must restructure.

**✅ Model the join table as an entity:**
```java
@Entity
@Table(name = "membership",
       uniqueConstraints = @UniqueConstraint(columnNames = {"user_id", "org_id"}))
public class Membership {
    @Id @GeneratedValue private Long id;

    @ManyToOne(fetch = LAZY) @JoinColumn(name = "user_id") private User user;
    @ManyToOne(fetch = LAZY) @JoinColumn(name = "org_id")  private Organization organization;

    @Enumerated(STRING) private OrgRole role;            // ← the reason you need this
    private Instant joinedAt;
    @ManyToOne(fetch = LAZY) private User invitedBy;
}
```

**Two `@ManyToOne`s instead of a `@ManyToMany`.** More explicit, extensible, queryable, and it avoids `@ManyToMany`'s collection-update quirks (Hibernate often deletes *all* rows and re-inserts them when the collection changes).

**Say in an interview:** *"I avoid `@ManyToMany` and model the join table as an entity, because real many-to-many relationships almost always acquire attributes — a role, a timestamp, who created it — and retrofitting that is a migration."* **Vigil does exactly this** for user↔organization memberships.

**If you must use `@ManyToMany`:** use `Set`, not `List` (a `List` with `@ManyToMany` causes delete-all-and-reinsert behaviour), and put `cascade` on neither side except `PERSIST`/`MERGE`.

---

## 4. Cascade and `orphanRemoval`

```java
CascadeType.PERSIST    // save parent → save children
CascadeType.MERGE      // merge parent → merge children
CascadeType.REMOVE     // delete parent → delete children  ⚠️ DANGEROUS
CascadeType.REFRESH
CascadeType.DETACH
CascadeType.ALL        // all of the above  ⚠️⚠️
```

**`CascadeType.ALL` on a `@ManyToOne` is a genuine catastrophe:**

```java
@Entity
public class Monitor {
    @ManyToOne(cascade = CascadeType.ALL)      // ❌❌❌
    private Organization organization;
}
```
```java
monitorRepository.delete(monitor);
// → cascades REMOVE to Organization
// → which cascades to ALL ITS monitors, users, incidents, status pages...
// You deleted a customer's entire account by deleting one monitor.
```

**The rule: cascade flows from parent to child, following composition, never from child to parent.**

Ask: *"if the parent is deleted, do the children have any meaning?"*
- `Monitor` → `CheckResult`: no meaning without the monitor → **cascade is correct**
- `Monitor` → `Organization`: the org obviously outlives the monitor → **never cascade**

**`orphanRemoval = true`** deletes a child removed from the parent's collection:

```java
@OneToMany(mappedBy = "monitor", cascade = ALL, orphanRemoval = true)
private List<CheckResult> results;
```
```java
monitor.getResults().remove(result);      // → DELETE FROM check_result WHERE id = ?
```

`orphanRemoval` implies `REMOVE` but is stronger: `REMOVE` cascades only on parent deletion, while `orphanRemoval` also fires when a child is *disassociated*. Use it only for true composition where a child cannot exist independently.

**Performance warning:** cascading a delete over 1,000,000 `check_result` rows means Hibernate loads them all into memory and issues 1,000,000 DELETEs. For high-volume child tables, use `ON DELETE CASCADE` in the database schema plus a bulk `@Modifying` delete query — and skip the JPA cascade. **That's exactly what Vigil does for `check_result`**, and explaining why is a nice performance-awareness signal.

---

## 5. `equals` and `hashCode` — the most misunderstood topic in JPA

```java
// ❌ WRONG — Lombok's @Data / @EqualsAndHashCode on an entity
@Entity @Data
public class Monitor { ... }
```

**Why this is wrong, in three separate ways:**

1. **It includes every field,** so `equals` touches lazy associations → triggers queries, or `LazyInitializationException`, during a `Set.add()`.
2. **`toString()` includes associations** → infinite recursion on bidirectional relationships → `StackOverflowError`, often inside a log statement.
3. **Field values change**, so `hashCode` changes, so an entity in a `HashSet` becomes unfindable (Lesson 02).

**The core difficulty:** an entity's identity changes over its life. A new entity has `id == null`. After persist, it has an ID. If `hashCode` is based on the ID, it changes at persist time — breaking any `HashSet` the entity was already in.

### The correct implementation

```java
@Entity
public class Monitor {
    @Id @GeneratedValue private Long id;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        // ★ use Hibernate.getClass to handle proxies: a proxy's class is a SUBCLASS,
        //   so `getClass() != o.getClass()` would wrongly return false for proxy vs entity
        if (o == null || Hibernate.getClass(this) != Hibernate.getClass(o)) return false;
        Monitor other = (Monitor) o;
        return id != null && id.equals(other.getId());     // ★ null id → NEVER equal
    }

    @Override
    public int hashCode() {
        return getClass().hashCode();       // ★ CONSTANT — stable across the ID transition
    }
}
```

**Justifying each line — this is the part that impresses:**

- **`Hibernate.getClass()`** unwraps proxies. `monitor.equals(lazyProxyOfSameRow)` must be true, and `getClass()` on a proxy returns `Monitor$HibernateProxy$xY`, not `Monitor`.
- **`id != null && ...`** means two unsaved entities are never equal (they're distinct objects) and an unsaved entity never equals a saved one. That's the correct semantic.
- **A constant `hashCode`** looks wrong but is deliberately correct: it's stable across the transient→persistent transition, so an entity added to a `HashSet` before persisting remains findable after. The cost is that all entities of a type land in one hash bucket — irrelevant, because you should never hold thousands of entities in a `HashSet` anyway.

### The better answer: use a business/natural key

```java
@Entity
public class Monitor {
    @Column(nullable = false, unique = true, updatable = false)
    private UUID publicId = UUID.randomUUID();      // assigned at CONSTRUCTION, never changes

    @Override public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Monitor other)) return false;
        return publicId.equals(other.publicId);
    }
    @Override public int hashCode() { return publicId.hashCode(); }
}
```

**An immutable, application-assigned key sidesteps the whole problem.** Identity is defined at construction and never changes, so `equals`/`hashCode` are stable and meaningful from birth. **This is why assigned UUIDs are attractive despite the extra SELECT on `save()`** — and it connects back to Lesson 20's UUID v7 discussion. Being able to link those two decisions is a genuinely senior observation.

### Lombok on entities — the safe subset

```java
@Entity
@Getter @Setter                                   // ✅
@NoArgsConstructor(access = AccessLevel.PROTECTED) // ✅
@ToString(onlyExplicitlyIncluded = true)          // ✅ opt-in fields only
public class Monitor {
    @ToString.Include private Long id;
    @ToString.Include private String name;
    @ManyToOne(fetch = LAZY) private Organization organization;   // excluded from toString ✅
}
```

**Never on an entity: `@Data`, `@EqualsAndHashCode`, `@ToString` (unqualified), `@AllArgsConstructor`, `@Builder` (it can bypass required initialization of collections).**

---

## 6. Inheritance strategies

```java
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)      // default
@Inheritance(strategy = InheritanceType.JOINED)
@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
@MappedSuperclass                                          // not an entity at all
```

| Strategy | Schema | Pros | Cons |
|---|---|---|---|
| **SINGLE_TABLE** | One table + discriminator column | Fastest — no joins; polymorphic queries are trivial | **Subclass columns must be nullable** — you lose NOT NULL constraints; wide sparse table |
| **JOINED** | Parent table + one per subclass | Normalized, constraints preserved | A join per subclass access; slower |
| **TABLE_PER_CLASS** | One full table per concrete class | No joins for single-type queries | Polymorphic queries need `UNION ALL`; no shared sequence; **avoid** |
| **@MappedSuperclass** | No table for the parent | Shares mappings only | Not polymorphically queryable |

```java
// ✅ The one you'll actually use most: @MappedSuperclass for shared audit fields
@MappedSuperclass
@Getter @Setter
public abstract class BaseEntity {
    @CreatedDate  @Column(updatable = false) private Instant createdAt;
    @LastModifiedDate                        private Instant updatedAt;
    @CreatedBy    @Column(updatable = false) private String createdBy;
    @LastModifiedBy                          private String updatedBy;
    @Version                                 private long version;
}

@Entity
public class Monitor extends BaseEntity { ... }
```

**Practical guidance:** `SINGLE_TABLE` for small, similar hierarchies (Vigil's check configs); `JOINED` when subclasses differ substantially and you need real constraints; `@MappedSuperclass` for shared *fields* with no polymorphism (audit columns — by far the most common use); **never `TABLE_PER_CLASS`**.

**And the honest senior take:** *"Entity inheritance is often more trouble than it's worth. I'd frequently prefer one table with a type discriminator and a JSON column for type-specific data, or separate unrelated entities."* Vigil's `Monitor` uses a `type` enum plus a `jsonb` config column instead of an inheritance hierarchy — simpler to query, migrate, and reason about.

---

## 7. Auditing

```java
@Configuration
@EnableJpaAuditing(auditorAwareRef = "auditorProvider")
public class JpaAuditingConfig {
    @Bean
    AuditorAware<String> auditorProvider() {
        return () -> Optional.ofNullable(SecurityContextHolder.getContext().getAuthentication())
                .filter(Authentication::isAuthenticated)
                .map(Authentication::getName)
                .or(() -> Optional.of("system"));
    }
}
```
```java
@EntityListeners(AuditingEntityListener.class)      // required on the entity or @MappedSuperclass
@MappedSuperclass
public abstract class BaseEntity {
    @CreatedDate  @Column(updatable = false) private Instant createdAt;
    @LastModifiedDate                        private Instant updatedAt;
    @CreatedBy    @Column(updatable = false) private String createdBy;
    @LastModifiedBy                          private String updatedBy;
}
```

Four audit columns on every entity, for free. **Note the `or(() -> "system")` fallback** — scheduled jobs and message consumers have no `Authentication`, and without a fallback `@CreatedBy` is null for exactly the writes you most want attributed. Small detail, real consequence.

**Distinguish this from *audit logging*** (Lesson 10): these columns record *who last touched a row*; an audit log records *every action, including reads and failures*. Compliance usually needs both, and knowing the difference is worth mentioning.

**For full history, Hibernate Envers** (`@Audited`) writes a revision row per change into `_aud` tables. Powerful; adds significant write volume and schema. Mention it as the option for "we need to see what a monitor looked like last Tuesday."

---

## 8. Build & break — Vigil's full data model

Model the whole domain now. This is real project work.

```java
@Entity  // organization: id, name, slug (unique), plan, createdAt
@Entity  // app_user: id, email (unique), passwordHash, name, createdAt
@Entity  // membership: user, organization, role, joinedAt, invitedBy  ← join entity, NOT @ManyToMany
@Entity  // monitor: org, name, url, type, status, intervalSeconds, config (jsonb), @Version
@Entity  // check_result: monitor, status, responseTimeMs, statusCode, error, checkedAt
@Entity  // incident: monitor, status, startedAt, acknowledgedAt, resolvedAt, @Version
@Entity  // notification_channel: org, type, config (jsonb), enabled
@Entity  // status_page: org, slug (unique), title, monitors (join entity)
```

**Specific requirements — each exercises a trap from this lesson:**

1. **`BaseEntity` `@MappedSuperclass`** with all four audit fields plus `@Version`. Every entity extends it.
2. **`Membership` as a join entity** with a `role` enum — prove to yourself why `@ManyToMany` wouldn't have worked.
3. **`Monitor.config` as `@JdbcTypeCode(SqlTypes.JSON)`** on a `jsonb` column holding `Map<String,String>`. Verify Postgres stores real JSON (`SELECT config->>'expectedHeader' FROM monitor`).
4. **Every to-one association `LAZY`**, with `@JoinColumn` and a named `@ForeignKey`.
5. **Every enum `@Enumerated(EnumType.STRING)`.**
6. **Correct `equals`/`hashCode`** on every entity — use the `publicId` approach on at least one and the ID-based approach on another, so you've written both.
7. **`Monitor` → `CheckResult`: no JPA cascade.** Use `ON DELETE CASCADE` in the schema plus a bulk delete query. Write a comment explaining why (volume).
8. **`Monitor` → `Incident`: bidirectional with helper methods** `addIncident`/`removeIncident`.

### Break these on purpose — every one is a real bug you should feel

**a) The bidirectional trap.** Add a `CheckResult` to `monitor.getResults()` without setting `result.setMonitor(monitor)`. Watch the FK be null or the insert fail. Then add the helper method and watch it work.

**b) The ordinal disaster.** Set `@Enumerated(EnumType.ORDINAL)`, save rows with each status, then insert a new constant at the *front* of the enum and re-read. **Watch your data mean something different.** This is a five-minute experiment with permanent impact.

**c) `CascadeType.ALL` on `@ManyToOne`.** Put it on `Monitor.organization` in a scratch branch, delete one monitor, and watch the cascade take the organization (and everything else) with it. **Do this against throwaway data.** Then remove it forever.

**d) Lombok `@Data` on an entity.** Add it to `Monitor`, then `log.info("{}", monitor)` with a loaded bidirectional relationship. Watch the `StackOverflowError`. Then put two unsaved entities in a `HashSet`, save them, and try `set.contains(entity)`. Watch it fail.

**e) The lazy `@OneToOne`.** Make a nullable `@OneToOne` on the inverse side, mark it `LAZY`, and observe in the SQL log that **Hibernate queries it eagerly anyway.**

**f) `orphanRemoval` without clearing the owning side.** Remove a child from the collection but don't null its parent reference, and observe the delete doesn't happen as expected.

**g) Schema validation.** Set `ddl-auto: validate` with a mismatched column name and read the error. This is the setting that catches entity/schema drift before production does.

### Explain out loud (60s)

1. Why `EnumType.ORDINAL` is a data-corruption bug, with the version-1-to-version-2 example.
2. The owning side: which side holds the FK, and what happens when you update the other one.
3. The cascade question in one sentence, and the `@ManyToOne` catastrophe.
4. Why `@Data` on an entity breaks in three separate ways.
5. Why a constant `hashCode` on an entity is deliberately correct.

---

## 9. Production rules

| Do / Never | Why |
|---|---|
| **Always `@Enumerated(EnumType.STRING)`** | `ORDINAL` is the default and stores positions, so inserting a constant silently reinterprets every historical row with no error and no migration |
| **Put `@Version` on every entity two users can edit at once** | One column turns "the second admin silently overwrote the first" into a 409 the client can retry |
| **Always give `@OneToMany` a `mappedBy`** | Without it Hibernate assumes this side owns the relationship and quietly creates a join table you never asked for |
| **Write `addChild`/`removeChild` helpers that set both sides** | Only the owning side's state is written, so a one-sided update leaves the in-memory graph and the database disagreeing |
| **Cascade from parent to child only, never child to parent** | `CascadeType.ALL` on a `@ManyToOne` means deleting one monitor can delete the organization and everything under it |
| **Ask "do the children mean anything without the parent?" before cascading** | `CheckResult` without its monitor is meaningless, so cascade is right; an `Organization` outlives its monitors, so it never is |
| **Use database `ON DELETE CASCADE` plus a bulk delete for high-volume children** | A JPA cascade loads all million rows into memory and issues a million DELETEs |
| **Never put Lombok `@Data` or `@EqualsAndHashCode` on an entity** | It touches lazy associations during `Set.add()`, recurses infinitely in `toString()` on bidirectional links, and changes `hashCode` as fields change |
| **Base `equals` on the ID with `Hibernate.getClass()`, and return a constant `hashCode`** | A proxy's class is a subclass, so `getClass()` comparison fails; a constant hash stays stable across the transient→persistent transition |
| **Prefer a stable business key when the entity has one** | It gives you real equality semantics before persist, instead of "no two unsaved entities are ever equal" |
| **Never make an entity class or its getters `final`, and keep the no-arg constructor `protected`** | `final` silently disables proxying and therefore lazy loading; `protected` tells your own code not to use the constructor Hibernate needs |
| **Never use a record as an entity** | No no-arg constructor, `final` class, immutable fields — all three are incompatible with proxying and dirty checking |

---

## 10. Interview traps

**Q1. Why can't a record be a JPA entity?**
Records are `final`, have no no-arg constructor, and have immutable fields. Hibernate needs a non-final class (to generate proxy subclasses for lazy loading), a no-arg constructor (to instantiate reflectively before populating), and mutable fields (for dirty checking). Records are ideal for DTOs and projections instead.

**Q2. Why must you use `EnumType.STRING`?**
`ORDINAL` (the default) stores the enum's index. Adding or reordering constants silently reinterprets every existing row — no error, no migration, complete data corruption. `STRING` stores the name; note it still breaks on *rename*, so long-lived schemas sometimes use an `AttributeConverter` to a stable code.

**Q3. Explain the bidirectional relationship trap.**
`mappedBy` marks the collection as the inverse side; Hibernate persists only changes to the **owning** side (the one holding the FK). Adding to `parent.getChildren()` without setting `child.setParent(parent)` results in a null FK — the object graph looks right, the database doesn't. Always write `addX`/`removeX` helpers that update both sides.

**Q4. Which side owns a relationship?**
The side with the foreign key column — always the `@ManyToOne` side, or the `@OneToOne` side with `@JoinColumn`. The side with `mappedBy` is the inverse and is not persisted.

**Q5. Why is `CascadeType.ALL` on `@ManyToOne` dangerous?**
Deleting the child cascades REMOVE to the parent, which cascades to all *its* children — deleting a monitor can delete the whole organization. Cascades follow composition, parent→child only. Test: "if the parent is deleted, do the children still mean anything?"

**Q6. How do you implement `equals`/`hashCode` on an entity?**
Use `Hibernate.getClass()` (not `getClass()`) so proxies compare correctly; compare only the ID and return false when the ID is null (so unsaved entities are never equal); and return a **constant** `hashCode` so it's stable across the transient→persistent transition. Better still: use an application-assigned immutable business key (e.g. a UUID set at construction), which makes identity stable from birth.

**Q7. Why is Lombok `@Data` wrong on an entity?**
It generates `equals`/`hashCode` over all fields — touching lazy associations (queries or `LazyInitializationException` inside `Set.add`) and producing an unstable hash — and a `toString` that recurses infinitely across bidirectional relationships. Safe subset: `@Getter`, `@Setter`, `@NoArgsConstructor(access = PROTECTED)`, and `@ToString(onlyExplicitlyIncluded = true)`.

**Q8. Why avoid `@ManyToMany`?**
The join table can hold only two FKs, but real relationships acquire attributes (role, joined-at, invited-by), and adding one is a migration. Modelling the join table as an entity with two `@ManyToOne`s is explicit, extensible, and avoids Hibernate's delete-all-and-reinsert collection behaviour.

**Q9. Which inheritance strategy would you choose?**
`SINGLE_TABLE` for small similar hierarchies (fast, no joins, but subclass columns must be nullable). `JOINED` when subclasses differ a lot and you need real NOT NULL/FK constraints. `@MappedSuperclass` for shared fields without polymorphism — the most common real use, for audit columns. Avoid `TABLE_PER_CLASS`. Often the best answer is no inheritance at all: a type discriminator plus a JSON column.

**Q10. Why isn't my lazy `@OneToOne` lazy?**
A nullable `@OneToOne` on the inverse (`mappedBy`) side must be queried to know whether to return a proxy or null, so Hibernate loads it eagerly regardless. Fixes: `optional = false`, put the FK on this side, or use bytecode enhancement. Often better: model it as `@ManyToOne` with a unique constraint.

**Q11. How do you get created/updated audit fields automatically?**
`@EnableJpaAuditing` with an `AuditorAware` bean, `@EntityListeners(AuditingEntityListener.class)`, and `@CreatedDate`/`@LastModifiedDate`/`@CreatedBy`/`@LastModifiedBy` on a `@MappedSuperclass`. Include a fallback auditor (e.g. `"system"`) so scheduled jobs and message consumers don't write nulls.

**Q12. Why not cascade the delete of a million child rows?**
JPA cascade loads every child into the persistence context and issues one DELETE each — enormous memory and time cost. Use a database-level `ON DELETE CASCADE` plus a bulk `@Modifying` delete, accepting that the persistence context won't know about it (so clear it).

---

## 11. Retention check

1. Name JPA's three structural requirements on an entity class and what each is for.
2. Explain the bidirectional trap and the exact fix, in 30 seconds.
3. Why is a constant `hashCode` the *correct* choice for an ID-based entity `equals`?
4. What does `EnumType.ORDINAL` do when you insert a new constant, and why is that worse than an exception?
5. Give the test for whether a cascade is appropriate.
6. Why model a join table as an entity instead of using `@ManyToMany`?

---

**Next:** [Lesson 22 — Spring Data JPA repositories](22-spring-data-jpa-repositories.md) — querying efficiently: derived queries, JPQL, projections, Specifications, and when each is the right tool.

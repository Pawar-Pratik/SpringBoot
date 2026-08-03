# Lesson 16 — API design: DTOs, validation & error handling

> **Why this lesson exists:** This is the lesson with the highest ratio of "how you're judged" to "how hard it is." An interviewer looking at your code notices immediately whether you return entities, whether your errors are consistent, and whether your validation is real. All three are cheap to get right and expensive to retrofit.

**Time:** ~75 minutes · **Prereq:** Lesson 15

---

## 1. Why DTOs — the argument in full

Returning JPA entities from controllers is the single most common design flaw in Spring codebases. Here's why, with the reasons ordered by how badly each will hurt you.

```java
// ❌ The mistake
@GetMapping("/{id}")
public Monitor get(@PathVariable UUID id) {
    return monitorRepository.findById(id).orElseThrow();
}
```

**1. Your database schema becomes your public API contract.** Rename a column, and every client breaks. Add a field, and you've published it forever. **You've coupled your storage decisions to your consumers' code**, and you can no longer refactor the schema without a versioned API migration. This is the reason that costs the most over time.

**2. Accidental data exposure.** Every field on the entity is serialized — including `passwordHash`, `internalNotes`, `apiKeySecret`, and `deletedAt`. Adding a sensitive field to the entity silently publishes it. Security incidents have started exactly here.

**3. Lazy-loading explosions.** Jackson calls every getter. On `monitor.getCheckResults()`:
- Inside a transaction → **fires a query for a million rows** and tries to serialize them.
- Outside a transaction → `LazyInitializationException` → 500.
- Bidirectional relationship → **infinite recursion** → `StackOverflowError`.

That last one is the classic: `Monitor → Organization → List<Monitor> → Organization → ...`. People "fix" it with `@JsonIgnore` on the entity, which means **JSON concerns are now polluting your persistence model** — two problems, not one.

**4. No shaping.** A list endpoint and a detail endpoint want different fields. One entity can't be both without `@JsonView` gymnastics.

**5. Input is worse than output.** `@RequestBody Monitor` lets a client set `id`, `createdAt`, `organizationId`, or `status` — a **mass assignment vulnerability**. A user could reassign their monitor to another organization. This is a real, exploitable class of bug.

### The correct shape

```java
// INPUT — only what a client may set
public record CreateMonitorRequest(
        @NotBlank @Size(max = 100) String name,
        @NotBlank @Pattern(regexp = "^https?://.+") String url,
        @Min(30) @Max(86400) int intervalSeconds,
        @NotNull MonitorType type
) {}

// OUTPUT (detail) — only what a client may see
public record MonitorResponse(
        UUID id, String name, String url, MonitorType type, MonitorStatus status,
        int intervalSeconds, Instant lastCheckedAt, Integer lastResponseTimeMs,
        BigDecimal uptime30d, Instant createdAt
) {}

// OUTPUT (list) — lighter, for tables
public record MonitorSummaryResponse(
        UUID id, String name, MonitorStatus status, BigDecimal uptime30d
) {}
```

**"But that's boilerplate!"** Yes — and it's the cheapest insurance in your codebase. The pushback is understandable; the answer is that a mapper takes 10 minutes and a leaked `passwordHash` takes a week and a disclosure email.

### Mapping: by hand or MapStruct

**By hand** (fine for small projects, zero magic):
```java
@Component
class MonitorMapper {
    MonitorResponse toResponse(Monitor m) {
        return new MonitorResponse(m.getId(), m.getName(), m.getUrl(), m.getType(),
                m.getStatus(), m.getIntervalSeconds(), m.getLastCheckedAt(),
                m.getLastResponseTimeMs(), m.getUptime30d(), m.getCreatedAt());
    }
}
```

**MapStruct** (the industry standard for anything larger):
```java
@Mapper(componentModel = "spring", unmappedTargetPolicy = ReportingPolicy.ERROR)
public interface MonitorMapper {
    MonitorResponse toResponse(Monitor monitor);
    MonitorSummaryResponse toSummary(Monitor monitor);

    @Mapping(target = "id", ignore = true)
    @Mapping(target = "status", constant = "PENDING")
    Monitor toEntity(CreateMonitorRequest request);
}
```

**MapStruct generates the implementation at compile time** — no reflection, no runtime cost, and mapping mistakes are **compile errors**. `unmappedTargetPolicy = ERROR` is the setting that earns its keep: add a field to the DTO and forget to map it, and the build fails instead of silently returning null.

> **Why MapStruct over ModelMapper/Dozer:** those map reflectively at runtime, so a mismatch is a silent null or a runtime exception, and they cost CPU on every call. MapStruct is a compile-time annotation processor generating plain Java you can read in `target/generated-sources`. **Read that generated file once** — it demystifies the whole thing. If asked "MapStruct or ModelMapper?", the answer is MapStruct, and the reason is compile-time safety.

---

## 2. Bean Validation

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

**This is required.** It was removed from `starter-web` in Boot 2.3, and its absence means `@Valid` silently does nothing (Lesson 11).

### The constraints

```java
public record CreateMonitorRequest(
        @NotNull                     UUID organizationId,   // not null (empty string ok)
        @NotEmpty                    List<String> tags,     // not null, size > 0
        @NotBlank @Size(min = 1, max = 100) String name,    // not null, trimmed length > 0
        @Email                       String contactEmail,
        @Pattern(regexp = "^https?://.+", message = "must be a valid http(s) URL") String url,
        @Min(30) @Max(86400)         int intervalSeconds,
        @Positive                    int retries,
        @DecimalMin("0.0") @DecimalMax("100.0") BigDecimal slaTarget,
        @Past                        Instant createdBefore,
        @Future                      Instant scheduledFor,
        @Valid                       AlertConfig alertConfig      // ★ CASCADES into the nested object
) {}
```

**`@NotNull` vs `@NotEmpty` vs `@NotBlank`** — a guaranteed interview question:

| | `null` | `""` | `"  "` | `[]` |
|---|---|---|---|---|
| `@NotNull` | ❌ fail | ✅ pass | ✅ pass | ✅ pass |
| `@NotEmpty` | ❌ | ❌ | ✅ pass | ❌ |
| `@NotBlank` | ❌ | ❌ | ❌ | n/a (String only) |

**For a user-supplied string, you almost always want `@NotBlank`** — `@NotNull` happily accepts `"   "`.

**`@Valid` on a nested field is required for cascading.** Without it, `AlertConfig`'s constraints are never checked. This is a silent, common bug.

### Triggering validation

```java
// 1. Request bodies — @Valid on the parameter
@PostMapping
public MonitorResponse create(@Valid @RequestBody CreateMonitorRequest req) { }

// 2. Path/query params — @Validated on the CLASS is required
@RestController
@Validated
public class MonitorController {
    @GetMapping
    public List<?> list(@RequestParam @Min(1) @Max(100) int size) { }
}

// 3. Service method params — @Validated on the class (proxy-based, so all Lesson 09 rules apply)
@Service
@Validated
public class MonitorService {
    public void create(@Valid CreateMonitorRequest req) { }
}
```

**`@Valid` vs `@Validated`** — the distinction people get wrong:

| | `@Valid` | `@Validated` |
|---|---|---|
| From | Jakarta Bean Validation | Spring |
| Place on | parameters, fields (cascading) | **classes** (enables method validation), parameters (with groups) |
| Supports validation **groups** | ❌ | ✅ |
| Failure exception | `MethodArgumentNotValidException` | `ConstraintViolationException` |

**Those two exceptions produce different error shapes**, so your `@ControllerAdvice` must handle both — a detail almost everyone misses on the first pass, and a great thing to demonstrate.

### Validation groups

```java
public interface OnCreate {}
public interface OnUpdate {}

public record MonitorRequest(
        @Null(groups = OnCreate.class) @NotNull(groups = OnUpdate.class) UUID id,
        @NotBlank(groups = {OnCreate.class, OnUpdate.class}) String name
) {}

@PostMapping public MonitorResponse create(@Validated(OnCreate.class) @RequestBody MonitorRequest r) {}
@PutMapping  public MonitorResponse update(@Validated(OnUpdate.class) @RequestBody MonitorRequest r) {}
```

Useful, but **separate DTOs are usually clearer** than groups. Groups reuse one class at the cost of making its rules conditional and harder to read. Know they exist; reach for them only when the DTOs are genuinely near-identical.

### Custom validators (Vigil needs this one)

```java
@Target({FIELD, PARAMETER}) @Retention(RUNTIME)
@Constraint(validatedBy = SafeUrlValidator.class)
public @interface SafeUrl {
    String message() default "URL is not permitted";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

```java
public class SafeUrlValidator implements ConstraintValidator<SafeUrl, String> {

    private static final Set<String> ALLOWED_SCHEMES = Set.of("http", "https");

    @Override
    public boolean isValid(String value, ConstraintValidatorContext ctx) {
        if (value == null) return true;                  // ★ null is @NotNull's job, not ours
        try {
            URI uri = URI.create(value);
            if (!ALLOWED_SCHEMES.contains(uri.getScheme())) return false;

            // SSRF defence: a monitoring tool takes URLs from users and fetches them.
            // Without this, a user points a monitor at http://169.254.169.254/ (cloud metadata)
            // or http://localhost:8080/actuator/env and reads the RESPONSE BODY in your UI.
            InetAddress addr = InetAddress.getByName(uri.getHost());
            if (addr.isLoopbackAddress() || addr.isSiteLocalAddress()
                    || addr.isLinkLocalAddress() || addr.isAnyLocalAddress()) {
                return false;
            }
            return true;
        } catch (Exception e) {
            return false;
        }
    }
}
```

**Two lessons in there, and both matter:**

1. **A validator should return `true` for null.** Composability: let `@NotNull` own nullness so `@SafeUrl` works on optional fields too.
2. **This SSRF check is a genuine, exploitable vulnerability class for Vigil specifically.** Any app that fetches a user-supplied URL is an SSRF risk — cloud metadata endpoints (`169.254.169.254`) hand out IAM credentials. **Being able to explain why your monitoring tool needs SSRF protection is one of the strongest things you can say about this project in an interview.** (Note: validation-time DNS resolution isn't a complete defence — DNS rebinding can change the answer between validation and fetch — so the real fix also checks at connection time. Say that too; it shows you understand the limits of your own control.)

**Cross-field validation** goes on the class:
```java
@Target(TYPE) @Retention(RUNTIME)
@Constraint(validatedBy = IntervalBelowTimeoutValidator.class)
public @interface IntervalAboveTimeout { String message() default "interval must exceed timeout"; }
```

---

## 3. Error handling — the thing that makes an API feel professional

### RFC 7807 / RFC 9457 Problem Details — use the standard

Spring 6 has built-in support. **Use it** rather than inventing a shape.

```json
{
  "type": "https://vigil.io/problems/validation-error",
  "title": "Validation failed",
  "status": 400,
  "detail": "2 fields are invalid",
  "instance": "/api/v1/monitors",
  "correlationId": "8f3a2b1c-4d5e-6f7a-8b9c-0d1e2f3a4b5c",
  "timestamp": "2026-07-30T14:32:11Z",
  "errors": [
    { "field": "name", "message": "must not be blank", "rejectedValue": "" },
    { "field": "intervalSeconds", "message": "must be greater than or equal to 30", "rejectedValue": 10 }
  ]
}
```

**Why a standard shape matters:** clients can write one error handler instead of N. `type` is a stable, machine-readable identifier (unlike `title`, which you may reword). And `correlationId` in the body means **a user can paste your error into a support ticket and you can find the exact log line** — a small feature that changes how supportable your system is.

### The global handler

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    // ---- 400: request body validation (@Valid on @RequestBody) ----
    @ExceptionHandler(MethodArgumentNotValidException.class)
    ProblemDetail onValidation(MethodArgumentNotValidException ex) {
        var pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        pd.setType(URI.create("https://vigil.io/problems/validation-error"));
        pd.setTitle("Validation failed");
        pd.setDetail(ex.getBindingResult().getErrorCount() + " field(s) are invalid");
        pd.setProperty("errors", ex.getBindingResult().getFieldErrors().stream()
                .map(fe -> Map.of(
                        "field", fe.getField(),
                        "message", Objects.requireNonNullElse(fe.getDefaultMessage(), "invalid"),
                        "rejectedValue", String.valueOf(fe.getRejectedValue())))
                .toList());
        return decorate(pd);
    }

    // ---- 400: param/path validation (@Validated on the class) — DIFFERENT exception ----
    @ExceptionHandler(ConstraintViolationException.class)
    ProblemDetail onConstraintViolation(ConstraintViolationException ex) {
        var pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        pd.setTitle("Invalid request parameters");
        pd.setProperty("errors", ex.getConstraintViolations().stream()
                .map(v -> Map.of("field", v.getPropertyPath().toString(),
                                 "message", v.getMessage()))
                .toList());
        return decorate(pd);
    }

    // ---- 400: malformed JSON ----
    @ExceptionHandler(HttpMessageNotReadableException.class)
    ProblemDetail onUnreadable(HttpMessageNotReadableException ex) {
        var pd = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        pd.setTitle("Malformed request body");
        pd.setDetail("Request body is not valid JSON or has incompatible types");
        // ⚠️ Do NOT echo ex.getMessage() — it can contain internal class names and payload data.
        return decorate(pd);
    }

    // ---- 404 ----
    @ExceptionHandler(ResourceNotFoundException.class)
    ProblemDetail onNotFound(ResourceNotFoundException ex) {
        var pd = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
        pd.setTitle("Resource not found");
        return decorate(pd);
    }

    // ---- 409 ----
    @ExceptionHandler({DuplicateResourceException.class, OptimisticLockingFailureException.class})
    ProblemDetail onConflict(Exception ex) {
        var pd = ProblemDetail.forStatus(HttpStatus.CONFLICT);
        pd.setTitle("Conflict");
        pd.setDetail("The resource was modified or already exists. Re-read and retry.");
        return decorate(pd);
    }

    // ---- 403 ----
    @ExceptionHandler(AccessDeniedException.class)
    ProblemDetail onAccessDenied(AccessDeniedException ex) {
        var pd = ProblemDetail.forStatus(HttpStatus.FORBIDDEN);
        pd.setTitle("Access denied");
        // ⚠️ Deliberately vague — don't reveal WHICH permission is missing, or that the
        //    resource exists at all. Detail here is an information-disclosure channel.
        return decorate(pd);
    }

    // ---- 429 ----
    @ExceptionHandler(RateLimitExceededException.class)
    ResponseEntity<ProblemDetail> onRateLimit(RateLimitExceededException ex) {
        var pd = ProblemDetail.forStatus(HttpStatus.TOO_MANY_REQUESTS);
        pd.setTitle("Rate limit exceeded");
        return ResponseEntity.status(429)
                .header(HttpHeaders.RETRY_AFTER, String.valueOf(ex.retryAfterSeconds()))
                .body(decorate(pd));
    }

    // ---- 500: the catch-all ----
    @ExceptionHandler(Exception.class)
    ProblemDetail onUnexpected(Exception ex) {
        String errorId = UUID.randomUUID().toString();
        log.error("Unhandled exception errorId={}", errorId, ex);      // ★ full stack trace to LOGS
        var pd = ProblemDetail.forStatus(HttpStatus.INTERNAL_SERVER_ERROR);
        pd.setTitle("Internal server error");
        pd.setDetail("An unexpected error occurred. Quote errorId when contacting support.");
        pd.setProperty("errorId", errorId);                            // ★ safe correlation to CLIENT
        return decorate(pd);
    }

    private ProblemDetail decorate(ProblemDetail pd) {
        pd.setProperty("timestamp", Instant.now());
        pd.setProperty("correlationId", MDC.get("correlationId"));
        return pd;
    }
}
```

### The four rules of error handling

**1. Never leak internals to clients.** No stack traces, no SQL, no class names, no framework messages. An attacker uses those to fingerprint your stack and find known CVEs. And `HttpMessageNotReadableException.getMessage()` genuinely contains your class names and fragments of the submitted payload.

```yaml
server:
  error:
    include-stacktrace: never
    include-message: never          # 'always' is fine in dev only
    include-binding-errors: never
```

**2. Always log the full detail server-side, with an ID both sides can see.** The `errorId` pattern above is the reconciliation: the client gets a harmless UUID, your logs get everything, and support can join them. **This one pattern makes an API feel professionally operated.**

**3. Be deliberately vague on authorization failures.** Detailed 403s ("you lack MONITOR_DELETE on org 42") are an enumeration oracle. Worse, distinguishing 404 from 403 tells an attacker *which resources exist* — for sensitive resources, returning 404 for "exists but you can't see it" is the right call. Be able to explain that trade-off.

**4. Handle both validation exception types.** `MethodArgumentNotValidException` (body) and `ConstraintViolationException` (params). Miss the second and query-param validation returns an ugly, inconsistent 500.

### `@ControllerAdvice` scoping and ordering

```java
@RestControllerAdvice(basePackages = "com.vigil.api.v2")     // only v2 controllers
@RestControllerAdvice(annotations = RestController.class)
@Order(Ordered.HIGHEST_PRECEDENCE)                           // when multiple advices apply
```

Spring picks the **most specific** handler for the thrown type, so a `@ExceptionHandler(Exception.class)` catch-all doesn't shadow your specific handlers. But if two advices handle the same type, `@Order` decides.

**Alternative: `ResponseEntityExceptionHandler`.** Extend it and you inherit handlers for ~15 standard Spring MVC exceptions already producing `ProblemDetail`, overriding only what you want. Less code; slightly less control. Worth knowing both exist.

### `ErrorResponseException` — the lightweight option

```java
throw ErrorResponseException.builder(HttpStatus.CONFLICT)
        .detail("Monitor name already exists in this organization")
        .build();
```
Or make your exceptions implement `ErrorResponse` so they carry their own `ProblemDetail` and need no handler at all. Elegant for domain exceptions with a fixed mapping.

---

## 4. API design decisions you'll be asked to defend

### Versioning

| Strategy | Example | Verdict |
|---|---|---|
| **URI path** | `/api/v1/monitors` | ✅ **Recommended.** Visible, cacheable, trivially routable at a gateway, easy to document |
| Header | `X-Api-Version: 2` | Purist-clean, but invisible in logs/browsers and awkward to cache |
| Accept header | `Accept: application/vnd.vigil.v2+json` | "Correct" per REST theory; painful in practice |
| Query param | `?version=2` | Muddles versioning with filtering |

**Say: URI path versioning, because it's explicit, debuggable, and routable — and note that the real work isn't the scheme, it's the deprecation policy.** Version only on *breaking* changes; additive changes don't need one. Publish a sunset timeline (`Sunset` and `Deprecation` headers exist for this), and run v1 and v2 in parallel during migration. **The deprecation-policy point is what separates a thoughtful answer from a trivia answer.**

### Pagination

```java
// Offset pagination — simple, and what Spring Data gives you
GET /api/v1/monitors?page=0&size=20&sort=createdAt,desc
```
```java
@GetMapping
public PagedResponse<MonitorSummaryResponse> list(
        @PageableDefault(size = 20, sort = "createdAt", direction = DESC) Pageable pageable) {
    return service.list(pageable);
}
```
```properties
spring.data.web.pageable.max-page-size=100     # ★ cap it, or a client requests size=1000000
```

**The offset problem, and why it matters for Vigil:** `OFFSET 1000000` makes the database scan and discard a million rows — it gets slower the deeper you page. And with concurrent inserts, rows shift between pages, so items are duplicated or skipped.

**Cursor (keyset) pagination** fixes both:
```
GET /api/v1/check-results?after=eyJ0cyI6MTcyMjM0NTY3OCwiaWQiOiJhYmMifQ&limit=50

WHERE (checked_at, id) < (:lastCheckedAt, :lastId)
ORDER BY checked_at DESC, id DESC
LIMIT 50
```
Constant time at any depth, stable under inserts. Costs: no random page access, no total count.

**Vigil uses offset pagination for monitors (small, needs page numbers) and cursor pagination for check results (millions of rows, append-only, infinite scroll).** Being able to explain *why each endpoint uses a different strategy* is exactly the kind of reasoning that stands out — it shows you pick based on access pattern rather than habit.

### Idempotency for POST

```java
@PostMapping
public MonitorResponse create(
        @RequestHeader(value = "Idempotency-Key", required = false) String idempotencyKey,
        @Valid @RequestBody CreateMonitorRequest req) {
    return service.create(req, idempotencyKey);
}
```
Store the key → response mapping (Redis, TTL 24h). A repeat returns the *original* response instead of creating a duplicate. Stripe's API is the canonical reference. Essential for payments and anything a mobile client retries. (Lesson 34.)

### Other decisions, briefly

- **Naming:** plural nouns (`/monitors`), kebab-case paths (`/status-pages`), no verbs (`/monitors/{id}/pause` is an acceptable pragmatic exception for actions that aren't CRUD — REST purism here costs more than it gains).
- **Filtering:** `?status=DOWN&tag=production`. For complex queries, RSQL/`Specification` (Lesson 22).
- **Sparse fieldsets:** `?fields=id,name,status` — nice, but only add it when a client actually needs it.
- **HATEOAS:** hypermedia links in responses. Spring HATEOAS exists; **most real APIs skip it** because clients don't consume links. Know what it is, know that skipping it is a defensible and normal choice.
- **Bulk operations:** `POST /monitors/bulk`. Decide and document whether it's all-or-nothing (transactional) or per-item (207 Multi-Status with individual results). Vigil uses per-item, because one bad row shouldn't reject an import of 500.

---

## 5. Build this — Vigil's API, properly

**1. Split your DTOs** into `CreateMonitorRequest`, `UpdateMonitorRequest`, `MonitorResponse`, `MonitorSummaryResponse`. Verify no entity ever crosses the controller boundary.

**2. Add MapStruct** with `unmappedTargetPolicy = ERROR`:
```xml
<dependency>
  <groupId>org.mapstruct</groupId><artifactId>mapstruct</artifactId><version>1.6.3</version>
</dependency>
```
```xml
<!-- in maven-compiler-plugin <annotationProcessorPaths> -->
<path><groupId>org.mapstruct</groupId><artifactId>mapstruct-processor</artifactId>
      <version>1.6.3</version></path>
<!-- ⚠️ If using Lombok, lombok-mapstruct-binding is also required, and ORDER MATTERS:
     lombok, then lombok-mapstruct-binding, then mapstruct-processor. -->
```
Build, then **open `target/generated-sources/annotations/.../MonitorMapperImpl.java` and read it.** Then add a field to the DTO without mapping it and watch the **build fail**. That's the value.

**3. Full validation.** Add constraints to every input DTO. Write the `@SafeUrl` validator from §2 including the SSRF check. Test it with `http://localhost:8080/actuator/env` and `http://169.254.169.254/` — both must be rejected.

**4. ⭐ The global exception handler.** Implement the full `GlobalExceptionHandler` from §3. Then verify each path with curl:
```bash
curl -i -X POST localhost:8080/api/v1/monitors -H 'Content-Type: application/json' -d '{"name":"","url":"nope","intervalSeconds":5}'
```
```bash
curl -i -X POST localhost:8080/api/v1/monitors -H 'Content-Type: application/json' -d '{invalid json'
```
```bash
curl -i localhost:8080/api/v1/monitors/00000000-0000-0000-0000-000000000000
```
```bash
curl -i "localhost:8080/api/v1/monitors?size=99999"
```
Confirm: **every** error has the same shape, a correlation ID, and no stack trace. That last check is the one to be strict about.

**5. Prove the two-exception problem.** Add `@Validated` to the controller class and `@Min(1)` to a `@RequestParam`. Send an invalid value. Without a `ConstraintViolationException` handler you get a 500 — see it, then fix it. **This is the detail most people ship broken.**

**6. Pagination both ways.** Offset for `/monitors` (with `max-page-size=100` — verify `size=999999` is capped). Then implement cursor pagination for `/check-results` with a base64-encoded cursor. Compare the generated SQL.

**7. Test the error contract.** A `@WebMvcTest` asserting the JSON shape of a validation failure. Errors are part of your API — regressions in them break clients just as hard as changes to success responses, and almost nobody tests them. (Module 8.)

---

## 6. Interview traps

**Q1. Why not return JPA entities from controllers?**
Five reasons: it makes your schema the public API contract (renames become breaking changes); it risks exposing sensitive fields automatically; Jackson triggers lazy loading → N+1, `LazyInitializationException`, or infinite recursion on bidirectional relations; one entity can't shape both list and detail responses; and as *input* it's a mass-assignment vulnerability letting clients set `id`, `status`, or `organizationId`.

**Q2. `@NotNull` vs `@NotEmpty` vs `@NotBlank`?**
`@NotNull` — only rejects null. `@NotEmpty` — rejects null and zero-length (String/Collection/Map/array). `@NotBlank` — String only; rejects null, empty, and whitespace-only. For user-supplied text, `@NotBlank`.

**Q3. `@Valid` vs `@Validated`?**
`@Valid` is Jakarta Bean Validation: on parameters, and on fields to cascade into nested objects. `@Validated` is Spring: on a **class** to enable method-parameter validation (proxy-based), and it supports validation **groups**. They throw different exceptions — `MethodArgumentNotValidException` vs `ConstraintViolationException` — so a global handler must cover both.

**Q4. Why won't validation on a nested object run?**
Missing `@Valid` on the nested field. Cascading is opt-in.

**Q5. How do you handle exceptions globally?**
`@RestControllerAdvice` with `@ExceptionHandler` methods returning `ProblemDetail` (RFC 7807/9457). Spring selects the most specific handler for the thrown type. Alternatively extend `ResponseEntityExceptionHandler` to inherit handling for standard MVC exceptions.

**Q6. What must never appear in an error response?**
Stack traces, SQL, framework/class names, internal messages, and specifics about *which* permission failed. Log all of it server-side with a generated `errorId`, and return only that ID to the client so support can correlate.

**Q7. Why did validation stop working after upgrading to Boot 2.3+?**
`spring-boot-starter-validation` was removed from `starter-web`, so Hibernate Validator was no longer on the classpath and `@Valid` became a no-op — requests that should have been rejected reached controllers with invalid data.

**Q8. How do you version an API?**
URI path (`/api/v1/...`) — visible, cacheable, gateway-routable. Version only on breaking changes; additive changes don't need one. The harder part is the deprecation policy: parallel versions during migration, `Deprecation`/`Sunset` headers, a published timeline, and usage metrics per version so you know when it's safe to remove.

**Q9. What's wrong with offset pagination, and what's the alternative?**
`OFFSET n` makes the DB scan and discard n rows, so deep pages degrade linearly; and concurrent inserts shift rows, duplicating or skipping items across pages. Cursor/keyset pagination (`WHERE (ts, id) < (:lastTs, :lastId) ORDER BY ts DESC, id DESC LIMIT n`) is constant-time and stable, at the cost of no random page access and no total count.

**Q10. Why MapStruct over ModelMapper?**
MapStruct generates plain Java at compile time — zero reflection, readable output, and unmapped fields become **compile errors** with `unmappedTargetPolicy=ERROR`. Reflective mappers fail silently or at runtime and cost CPU per call.

**Q11. Should you use HATEOAS?**
Know what it is (hypermedia links driving client navigation, REST maturity level 3), and that most real APIs skip it because clients hardcode URLs anyway. Choosing not to use it is normal and defensible; not knowing it exists is not.

**Q12. 404 or 403 for a resource the user isn't allowed to see?**
403 is more honest, but distinguishing them tells an attacker which resources exist. For sensitive resources, return 404 to avoid the enumeration oracle. It's a deliberate trade-off between clarity and information disclosure — state it as such.

---

## 7. Retention check

1. Give the five reasons against returning entities, and name the one that's a *security* issue.
2. `@NotBlank` vs `@NotEmpty` on a `String` field — which do you want and why?
3. Which two exceptions must a global handler cover for validation, and what triggers each?
4. What's the `errorId` pattern and what problem does it solve for both client and operator?
5. Why is SSRF protection specifically important for Vigil, and what's the limit of validation-time checking?
6. When would you use cursor pagination instead of offset, and what do you give up?

---

**Next:** [Lesson 17 — OpenAPI, CORS, filters & interceptors](17-openapi-cors-filters-interceptors.md) — documentation, browser rules, and choosing the right layer for cross-cutting HTTP concerns.

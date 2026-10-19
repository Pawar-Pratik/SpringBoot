# Lesson 15 — Controllers, mapping & serialization

> **Why this lesson exists:** This is the layer you'll touch every single day. It's also full of small details that produce baffling bugs — a `LocalDateTime` serialized as an array, a `@RequestParam` that silently accepts null, a 500 instead of a 400. Get this lesson right and your APIs behave exactly as you intend.

**Time:** ~70 minutes · **Prereq:** Lesson 14

---

## 1. The idea in one sentence

> **A controller's only job is to translate HTTP into a method call and a return value back into HTTP — so every detail in this lesson exists to make that translation explicit, and every bug in it is something accepted, dropped or serialized silently.**

A missing `@Valid`, a `required = false` bound to an `int`, an `Instant` rendered as an array: none of them throws where you'd look. Being deliberate about binding and serialization is what makes a controller behave the way you intended.

---

## 2. Controller basics

```java
@RestController                                  // = @Controller + @ResponseBody on every method
@RequestMapping("/api/v1/monitors")              // class-level path prefix
@RequiredArgsConstructor
@Validated                                       // enables validation on method params
public class MonitorController {

    private final MonitorService monitorService;

    @GetMapping
    public PagedResponse<MonitorResponse> list(@RequestParam(defaultValue = "0") int page,
                                              @RequestParam(defaultValue = "20") int size) {
        return monitorService.list(PageRequest.of(page, size));
    }

    @GetMapping("/{id}")
    public MonitorResponse get(@PathVariable UUID id) {
        return monitorService.get(id);            // throws → @ControllerAdvice → 404
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public MonitorResponse create(@Valid @RequestBody CreateMonitorRequest request) {
        return monitorService.create(request);
    }

    @PutMapping("/{id}")
    public MonitorResponse replace(@PathVariable UUID id,
                                   @Valid @RequestBody UpdateMonitorRequest request) {
        return monitorService.replace(id, request);
    }

    @PatchMapping("/{id}")
    public MonitorResponse patch(@PathVariable UUID id,
                                 @RequestBody JsonNode patch) {
        return monitorService.patch(id, patch);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable UUID id) {
        monitorService.delete(id);
    }
}
```

**`@RestController` vs `@Controller`:** `@RestController` adds `@ResponseBody` to every method, so return values are serialized into the response body. `@Controller` returns view names for a template engine. Use `@RestController` for APIs, `@Controller` for server-rendered HTML (Vigil's public status page uses `@Controller` + Thymeleaf).

**Note there's no `try/catch` and no `ResponseEntity` ceremony.** Exceptions propagate to a `@ControllerAdvice` (Lesson 16), and status codes come from `@ResponseStatus`. That's the clean style; use `ResponseEntity` only when you genuinely need dynamic status or headers.

### Mapping annotations

```java
@GetMapping     @PostMapping     @PutMapping     @PatchMapping     @DeleteMapping
// all shorthand for @RequestMapping(method = RequestMethod.X)

@RequestMapping(
    value    = "/search",
    method   = RequestMethod.GET,
    params   = "type",                          // only if ?type= is present
    headers  = "X-Api-Version=2",               // only with this header
    consumes = MediaType.APPLICATION_JSON_VALUE,// only for this Content-Type
    produces = MediaType.APPLICATION_JSON_VALUE // only if Accept matches; sets Content-Type
)
```

`consumes`/`produces` participate in mapping: a mismatch yields **415 Unsupported Media Type** or **406 Not Acceptable** rather than reaching your method. That's Spring doing content negotiation for you.

**Path patterns:**
```java
@GetMapping("/{id}")                    // exact segment
@GetMapping("/{orgId}/monitors/{id}")   // multiple
@GetMapping("/files/**")                // any remaining path
@GetMapping("/{id:[0-9]+}")             // regex-constrained
@GetMapping({"/list", "/all"})          // multiple paths
```

> **Boot 3 note:** Spring 6 switched from `AntPathMatcher` to `PathPatternParser` by default. It's faster and stricter — most notably, `**` is only allowed at the *end* of a pattern. If you migrate an old app with `/api/**/monitors`, it will now fail. Good migration trivia.

---

## 3. Getting data in

### `@PathVariable` — resource identity

```java
@GetMapping("/{id}")
public MonitorResponse get(@PathVariable UUID id) { }

@GetMapping("/{orgId}/monitors/{monitorId}")
public MonitorResponse get(@PathVariable UUID orgId,
                           @PathVariable("monitorId") UUID id) { }   // explicit when names differ
```

Path variables are **always required** (a missing one means a different URL, so it wouldn't map). Spring converts `String → UUID/Long/enum/LocalDate` automatically; a bad value gives 400 via `MethodArgumentTypeMismatchException`.

### `@RequestParam` — query strings and form fields

```java
@GetMapping
public List<MonitorResponse> search(
        @RequestParam String q,                                  // REQUIRED — 400 if missing
        @RequestParam(required = false) MonitorStatus status,     // null if absent
        @RequestParam(defaultValue = "20") int size,              // default (implies not required)
        @RequestParam Optional<String> tag,                       // Optional style
        @RequestParam(name = "created_after") Instant createdAfter,  // different external name
        @RequestParam List<String> tags,                          // ?tags=a&tags=b  OR  ?tags=a,b
        @RequestParam Map<String, String> allParams               // everything, untyped
) { }
```

**The trap:** `@RequestParam String q` is **required by default**. Omit it and you get 400 — which is usually what you want, but people are surprised. And `@RequestParam(required = false) int size` is a bug: `int` can't be null, so Spring throws when the param is absent. Use `Integer`, or supply a `defaultValue`.

**Better still, for anything more than ~3 params, bind an object:**

```java
public record MonitorSearchCriteria(
        String q,
        MonitorStatus status,
        List<String> tags,
        @Min(0) int page,
        @Min(1) @Max(100) int size
) {}

@GetMapping
public PagedResponse<MonitorResponse> search(@Valid MonitorSearchCriteria criteria) { }
```

No annotation needed — Spring binds query params to object properties by name. **This is much cleaner than eight `@RequestParam`s, and it's validatable.** A genuinely good thing to show in a code review.

### `@RequestBody` — JSON payloads

```java
@PostMapping
public MonitorResponse create(@Valid @RequestBody CreateMonitorRequest request) { }
```

Jackson deserializes via an `HttpMessageConverter` (Lesson 14). **`@Valid` triggers Bean Validation** — omit it and invalid data reaches your service. This is a very common bug: the annotations are on the DTO, but nobody added `@Valid`, so nothing validates. (Lesson 16.)

### The rest

```java
@RequestHeader("X-Api-Key") String apiKey
@RequestHeader(value = "X-Tenant", required = false) String tenant
@RequestHeader HttpHeaders headers                     // all of them

@CookieValue("session_id") String sessionId

@RequestPart("file") MultipartFile file                // multipart with a JSON part
@RequestPart("metadata") UploadMetadata metadata

// Injectable directly:
HttpServletRequest request                             // escape hatch — avoid in business code
HttpServletResponse response
Principal principal                                    // authenticated user
Authentication authentication                          // Spring Security's richer version
@AuthenticationPrincipal UserDetails user              // your user type — prefer this
Locale locale
UriComponentsBuilder uriBuilder                        // for building Location headers
```

**File upload:**
```java
@PostMapping(value = "/import", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
public ImportResult importMonitors(@RequestPart("file") MultipartFile file) {
    if (file.isEmpty()) throw new BadRequestException("file is empty");
    if (file.getSize() > 5 * 1024 * 1024) throw new BadRequestException("max 5MB");
    // ⚠️ NEVER trust file.getOriginalFilename() — path traversal ("../../etc/passwd").
    //    Generate your own name; validate the content type by sniffing, not by header.
    try (var in = file.getInputStream()) { return importer.importFrom(in); }
}
```
```properties
spring.servlet.multipart.max-file-size=5MB
spring.servlet.multipart.max-request-size=10MB
```
The filename warning is real — trusting client-supplied filenames is a classic vulnerability, and mentioning it unprompted is a strong security signal.

### Type conversion and custom converters

Spring converts request strings to types via `ConversionService`. Register your own:

```java
@Component
public class StringToMonitorIdConverter implements Converter<String, MonitorId> {
    @Override public MonitorId convert(String source) { return MonitorId.of(source); }
}
```

Now `@PathVariable MonitorId id` works, and you get a typed ID rather than a raw `UUID` — a nice way to prevent "passed the org ID where a monitor ID belonged" bugs at compile time.

For date formats:
```java
@RequestParam @DateTimeFormat(iso = ISO.DATE) LocalDate from            // 2026-07-30
@RequestParam @DateTimeFormat(pattern = "yyyy-MM-dd HH:mm") LocalDateTime at
```

---

## 4. Getting data out

### Return the object (preferred)

```java
@GetMapping("/{id}")
public MonitorResponse get(@PathVariable UUID id) { return service.get(id); }   // → 200 + JSON
```

Clean, testable, and the response type is visible in the signature — which matters for OpenAPI generation and for readers.

### `ResponseEntity` — when you need control

```java
@PostMapping
public ResponseEntity<MonitorResponse> create(@Valid @RequestBody CreateMonitorRequest req) {
    MonitorResponse created = service.create(req);
    URI location = ServletUriComponentsBuilder.fromCurrentRequest()
            .path("/{id}").buildAndExpand(created.id()).toUri();
    return ResponseEntity.created(location)              // 201 + Location header
            .header("X-Request-Cost", "1")
            .body(created);
}

// Conditional GET with an ETag — real bandwidth savings for polled endpoints
@GetMapping("/{id}")
public ResponseEntity<MonitorResponse> get(@PathVariable UUID id) {
    MonitorResponse m = service.get(id);
    return ResponseEntity.ok()
            .eTag("\"" + m.version() + "\"")
            .cacheControl(CacheControl.maxAge(Duration.ofSeconds(30)))
            .body(m);
}
```

**Use `ResponseEntity` when the status or headers are dynamic. Otherwise return the object** — `ResponseEntity` everywhere adds noise and obscures the actual response type.

**A note on `ResponseEntity<?>`:** avoid it. It defeats OpenAPI generation and tells readers nothing. If you need different response types, that's usually an exception (handled by `@ControllerAdvice`) rather than a union return type.

### `@ResponseStatus`

```java
@PostMapping @ResponseStatus(HttpStatus.CREATED)
public MonitorResponse create(...) { }

@DeleteMapping("/{id}") @ResponseStatus(HttpStatus.NO_CONTENT)
public void delete(...) { }
```

Also on exceptions:
```java
@ResponseStatus(HttpStatus.NOT_FOUND)
public class MonitorNotFoundException extends RuntimeException { }
```
Simple, but a `@ControllerAdvice` gives you a consistent error *body*, which matters more (Lesson 16).

---

## 5. JSON serialization with Jackson

Boot auto-configures an `ObjectMapper`. Understanding and controlling it is what separates a polished API from a leaky one.

### Field-level control

```java
public record MonitorResponse(
        UUID id,
        String name,

        @JsonProperty("check_url")                         // rename in JSON
        String url,

        @JsonFormat(shape = STRING, pattern = "yyyy-MM-dd'T'HH:mm:ss'Z'", timezone = "UTC")
        Instant lastCheckedAt,

        @JsonInclude(JsonInclude.Include.NON_NULL)         // omit when null
        String lastError,

        @JsonIgnore                                         // never serialize
        String internalNote,

        @JsonProperty(access = READ_ONLY)                   // output only; ignored on input
        MonitorStatus status
) {}
```

```java
// On an entity/DTO used for input:
@JsonProperty(access = WRITE_ONLY)  String password;        // accepted, never returned
@JsonIgnoreProperties(ignoreUnknown = true)                 // tolerate extra fields from clients
@JsonAnyGetter / @JsonAnySetter                             // dynamic key/value maps
@JsonUnwrapped                                              // flatten a nested object
@JsonView(Views.Admin.class)                                // conditional field sets
```

**`@JsonProperty(access = WRITE_ONLY)` on password fields is a habit worth building.** It's a one-annotation defence against accidentally returning a hash in a response body.

### Global configuration — the settings that matter

```yaml
spring:
  jackson:
    default-property-inclusion: non_null      # drop nulls globally
    property-naming-strategy: SNAKE_CASE      # last_checked_at instead of lastCheckedAt
    serialization:
      write-dates-as-timestamps: false        # ★ ISO-8601 strings, not epoch numbers
      indent-output: false                    # true only for local debugging
    deserialization:
      fail-on-unknown-properties: false       # tolerant of client-added fields
      fail-on-null-for-primitives: true       # catch missing required numbers
    time-zone: UTC
```

**`write-dates-as-timestamps: false` is the important one.** Without it (and without `jackson-datatype-jsr310` registered), `Instant` may serialize as `1753834800.000000000` or `LocalDateTime` as `[2026,7,30,14,30]` — an array. That array is one of the most confusing outputs a new Spring developer encounters. Boot registers the JSR-310 module and sets this correctly by default in modern versions, but **you must recognize the symptom**, because it appears the moment someone hand-builds an `ObjectMapper`.

**Which leads to the rule about customizing Jackson:**

```java
// ✅ CUSTOMIZE — keeps all of Boot's sensible defaults and modules
@Bean
Jackson2ObjectMapperBuilderCustomizer jacksonCustomizer() {
    return builder -> builder
            .serializationInclusion(JsonInclude.Include.NON_NULL)
            .featuresToDisable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
            .modules(new JavaTimeModule());
}

// ❌ REPLACE — @ConditionalOnMissingBean backs off, and you lose EVERY Boot default:
//    JSR-310 module, parameter names module, property inclusion, naming strategy...
@Bean
ObjectMapper objectMapper() { return new ObjectMapper(); }
```

**"Customize, don't replace" is the general Spring Boot principle** (Lesson 11), and Jackson is where it bites hardest. This is a genuinely good interview answer: *"I'd add a `Jackson2ObjectMapperBuilderCustomizer` rather than defining an `ObjectMapper` bean, because defining the bean makes Boot's auto-configuration back off and I'd silently lose the JavaTimeModule and the rest of the defaults."*

### Custom serializers

```java
public class DurationSecondsSerializer extends JsonSerializer<Duration> {
    @Override
    public void serialize(Duration value, JsonGenerator gen, SerializerProvider sp)
            throws IOException {
        gen.writeNumber(value.toSeconds());
    }
}
```
```java
@JsonSerialize(using = DurationSecondsSerializer.class)
Duration interval
```

Or register globally via a `SimpleModule` bean. Use these for domain value objects (`Money`, `MonitorId`, `Percentage`) so JSON shape is defined once rather than annotated everywhere.

### Polymorphic JSON (for Vigil's check outcomes)

```java
@JsonTypeInfo(use = NAME, property = "type")
@JsonSubTypes({
    @JsonSubTypes.Type(value = HttpCheckConfig.class, name = "HTTP"),
    @JsonSubTypes.Type(value = TcpCheckConfig.class,  name = "TCP"),
    @JsonSubTypes.Type(value = SslCheckConfig.class,  name = "SSL")
})
public sealed interface CheckConfig permits HttpCheckConfig, TcpCheckConfig, SslCheckConfig {}
```

```json
{ "type": "HTTP", "url": "https://api.example.com", "expectedStatus": 200 }
```

**Security warning worth knowing:** `@JsonTypeInfo(use = CLASS)` — putting a Java class name in the JSON — is a **deserialization gadget vulnerability**: an attacker names a dangerous class and Jackson instantiates it. This is the root of a long list of CVEs and the reason `enableDefaultTyping()` was deprecated. **Always use `use = NAME` with an explicit `@JsonSubTypes` allowlist.** Mentioning this unprompted is a strong security signal.

> **Boot 4 delta:** Spring Boot 4 moves to **Jackson 3** (`tools.jackson.*` package), which changes some defaults and makes `ObjectMapper` immutable-by-builder. Concepts are unchanged; imports and a few config keys move.

---

## 6. HTTP caching (cheap and underused)

For a public status page polled by thousands of browsers, this is real money:

```java
@GetMapping("/status/{slug}")
public ResponseEntity<StatusPageResponse> statusPage(@PathVariable String slug) {
    StatusPageResponse page = service.render(slug);
    return ResponseEntity.ok()
            .cacheControl(CacheControl.maxAge(Duration.ofSeconds(30)).cachePublic())
            .eTag("\"" + page.checksum() + "\"")
            .lastModified(page.generatedAt())
            .body(page);
}
```

With an ETag, a client sends `If-None-Match` and you can return **304 Not Modified** with no body. Spring can do the comparison automatically:

```java
@Bean
ShallowEtagHeaderFilter shallowEtagHeaderFilter() { return new ShallowEtagHeaderFilter(); }
```

**But know the trade-off:** the *shallow* filter buffers the whole response in memory to hash it, so it saves bandwidth, not server work. For genuine savings you want a "deep" ETag from a cheap version/checksum you already have (a row `version` column, an `updated_at`), checked *before* doing the expensive work. Explaining that distinction is a nice piece of depth.

---

## 7. `@Controller` + Thymeleaf (for Vigil's public status page)

```java
@Controller
@RequiredArgsConstructor
public class StatusPageController {

    private final StatusPageService service;

    @GetMapping("/status/{slug}")
    public String show(@PathVariable String slug, Model model) {
        model.addAttribute("page", service.render(slug));
        return "status-page";                     // → templates/status-page.html
    }
}
```

```html
<!-- src/main/resources/templates/status-page.html -->
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<body>
  <h1 th:text="${page.name}">Status</h1>
  <div th:each="m : ${page.monitors}">
    <span th:text="${m.name}"></span>
    <span th:text="${m.status}" th:classappend="${m.status.name().toLowerCase()}"></span>
  </div>
</body>
</html>
```

**Why bother with server-rendered HTML in 2026?** Three real reasons for a status page specifically: it must load when your JS bundle's CDN is down (it's a *status* page — it has to work during outages), it needs to be indexable and instantly cacheable, and it should render in one request with no client-side data fetching. **That's a genuinely good architectural justification, not nostalgia** — and being able to explain *why you chose SSR here and JSON+SPA for the dashboard* is exactly the kind of reasoning senior interviews probe for.

---

## 8. Build & break — Vigil's monitor API

**Now we start the real project.** Create the full `monitor` feature slice in `com.vigil.monitor`.

**1. The DTOs:**
```java
public record CreateMonitorRequest(
        @NotBlank @Size(max = 100) String name,
        @NotBlank @URL String url,
        @Min(30) @Max(86400) int intervalSeconds,
        @NotNull MonitorType type,
        Map<String, String> expectedHeaders
) {}

public record MonitorResponse(
        UUID id, String name, String url, MonitorType type,
        MonitorStatus status, int intervalSeconds,
        Instant lastCheckedAt, Integer lastResponseTimeMs,
        @JsonInclude(NON_NULL) String lastError,
        Instant createdAt
) {}

public record PagedResponse<T>(List<T> content, int page, int size,
                               long totalElements, int totalPages, boolean last) {}
```

**2. The controller** with all five CRUD endpoints, correct status codes, a `Location` header on create, and a criteria object for search. Use an in-memory `ConcurrentHashMap` in the service for now — JPA arrives in Module 6.

**3. Exercise every input binding.** Add endpoints that use `@PathVariable`, `@RequestParam` (required, optional, defaulted, list), `@RequestHeader`, and a criteria object. Then break each one and read the error:
- Omit a required `@RequestParam` → 400, read the message
- Pass `?size=abc` → `MethodArgumentTypeMismatchException`
- Pass `@RequestParam(required = false) int size` with no value → **read this failure carefully; it's the `int` vs `Integer` trap**
- POST malformed JSON → `HttpMessageNotReadableException`
- POST with `Content-Type: text/plain` → **415**
- GET with `Accept: application/xml` → **406**

**4. Control the JSON precisely.** Add a field with `@JsonIgnore`, one renamed with `@JsonProperty`, and an `Instant`. Set `spring.jackson.serialization.write-dates-as-timestamps=true` and look at the output — **then set it back to false.** Seeing the epoch-number form once means you'll recognize it instantly in the wild.

**5. Prove "customize, don't replace."** Define `@Bean ObjectMapper objectMapper() { return new ObjectMapper(); }` and hit an endpoint returning an `Instant`. It fails (or produces garbage) because `JavaTimeModule` is gone. Read the exception. Then delete it and use a `Jackson2ObjectMapperBuilderCustomizer` instead. **This is the exercise that makes the Boot 11 lesson concrete.**

**6. ETags.** Add ETag support to `GET /monitors/{id}`. Verify with:
```bash
curl -i http://localhost:8080/api/v1/monitors/{id}
```
```bash
curl -i -H 'If-None-Match: "0"' http://localhost:8080/api/v1/monitors/{id}
```
Confirm the second returns **304** with an empty body.

**7. The status page.** Add `spring-boot-starter-thymeleaf`, write `StatusPageController` and a minimal template. This is the first thing about Vigil a real user will see.

### Explain out loud (60s)

1. What `@RestController` adds over `@Controller`, and when you'd still want the latter.
2. The `int` vs `Integer` trap on an optional request parameter.
3. Why declaring your own `ObjectMapper` bean breaks date handling.
4. When a request produces 415 and when 406, and who decides.
5. Why returning an entity from a controller is a contract problem, not just a performance one.

---

## 9. Production rules

| Do / Never | Why |
|---|---|
| **Put `@Valid` on every `@RequestBody`** | Constraint annotations on the DTO do nothing without it, so invalid data reaches your service and the validation looks like it's working |
| **Never bind an optional `@RequestParam` to a primitive** | `@RequestParam(required = false) int size` can't hold null, so an absent parameter throws instead of defaulting — use `Integer` or `defaultValue` |
| **Bind a criteria object once you pass about three query parameters** | Spring binds by property name with no extra annotation, and the object can carry `@Min`/`@Max` — eight `@RequestParam`s can't be validated as a unit |
| **Never return an entity from a controller** | It welds your schema to your API contract and invites lazy-loading serialization failures; return a record DTO |
| **Declare `consumes` and `produces` on mappings** | A mismatch then becomes a 415 or 406 during mapping, instead of a confusing failure inside your method |
| **Customize Jackson with `Jackson2ObjectMapperBuilderCustomizer`, never your own `ObjectMapper` bean** | Declaring the bean replaces Boot's, silently dropping `JavaTimeModule` and every other registered default |
| **Keep `write-dates-as-timestamps=false`** | ISO-8601 strings are readable, unambiguous and parseable by every client; epoch arrays are a debugging tax forever |
| **Never trust `MultipartFile.getOriginalFilename()`** | It is attacker-supplied and the classic path-traversal vector — generate your own name and validate size and type |
| **Use `ResponseEntity` only when status or headers are genuinely dynamic** | Otherwise `@ResponseStatus` plus a plain return type reads better and keeps the handler free of HTTP ceremony |
| **Never `try/catch` in a controller** | Exceptions belong to `@ControllerAdvice`, where the mapping to a status code is defined once instead of per method |
| **Put `**` only at the end of a path pattern** | Spring 6's `PathPatternParser` rejects it elsewhere, so an old `/api/**/monitors` fails outright after a Boot 3 upgrade |

---

## 10. Interview traps

**Q1. `@Controller` vs `@RestController`?**
`@RestController` = `@Controller` + `@ResponseBody` on every method, so return values are serialized to the body. `@Controller` returns view names for a template engine. Use `@RestController` for APIs, `@Controller` for server-rendered HTML.

**Q2. `@RequestParam` vs `@PathVariable`?**
`@PathVariable` extracts part of the URL path — used for resource identity, always required. `@RequestParam` reads query string or form data — used for filtering, sorting, pagination, and is required by default unless you set `required=false` or a `defaultValue`.

**Q3. What happens if a required `@RequestParam` is missing?**
`MissingServletRequestParameterException` → 400 Bad Request. But `@RequestParam(required=false) int x` throws instead of yielding null, because a primitive can't be null — use `Integer` or a `defaultValue`.

**Q4. When do you return `ResponseEntity` instead of the object?**
When status or headers are dynamic: 201 with a `Location`, 304 with an ETag, custom headers, or varying status. Otherwise return the object — it's cleaner and makes the response type explicit for OpenAPI and readers. Avoid `ResponseEntity<?>`.

**Q5. How does Spring convert a request body to an object?**
`RequestResponseBodyMethodProcessor` (an argument resolver) selects an `HttpMessageConverter` based on `Content-Type` — normally `MappingJackson2HttpMessageConverter` — which uses Jackson to deserialize. The reverse path serializes the return value based on the `Accept` header.

**Q6. Why would an `Instant` serialize as a number or an array?**
`WRITE_DATES_AS_TIMESTAMPS` is enabled, or `JavaTimeModule` isn't registered. It almost always happens because someone defined their own bare `ObjectMapper` bean, which makes Boot's auto-configuration back off and drops all its default modules and settings.

**Q7. How do you customize Jackson without losing Boot's defaults?**
Add a `Jackson2ObjectMapperBuilderCustomizer` bean, or use `spring.jackson.*` properties. Defining an `ObjectMapper` bean replaces Boot's entirely — the general principle is "customize, don't replace."

**Q8. What status code for POST creating a resource, and what header?**
201 Created, with a `Location` header pointing at the new resource. Build it with `ServletUriComponentsBuilder.fromCurrentRequest().path("/{id}")`.

**Q9. Why is `@JsonTypeInfo(use = CLASS)` dangerous?**
It lets the JSON payload specify a Java class to instantiate — a deserialization gadget attack that has produced numerous CVEs. Always use `use = NAME` with an explicit `@JsonSubTypes` allowlist.

**Q10. Which annotation stops a password from ever appearing in a response?**
`@JsonProperty(access = Access.WRITE_ONLY)` — accepted on input, never serialized on output. `@JsonIgnore` blocks both directions.

**Q11. How would you avoid ten `@RequestParam`s on a search endpoint?**
Bind a criteria object: a record whose components match the parameter names, with `@Valid` for constraints. Spring binds query params to it by name without any annotation.

**Q12. What's the trade-off with `ShallowEtagHeaderFilter`?**
It buffers the entire response in memory to compute the hash, so it saves *bandwidth* but not server work — you've already done the expensive query. A deep ETag from an existing version/timestamp, checked before the expensive work, saves both.

---

## 11. Retention check

1. Give the trap with `@RequestParam(required = false) int`.
2. What does `@RestController` add over `@Controller`, mechanically?
3. Why does defining your own `ObjectMapper` bean break date serialization?
4. What status code and header does a successful POST return?
5. Why is `@JsonTypeInfo(use = NAME)` required rather than `use = CLASS`?
6. Give a real architectural reason to server-render a status page rather than use a SPA.

---

**Next:** [Lesson 16 — API design: DTOs, validation & error handling](16-api-design-dto-validation-errors.md) — the lesson that most improves how your code is judged in a review.

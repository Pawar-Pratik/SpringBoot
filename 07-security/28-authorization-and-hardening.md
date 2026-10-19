# Lesson 28 — Authorization & hardening

> **Why this lesson exists:** Authentication asks "who are you?" — a solved problem you configure. **Authorization asks "may you do this to *this specific object*?"** — and that's where real applications leak data. Broken access control is #1 on the OWASP Top 10, and it's #1 because it's *application-specific*: no framework can fix it for you. This lesson is about getting it right, plus the hardening that keeps a portfolio project from being embarrassing when someone actually looks.

**Time:** ~75 minutes · **Prereq:** Lessons 25–27

---

## 1. The idea in one sentence

> **Authentication is a framework problem you configure once; authorization is "may *this* user do *this* to *this specific row*", which no framework can answer for you — which is exactly why broken access control is first on the OWASP Top 10.**

The practical consequence runs through the whole lesson: prefer controls that can't be forgotten. A tenant-scoped query beats a check after loading, row-level security beats both, and a separate Actuator port beats any amount of correct configuration.

---

## 2. Roles vs authorities vs permissions

Spring Security has exactly one concept: **`GrantedAuthority`**, a string. Everything else is convention.

```java
new SimpleGrantedAuthority("ROLE_ADMIN")      // a "role" — a coarse job description
new SimpleGrantedAuthority("monitors:delete") // a "permission" — a fine-grained capability
```

| | Role | Permission (authority) |
|---|---|---|
| Granularity | Coarse: `ADMIN`, `MEMBER`, `VIEWER` | Fine: `monitors:write`, `billing:read` |
| Count | 3–7 | dozens |
| Changes | Rarely | With every feature |
| Checked as | `hasRole("ADMIN")` | `hasAuthority("monitors:delete")` |

### The design that works: **roles grant permissions; code checks permissions**

```java
public enum OrgRole {
    OWNER (Set.of("monitors:*", "incidents:*", "billing:*", "members:*", "org:delete")),
    ADMIN (Set.of("monitors:*", "incidents:*", "members:read", "members:invite")),
    MEMBER(Set.of("monitors:read", "monitors:write", "incidents:read", "incidents:ack")),
    VIEWER(Set.of("monitors:read", "incidents:read"));

    private final Set<String> permissions;
}
```
```java
// ✅ Code checks the PERMISSION
@PreAuthorize("hasAuthority('monitors:delete')")
public void deleteMonitor(UUID id) { }

// ❌ Code checks the ROLE
@PreAuthorize("hasRole('ADMIN')")
public void deleteMonitor(UUID id) { }
```

**Why this matters concretely:** a customer asks for a "Billing Manager" role that can see invoices but not touch monitors. With permission checks, you add one enum constant and ship. With role checks, you edit `@PreAuthorize` on every affected method, retest everything, and hope you found them all.

**Say it this way in an interview:** *"I model roles as bundles of permissions and have code check permissions, because roles are business vocabulary that changes and permissions are technical capabilities that don't. Checking roles in code hardcodes today's org chart into your source."* That's a genuinely senior framing.

**The `ROLE_` prefix, once more:** `hasRole("X")` looks for the authority `ROLE_X`; `hasAuthority("X")` looks for `X` literally. Pick one convention. Since permissions don't want a `ROLE_` prefix, most codebases store roles as `ROLE_ADMIN` and permissions bare, and use `hasRole` and `hasAuthority` accordingly.

---

## 3. Where to enforce authorization

```
┌─ URL-based:      HttpSecurity.authorizeHttpRequests()      coarse, in one place
├─ Method-based:   @PreAuthorize on services                  the main line of defence
├─ Object-based:   "does THIS user own THIS row?"            ★ where breaches happen
└─ Data-based:     tenant filter / row-level security         the safety net
```

**All four, in layers.** URL rules alone are insufficient — the moment your API has `/monitors/{id}`, URL rules can't tell you whether *this* user may see *that* monitor.

### Layer 1 — URL rules

```java
.authorizeHttpRequests(auth -> auth
    .requestMatchers("/api/v1/auth/**").permitAll()
    .requestMatchers(HttpMethod.GET, "/api/v1/status/**").permitAll()
    .requestMatchers("/api/v1/admin/**").hasRole("SUPERADMIN")
    .requestMatchers(HttpMethod.DELETE, "/api/v1/**").hasAuthority("monitors:delete")
    .anyRequest().authenticated())          // ★ deny by default
```

Good for broad strokes and for making the public surface auditable in one file. **`.anyRequest().authenticated()` last, always** (Lesson 25).

### Layer 2 — Method security

```java
@Configuration
@EnableMethodSecurity                      // required; prePostEnabled is true by default
public class SecurityConfig { }
```

```java
@Service
public class MonitorService {

    @PreAuthorize("hasAuthority('monitors:write')")
    public MonitorResponse create(CreateMonitorRequest req) { }

    // Access method arguments with #paramName
    @PreAuthorize("hasAuthority('monitors:delete') and @monitorGuard.isOwner(#id, principal)")
    public void delete(UUID id) { }

    // Filter a returned collection element by element
    @PostFilter("@monitorGuard.canRead(filterObject, principal)")
    public List<Monitor> findAll() { }

    // Check the RETURN VALUE — note the ordering caveat below
    @PostAuthorize("returnObject.organizationId == principal.orgId")
    public Monitor get(UUID id) { }

    @PreAuthorize("isAuthenticated() and principal.username == #email")
    public void updateProfile(String email, ProfileRequest req) { }
}
```

**Method security is proxy-based (Lesson 09), which means every rule from that lesson applies:**
- `private`/`final`/`static` methods → **silently unchecked**
- **self-invocation → the check is silently skipped**

**That's not a performance quirk — it's an authorization bypass.** A `@PreAuthorize` method called internally via `this.` runs with no check at all. **This is the single most dangerous consequence of proxy-based AOP**, and stating it in an interview is a strong security signal.

**`@PostAuthorize` and `@PostFilter` caveats:**
- `@PostAuthorize` runs **after** the method — so any side effects already happened. Never use it on a method that writes.
- `@PostFilter` loads everything then discards — so it's wrong for pagination (you'd return fewer than `size` items) and wasteful on large sets. **Filter in the query instead** (§4).

**`@Secured` and JSR-250 `@RolesAllowed`** exist but only do role checks with no SpEL. Use `@PreAuthorize`.

### Layer 3 — Object-level authorization: where breaches actually happen

**This is the most important section in the lesson.**

```java
// ❌ IDOR — Insecure Direct Object Reference. OWASP #1.
@GetMapping("/api/v1/monitors/{id}")
public MonitorResponse get(@PathVariable UUID id) {
    return monitorService.get(id);         // ANY authenticated user reads ANY monitor
}
```

The user is authenticated. They have `monitors:read`. **They just don't own monitor `id`** — and nothing checks. Change the ID in the URL and you're reading another customer's data. **This is the most common real-world API vulnerability, and it is invisible to every scanner and every URL-based rule.**

**Fix 1 — scope the query (best):**
```java
// The tenant is part of the WHERE clause. Unauthorized rows cannot be returned.
@Query("SELECT m FROM Monitor m WHERE m.id = :id AND m.organization.id = :orgId")
Optional<Monitor> findByIdAndOrgId(@Param("id") UUID id, @Param("orgId") UUID orgId);
```
```java
@Transactional(readOnly = true)
public MonitorResponse get(UUID id) {
    return repo.findByIdAndOrgId(id, currentUser.orgId())
            .map(mapper::toResponse)
            .orElseThrow(() -> new MonitorNotFoundException(id));    // ★ 404, not 403
}
```

**Why this is better than a permission check after loading:** it's not possible to forget the check *and* return the object, because the object is never loaded. Authorization is expressed as data access rather than as a separate step that can be skipped. **This is the pattern to advocate for.**

**And return 404, not 403.** A 403 confirms the resource exists, letting an attacker enumerate valid IDs across tenants. **For cross-tenant access, "not found" is both safer and truthful** — it doesn't exist *in your tenant*.

**Fix 2 — a guard bean, when the check is complex:**
```java
@Component("monitorGuard")
@RequiredArgsConstructor
public class MonitorGuard {
    private final MonitorRepository repo;

    public boolean isOwner(UUID monitorId, VigilUserDetails principal) {
        return repo.findOrgIdById(monitorId)                       // projection: one column
                .map(orgId -> principal.hasOrgAccess(orgId))
                .orElse(false);
    }
}
```
```java
@PreAuthorize("@monitorGuard.isOwner(#id, principal)")
public MonitorResponse get(UUID id) { }
```
Readable and reusable — but it costs an extra query, and it's a *separate* step that a new endpoint can omit. Prefer Fix 1 as the default and use guards for genuinely complex rules.

### Layer 4 — Data-level: the safety net

From Lesson 24 §5: a Hibernate `@Filter`, an unconditional `Specification` predicate, and **Postgres Row-Level Security**.

```sql
ALTER TABLE monitor ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON monitor
    USING (org_id = current_setting('app.current_org', true)::uuid);
```

**RLS is the layer that holds when your application code has a bug** — the database itself refuses to return other tenants' rows. Costs: a `SET LOCAL app.current_org` per transaction (which interacts with connection pooling, so it must be per-transaction, not per-connection), and some query plans get harder. **Worth it for a multi-tenant SaaS handling other people's data.**

---

## 4. Multi-tenant authorization — the Vigil model

Vigil's structure: a user belongs to multiple **organizations** with a **role in each**. So "am I an admin?" is meaningless without "of which organization?"

```java
public record VigilUserDetails(
        UUID userId,
        String email,
        String passwordHash,
        Map<UUID, OrgRole> orgRoles,       // ★ org → role
        boolean enabled
) implements UserDetails {

    public boolean hasPermission(UUID orgId, String permission) {
        OrgRole role = orgRoles.get(orgId);
        return role != null && role.grants(permission);
    }
}
```

**The tenant must come from a trusted source, never from the request body:**

```java
// ❌ CATASTROPHIC — the client tells you which tenant they are
@PostMapping("/monitors")
public MonitorResponse create(@RequestBody CreateMonitorRequest req) {
    return service.create(req.organizationId(), req);      // client-supplied tenant! 💀
}

// ✅ The tenant comes from the token or the path, and is VERIFIED against the user's memberships
@PostMapping("/orgs/{orgId}/monitors")
@PreAuthorize("@orgGuard.hasPermission(#orgId, 'monitors:write')")
public MonitorResponse create(@PathVariable UUID orgId,
                              @Valid @RequestBody CreateMonitorRequest req) {
    return service.create(orgId, req);
}
```

**Even with `orgId` in the path it must be verified** — a path variable is still client input. The guard checks it against the authenticated user's actual memberships. The point is not *where* the tenant identifier comes from; it's that **it is always validated against server-side truth.**

**The `TenantContext` pattern** for propagating it:
```java
public final class TenantContext {
    private static final ThreadLocal<UUID> CURRENT = new ThreadLocal<>();
    public static void set(UUID orgId) { CURRENT.set(orgId); }
    public static UUID require() {
        UUID id = CURRENT.get();
        if (id == null) throw new IllegalStateException("no tenant in context");
        return id;
    }
    public static void clear() { CURRENT.remove(); }        // ★ MANDATORY in a finally
}
```

Set it in a filter after authentication, clear it in a `finally`. **The `clear()` is not optional** — a pooled thread retaining the previous request's tenant is a cross-tenant data leak, and it's the same class of bug as MDC leakage (Lesson 14) but with far worse consequences. **And it doesn't cross `@Async`** — Lesson 08's lesson, fourth appearance. For async work, pass the tenant explicitly.

---

## 5. The OWASP issues that actually appear in Spring apps

### A01 — Broken Access Control (#1, and by a wide margin)

Covered above. **The checklist:**
- [ ] Every endpoint returning a resource by ID verifies ownership — ideally by scoping the query
- [ ] Deny-by-default URL rules
- [ ] No client-supplied tenant/user IDs trusted for authorization
- [ ] 404 rather than 403 for cross-tenant access
- [ ] **Integration tests asserting user A cannot read user B's data** — the only way to be sure
- [ ] Mass-assignment prevented (input DTOs, never entities — Lesson 16)

### A02 — Cryptographic Failures
- TLS everywhere; HSTS; no mixed content
- BCrypt/Argon2 for passwords (Lesson 25); **never** for anything you need to decrypt
- `SecureRandom` for tokens (Lesson 26)
- Encrypt sensitive columns at rest (`AttributeConverter`, Lesson 21) — and note you lose queryability
- **Never log secrets.** Redact in filters and audit aspects (Lessons 10, 17)

### A03 — Injection

**SQL injection** — parameterize (Lesson 19). The remaining risks in a Spring app:
- Native `@Query` with concatenation
- Dynamic `ORDER BY` — **cannot be parameterized; validate against an allowlist**
- `LIKE` patterns built from user input (also a ReDoS/performance risk)

**XSS** — for a JSON API, Jackson escapes output, so the risk is mostly in server-rendered templates. **Thymeleaf escapes by default with `th:text`; `th:utext` does not.** Vigil's status page uses `th:text` throughout. Add a strict CSP.

**Others worth naming:** command injection (`ProcessBuilder` with user input), path traversal (`../` in filenames — Lesson 15), LDAP injection, and **SpEL injection** — evaluating user input as a SpEL expression is remote code execution, and it has produced real Spring CVEs. Never build a SpEL expression from user input.

### A04 — Insecure Design
Rate limiting (Lesson 17), lockout after failed logins, MFA support, secure password reset (single-use, short-lived, `SecureRandom` tokens; don't reveal whether the email exists).

### A05 — Security Misconfiguration
- Actuator: **expose only `health` and `info` publicly**; everything else authenticated (§6)
- `server.error.include-stacktrace=never` (Lesson 16)
- Swagger UI disabled or authenticated for internal APIs (Lesson 17)
- No default credentials; no debug flags; `@EnableWebSecurity(debug=true)` off
- CORS not wildcarded with credentials (Lesson 17)

### A06 — Vulnerable and Outdated Components

**The one people neglect, and it's the one that gets you.** Log4Shell (CVE-2021-44228) and Spring4Shell were both dependency problems.

```xml
<plugin>
    <groupId>org.owasp</groupId>
    <artifactId>dependency-check-maven</artifactId>
    <configuration><failBuildOnCVSS>7</failBuildOnCVSS></configuration>
</plugin>
```
Plus GitHub Dependabot, and staying current on Spring Boot patch releases. **Boot's BOM is itself a security feature** — one version bump moves ~450 dependencies to tested-compatible, patched versions (Lesson 01).

### A07 — Identification and Authentication Failures
Prevent user enumeration (Lesson 25), short access tokens with revocable refresh tokens (Lesson 26), invalidate sessions on password change (bump `token_version`), and rate-limit login attempts.

### A08 — Software and Data Integrity Failures
**Insecure deserialization** — the Jackson polymorphic typing issue from Lesson 15: always `@JsonTypeInfo(use = NAME)` with an explicit `@JsonSubTypes` allowlist, never `use = CLASS` or default typing. And never deserialize untrusted input with Java's native `ObjectInputStream`.

### A09 — Security Logging and Monitoring Failures
Log authentication successes and failures, authorization denials, and admin actions — with correlation IDs (Lessons 10, 35). **Redact everything sensitive.** Alert on spikes in 401/403 rates: a sudden burst of 403s is often enumeration in progress.

### A10 — SSRF

**Vigil is an SSRF magnet by design** — its whole purpose is fetching user-supplied URLs. Covered in Lesson 16 §3, and worth restating because it's the most project-specific risk you have:

- Validate the scheme (http/https only)
- **Block private, loopback, and link-local addresses** — especially `169.254.169.254`, the cloud metadata endpoint that hands out IAM credentials
- **Resolve DNS and check the resolved IP at connection time**, not just at validation time — DNS rebinding can change the answer in between
- Disable redirects, or re-validate every hop (a public URL can 302 to `localhost`)
- Never return the response body to the user; return only status and timing
- **Best defence: run checkers in a network-isolated environment** with egress rules that make internal addresses unreachable regardless of application bugs

**That last point is the mature answer:** application-level SSRF validation is defence in depth, but *network* isolation is the control that actually holds. **Being able to say "I'd validate in the app and also make it structurally impossible at the network layer" is exactly the kind of layered thinking senior interviews reward.**

---

## 6. Securing Actuator

Actuator endpoints are extremely useful and extremely dangerous.

| Endpoint | Risk if public |
|---|---|
| `/env`, `/configprops` | **Leaks configuration — often including secrets** |
| `/heapdump` | **A full memory dump: passwords, tokens, PII, in plaintext** |
| `/threaddump` | Internal structure, possibly data in stack frames |
| `/mappings` | Complete API surface map |
| `/loggers` | **Writable** — an attacker can enable DEBUG logging to flood disk or leak data |
| `/shutdown` | **Stops your application** (disabled by default, thankfully) |

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus         # ★ allowlist, never '*'
      base-path: /internal                       # non-obvious path (weak, but free)
  endpoint:
    health:
      show-details: when-authorized              # ★ never 'always' — it leaks component names
      probes:
        enabled: true
  server:
    port: 9090                                   # ★★ a SEPARATE port, not exposed publicly
```

```java
.requestMatchers(EndpointRequest.to(HealthEndpoint.class, InfoEndpoint.class)).permitAll()
.requestMatchers(EndpointRequest.toAnyEndpoint()).hasRole("SUPERADMIN")
```

**`management.server.port` on a separate port is the strongest control** — your ingress/load balancer only exposes 8080, so Actuator is simply unreachable from the internet regardless of application configuration. **Same principle as network-level SSRF defence: prefer controls that don't depend on getting the application right.**

> **Real-world note:** publicly exposed `/actuator/heapdump` and `/actuator/env` are a well-known finding in bug bounty programs and internet-wide scans. It is a genuinely common, genuinely serious misconfiguration — and it's why this section exists.

---

## 7. Build & break — securing Vigil properly

**1. The permission model.** `OrgRole` enum with permission sets. `VigilUserDetails` with `Map<UUID, OrgRole>`. `hasPermission(orgId, permission)`.

**2. `@EnableMethodSecurity`** and `@PreAuthorize` with permission checks (not role checks) on every mutating service method.

**3. The `OrgGuard` bean** with `hasPermission(UUID orgId, String permission)`, used from `@PreAuthorize` as `@orgGuard.hasPermission(#orgId, 'monitors:write')`.

**4. ⭐ Exploit your own IDOR, then fix it.** This is the exercise that matters most.
- Create two organizations with one user each
- Write `GET /api/v1/monitors/{id}` **without** a tenant check
- As user A, request user B's monitor ID. **Read user B's data.** Sit with that for a moment.
- Fix it with `findByIdAndOrgId` and confirm you now get **404**
- Write the test that permanently prevents its return:
```java
@Test
void userCannotReadAnotherOrgsMonitor() throws Exception {
    mockMvc.perform(get("/api/v1/monitors/{id}", orgBMonitorId)
                    .header("Authorization", "Bearer " + orgAToken))
           .andExpect(status().isNotFound());     // 404, not 403 — no existence disclosure
}
```
**That test is a security control, and it's the kind of thing a reviewer notices immediately.**

**5. Restructure to tenant-scoped paths.** `/api/v1/orgs/{orgId}/monitors/...`, with the guard verifying `orgId` against memberships. Confirm that passing another org's ID gives 403/404.

**6. Row-Level Security.** Enable RLS on `monitor` and `check_result`. Set `app.current_org` per transaction. Then **deliberately remove the application-level tenant filter** and confirm the database still refuses to leak. **Watching your safety net catch a bug you introduced on purpose is the best possible demonstration of defence in depth.**

**7. `TenantContext` with a filter.** Set after authentication, clear in a `finally`. Then **remove the `clear()`**, hammer the API concurrently as two different tenants, and watch tenants bleed across requests. **This is the most alarming experiment in the module — do it once, in a test environment.**

**8. Lock down Actuator.** Move it to port 9090, allowlist `health,info,prometheus`, `show-details: when-authorized`. **Before** locking it down, hit `/actuator/env` and `/actuator/heapdump` and look at what's in them. Search the heapdump for your database password (`strings heapdump | grep -i password`). **Finding your own credentials in a file an anonymous HTTP request could have downloaded is a formative experience.**

**9. SSRF defence.** The `@SafeUrl` validator from Lesson 16, plus a connection-time IP check in the HTTP checker, plus redirect handling. Test with `http://localhost:8080/actuator/env`, `http://169.254.169.254/`, `http://127.0.0.1:5432`, and a public URL that redirects to localhost.

**10. Dependency scanning.** Add `dependency-check-maven` with `failBuildOnCVSS=7` and run it. Read the report. Then enable Dependabot on the repo.

**11. Security headers.** Configure CSP, HSTS, `X-Frame-Options`, and `Referrer-Policy`. Verify with `curl -I` and then run the deployed site through [securityheaders.com](https://securityheaders.com). **An A grade there is a nice concrete thing to mention in an interview.**

**12. The security test suite.** Write these — they're worth more than the features:
```java
@Test void unauthenticatedIs401() { }
@Test void wrongRoleIs403() { }
@Test void crossTenantReadIs404() { }
@Test void crossTenantWriteIs404() { }
@Test void crossTenantDeleteIs404() { }
@Test void listEndpointNeverReturnsOtherTenantsRows() { }
@Test void ssrfToPrivateAddressIsRejected() { }
@Test void ssrfViaRedirectIsRejected() { }
@Test void actuatorEnvRequiresAuthentication() { }
@Test void passwordHashIsNeverInAnyResponseBody() { }
@Test void massAssignmentOfOrgIdIsIgnored() { }
```

### Explain out loud (60s)

1. Why no framework can solve authorization for you.
2. IDOR: the bug, the query-scoping fix, and why it beats a check after loading.
3. Why 404 rather than 403 across tenants.
4. Why a self-invoked `@PreAuthorize` method is a security bug, not a performance one.
5. Three Actuator endpoints that must never be public, and the control that makes it moot.

---

## 8. Production rules

| Do / Never | Why |
|---|---|
| **Model roles as bundles of permissions, and have code check permissions** | Roles are business vocabulary that changes; a new "Billing Manager" role should be one enum constant, not an edit to every `@PreAuthorize` |
| **Scope the query by tenant rather than checking ownership after loading** | You cannot forget the check *and* return the object if the object is never loaded — authorization becomes data access rather than a separate step |
| **Return 404, not 403, for another tenant's resource** | A 403 confirms the row exists, which is an ID-enumeration oracle; "not found" is both safer and truthful in your tenant |
| **Never take the tenant from the request body** | It is client input; the tenant must come from the authenticated principal, and even a path variable must be validated against real memberships |
| **Never call a `@PreAuthorize` method via `this.`** | Method security is proxy-based, so a self-invoked call runs with no check at all — a silent authorization bypass, not a performance quirk |
| **Use `@PreAuthorize`, not `@Secured` or `@RolesAllowed`** | Only `@PreAuthorize` supports SpEL, which is what lets you express object-level and permission-based rules at all |
| **Avoid `@PostFilter` on anything paginated** | It loads everything and discards after the fact, so your page sizes become wrong as well as your performance |
| **Add row-level security for multi-tenant data** | It is the layer that still holds when your application code has a bug, which is the only kind of control that survives a bad day |
| **Clear any `TenantContext` `ThreadLocal` in a `finally`** | Worker threads are pooled, so a stale tenant on a reused thread serves one customer's data to another |
| **Run Actuator on a separate port** | Your ingress then exposes only 8080, so `/env` and `/heapdump` are unreachable regardless of how the application is configured |
| **Never expose `/heapdump` or `/env`** | A heap dump is every password, token and PII value in plaintext, and both are standard findings in internet-wide scans |
| **Never build a SpEL expression from user input** | Evaluating user-supplied SpEL is remote code execution, and it has produced real Spring CVEs |
| **Use `@JsonTypeInfo(use = NAME)` with an explicit `@JsonSubTypes` allowlist** | Class-based polymorphic typing lets an attacker choose which class to instantiate, which is deserialization RCE |
| **Scan dependencies and fail the build on high CVSS** | Log4Shell and Spring4Shell were both dependency problems, and the gap is always between disclosure and your next upgrade |
| **Defend SSRF in the app *and* at the network layer** | Application validation is defence in depth; network isolation is the control that holds when the validation has a gap |

---

## 9. Interview traps

**Q1. Difference between roles and authorities in Spring Security?**
Mechanically none — both are `GrantedAuthority` strings. By convention roles are coarse (`ROLE_ADMIN`) and permissions fine (`monitors:delete`). `hasRole("X")` prepends `ROLE_`; `hasAuthority("X")` is literal.

**Q2. Should code check roles or permissions?**
Permissions. Roles are business vocabulary that changes; permissions are technical capabilities that don't. Model roles as bundles of permissions so adding a new role is one enum change, not an audit of every `@PreAuthorize`. Checking roles in code hardcodes today's org chart into your source.

**Q3. What is IDOR and how do you prevent it?**
Insecure Direct Object Reference: an authenticated, permitted user accesses a resource they don't own by supplying its ID. Prevent by scoping the query (`findByIdAndOrgId`) so unauthorized rows are never loaded — stronger than a post-load check, which can be forgotten. Return **404, not 403**, to avoid confirming existence. And test it.

**Q4. `@PreAuthorize` vs `@PostAuthorize` vs `@PostFilter`?**
`@PreAuthorize` runs before the method — the default and the safe one. `@PostAuthorize` checks the return value **after execution**, so side effects have already happened; never use it on writes. `@PostFilter` filters a returned collection element-by-element — it loads everything then discards, so it breaks pagination and is wasteful; filter in the query instead.

**Q5. Why is `@PreAuthorize` on a self-invoked method a security bug?**
Method security is proxy-based, so `this.method()` bypasses the proxy and the check **never runs** — silently. Same for `private`, `final`, and `static` methods. It's not a performance quirk; it's an authorization bypass.

**Q6. How do you enforce multi-tenant isolation?**
Layers: (1) the tenant identifier comes from the token or a verified path variable, never from a request body; (2) it's validated against the user's actual memberships; (3) every query includes it — ideally structurally, via a `Specification` predicate or Hibernate `@Filter`; (4) Postgres Row-Level Security as a database-enforced safety net; (5) integration tests asserting cross-tenant reads return nothing.

**Q7. 403 or 404 for a resource the user can't access?**
404 for cross-tenant access. A 403 confirms the resource exists, enabling ID enumeration across tenants. "Not found" is also honest — it doesn't exist within the requester's tenant. Use 403 when the user can see the resource but lacks permission for the specific operation.

**Q8. Why is Vigil especially exposed to SSRF, and how do you defend it?**
Its core function is fetching user-supplied URLs. Defences: scheme allowlist; block private/loopback/link-local IPs (notably `169.254.169.254`, which serves cloud IAM credentials); **check the resolved IP at connection time**, not only at validation (DNS rebinding); disable or re-validate redirects; never return the response body. **And most importantly: network egress isolation, so internal addresses are unreachable even if the application code is wrong.**

**Q9. Which Actuator endpoints must never be public, and what's the best control?**
`/env` and `/configprops` (config, often secrets), `/heapdump` (a full memory dump containing credentials and PII in plaintext), `/threaddump`, `/mappings`, `/loggers` (writable). Best control: `management.server.port` on a separate port that the load balancer doesn't expose — so they're unreachable regardless of application configuration. Then allowlist `health,info` and require authentication for the rest.

**Q10. What's mass assignment and how do you prevent it?**
Binding a request body straight onto an entity, letting a client set fields it shouldn't — `id`, `status`, `organizationId`, `role`. Prevent with explicit input DTOs containing only client-settable fields (Lesson 16). `@JsonProperty(access = READ_ONLY)` helps on shared shapes, but separate DTOs are the real answer.

**Q11. Why is broken access control #1 on the OWASP Top 10?**
Because it's application-specific: no framework or scanner knows your ownership rules, so nothing can detect a missing one for you. Every other category has tooling; this one only has design discipline and tests.

**Q12. How do you avoid logging sensitive data?**
Redact in the request-logging filter (query strings carry tokens and API keys), redact arguments in audit aspects, `@JsonProperty(access = WRITE_ONLY)` on password fields, `@ToString(onlyExplicitlyIncluded = true)` on entities, `include-stacktrace=never`, and never log whole request bodies by default. Then grep your own logs for `password|token|secret|authorization` as a check.

---

## 10. Retention check

1. Why check permissions rather than roles, framed as a change-cost argument?
2. Explain IDOR and give the strongest fix, plus why it beats a post-load check.
3. Why is 404 preferred over 403 for cross-tenant access?
4. Why is a `@PreAuthorize` method called via `this.` a security hole?
5. Name the four layers of tenant isolation, and which one survives an application bug.
6. What's in a heap dump, and what's the best control for Actuator?

---

## Module 7 complete ✅

You can now build authentication (password, JWT, OAuth2/OIDC), enforce layered authorization including object-level and tenant isolation, and harden an application against the issues that actually appear in production Spring systems.

**Next:** [Lesson 29 — The testing pyramid & unit tests](../08-testing/29-testing-pyramid-and-unit.md) — the module that most distinguishes senior engineers from juniors.

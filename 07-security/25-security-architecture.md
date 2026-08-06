# Lesson 25 — Spring Security architecture

> **Why this lesson exists:** Spring Security has a reputation for being impenetrable, and that reputation comes entirely from people using it without understanding the filter chain. It's actually a very cleanly designed system: a chain of filters, each with one job. Once you can draw that chain, debugging a 401 or 403 takes a minute instead of an afternoon, and "how does Spring Security work?" becomes one of your best answers.

**Time:** ~80 minutes · **Prereq:** Lesson 14 (the filter chain)

---

## 1. The mental model

Spring Security is **one servlet `Filter`** that delegates to a chain of internal filters.

```
Tombat worker thread
    │
    ├─ Servlet filter chain
    │   ├─ CharacterEncodingFilter
    │   ├─ ★ springSecurityFilterChain  ──  DelegatingFilterProxy
    │   │       │
    │   │       └─▶ FilterChainProxy       (a Spring bean; knows all SecurityFilterChains)
    │   │              │
    │   │              └─▶ matches the request to ONE SecurityFilterChain, then runs it:
    │   │                    1.  DisableEncodeUrlFilter
    │   │                    2.  WebAsyncManagerIntegrationFilter
    │   │                    3.  SecurityContextHolderFilter      ← loads/clears the context
    │   │                    4.  HeaderWriterFilter               ← security response headers
    │   │                    5.  CorsFilter                       ← preflight handling  [L17]
    │   │                    6.  CsrfFilter                       ← CSRF token check
    │   │                    7.  LogoutFilter
    │   │                    8.  ★ AUTHENTICATION FILTERS
    │   │                        · UsernamePasswordAuthenticationFilter (form login)
    │   │                        · BearerTokenAuthenticationFilter (JWT)  [L26]
    │   │                        · OAuth2LoginAuthenticationFilter        [L27]
    │   │                        · your custom filters
    │   │                    9.  RequestCacheAwareFilter
    │   │                   10.  SecurityContextHolderAwareRequestFilter
    │   │                   11.  AnonymousAuthenticationFilter    ← "anonymous" principal
    │   │                   12.  SessionManagementFilter
    │   │                   13.  ★ ExceptionTranslationFilter     ← turns exceptions into 401/403
    │   │                   14.  ★ AuthorizationFilter            ← LAST: allow or deny
    │   │
    │   └─ your custom filters
    │
    └─▶ DispatcherServlet → controller
```

**Three things to take from this diagram:**

1. **Authentication and authorization happen entirely before your controller runs.** By the time your method executes, the user is either authenticated and authorized, or the request never arrived. That's why you don't write `if (!user.isAdmin())` in controllers.

2. **`ExceptionTranslationFilter` (13) sits just outside `AuthorizationFilter` (14).** It catches `AuthenticationException` → 401, and `AccessDeniedException` → 403. **This ordering is the entire explanation of 401 vs 403** (§4).

3. **The order is fixed and meaningful.** CSRF before authentication (so a stolen session can't be abused via CSRF even if valid). `AnonymousAuthenticationFilter` before authorization (so there's always *some* `Authentication` object, never null). `AuthorizationFilter` last (so all identity information is available).

### `DelegatingFilterProxy` — why it exists

The servlet container creates filters; it knows nothing about Spring beans. `DelegatingFilterProxy` is a plain servlet filter registered with the container that, on each request, looks up a Spring bean named `springSecurityFilterChain` and delegates to it. **That's the bridge between the servlet world and the Spring world** — and it's why your security filters can be beans with injected dependencies. Nice, concrete answer to a question people fumble.

---

## 2. The core objects

```java
// Authentication — the current principal, and the answer to "who is this?"
public interface Authentication extends Principal {
    Object getPrincipal();                                     // usually a UserDetails
    Object getCredentials();                                   // password/token — cleared after auth
    Collection<? extends GrantedAuthority> getAuthorities();    // roles/permissions
    boolean isAuthenticated();
    Object getDetails();
}
```

```java
// SecurityContext — a holder, so the Authentication can be swapped atomically
public interface SecurityContext { Authentication getAuthentication(); ... }

// SecurityContextHolder — a ThreadLocal (by default)
SecurityContextHolder.getContext().getAuthentication();
```

**`SecurityContextHolder` is `ThreadLocal`-based**, which means everything from Lesson 08 §1 applies: **it does not propagate to `@Async`, `@Scheduled`, executors, or reactive threads.** Same root cause as MDC and request scope. Fixes:

```java
// Option 1: a security-aware executor
@Bean
TaskExecutor securityAwareExecutor(ThreadPoolTaskExecutor delegate) {
    return new DelegatingSecurityContextTaskExecutor(delegate);
}

// Option 2: change the strategy so child threads inherit it
SecurityContextHolder.setStrategyName(SecurityContextHolder.MODE_INHERITABLETHREADLOCAL);
```
**Option 2 is a trap with thread pools** — a pooled thread inherits the context of whichever request first created it and keeps it forever. Use the delegating executor.

### Accessing the current user

```java
// ❌ Works, but couples business logic to a static holder and is awkward to test
var auth = SecurityContextHolder.getContext().getAuthentication();

// ✅ Inject it into the controller method — Spring resolves it
@GetMapping("/me")
public UserResponse me(@AuthenticationPrincipal VigilUserDetails user) { ... }

// ✅ Or a custom annotation via an argument resolver (Lesson 14 §3)
@GetMapping("/monitors")
public List<MonitorResponse> list(@CurrentOrg UUID orgId) { ... }
```

**In the service layer**, wrap the holder behind a small interface so it's mockable:
```java
public interface CurrentUserProvider { VigilUser require(); Optional<VigilUser> find(); }
```
That one abstraction makes every service test trivial instead of requiring a `SecurityContext` setup. Small design move, disproportionate payoff — and a nice thing to point out in a review.

---

## 3. The authentication flow

```
   Request with credentials
        │
        ▼
   Authentication FILTER  (e.g. UsernamePasswordAuthenticationFilter)
        │  builds an UNAUTHENTICATED Authentication token
        ▼
   AuthenticationManager  (ProviderManager)
        │  iterates its AuthenticationProviders, asking "can you handle this token type?"
        ▼
   AuthenticationProvider  (e.g. DaoAuthenticationProvider)
        │
        ├─▶ UserDetailsService.loadUserByUsername(username)
        │        └─▶ your database → UserDetails (username, hashed password, authorities)
        │            └─ throws UsernameNotFoundException if absent
        │
        ├─▶ PasswordEncoder.matches(rawPassword, storedHash)
        │
        └─▶ UserDetailsChecker: isEnabled? isAccountNonLocked? isCredentialsNonExpired?
        │
        ▼
   AUTHENTICATED Authentication  (credentials erased, authorities populated)
        │
        ▼
   SecurityContextHolder.getContext().setAuthentication(auth)
        │
        ▼
   AuthorizationFilter → your controller
```

**The pieces you implement, and the ones you don't:**

| Component | You implement it? |
|---|---|
| `AuthenticationManager` / `ProviderManager` | ❌ provided |
| `AuthenticationProvider` | ❌ usually (only for exotic auth) |
| **`UserDetailsService`** | ✅ **yes — loads your users** |
| **`PasswordEncoder`** | ❌ pick one; ✅ declare the bean |
| `UserDetails` | ✅ usually, wrapping your entity |

```java
@Service
@RequiredArgsConstructor
public class VigilUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    @Override
    @Transactional(readOnly = true)
    public UserDetails loadUserByUsername(String email) throws UsernameNotFoundException {
        return userRepository.findByEmailIgnoreCase(email)
                .map(VigilUserDetails::new)
                .orElseThrow(() -> new UsernameNotFoundException("user not found"));
        //                                                       ↑ deliberately vague
    }
}
```

**"user not found" and not "no user with email x@y.com":** the message can reach logs and, misconfigured, responses. **A distinguishable "user not found" vs "bad password" is a user-enumeration oracle** — an attacker learns which emails have accounts. Spring's `DaoAuthenticationProvider` already defends against this by converting `UsernameNotFoundException` into `BadCredentialsException` (controlled by `hideUserNotFoundExceptions`, default `true`) **and** by running a dummy password hash when the user doesn't exist, so the response time doesn't leak the answer either.

**That timing defence is a lovely detail** — mentioning it shows you've read the source, not just the docs.

### `UserDetails`

```java
public class VigilUserDetails implements UserDetails {
    private final UUID id;
    private final String email;
    private final String passwordHash;
    private final Set<GrantedAuthority> authorities;
    private final boolean enabled;

    public VigilUserDetails(User user) {
        this.id = user.getId();
        this.email = user.getEmail();
        this.passwordHash = user.getPasswordHash();
        this.enabled = user.isEnabled();
        this.authorities = user.getMemberships().stream()
                .map(m -> new SimpleGrantedAuthority("ROLE_" + m.getRole().name()))
                .collect(toSet());
    }

    @Override public String getUsername() { return email; }
    @Override public String getPassword() { return passwordHash; }
    @Override public Collection<? extends GrantedAuthority> getAuthorities() { return authorities; }
    @Override public boolean isEnabled() { return enabled; }
    @Override public boolean isAccountNonExpired() { return true; }
    @Override public boolean isAccountNonLocked() { return true; }
    @Override public boolean isCredentialsNonExpired() { return true; }

    public UUID getId() { return id; }        // ← your own accessor, used by @AuthenticationPrincipal
}
```

**Design note worth stating:** wrap your entity in a `UserDetails` rather than making your `User` entity implement `UserDetails`. If the entity implements it, you've coupled your domain model to a security framework, and — worse — a lazily-loaded entity held in the `SecurityContext` becomes detached, so touching an association throws `LazyInitializationException` from inside a security filter. That's a genuinely confusing bug with an easy prevention. **Copy what you need at authentication time; keep the security object small and immutable.**

---

## 4. 401 vs 403 — the debugging skill

**`ExceptionTranslationFilter` is where the distinction is made:**

```java
try {
    chain.doFilter(request, response);                 // AuthorizationFilter runs inside here
} catch (AuthenticationException ex) {
    // NOT AUTHENTICATED → delegate to AuthenticationEntryPoint → 401
    authenticationEntryPoint.commence(request, response, ex);
} catch (AccessDeniedException ex) {
    if (isAnonymous(currentUser)) {
        authenticationEntryPoint.commence(request, response, ex);   // ← anonymous + denied = 401
    } else {
        accessDeniedHandler.handle(request, response, ex);          // ← authenticated + denied = 403
    }
}
```

| Symptom | Meaning | Look at |
|---|---|---|
| **401 Unauthorized** | No valid authentication, or authentication failed | Is the credential/token present, well-formed, unexpired? Is the auth filter in the chain? Did `UserDetailsService` find the user? |
| **403 Forbidden** | Authenticated, but not permitted | Does the user have the required authority? **Is the `ROLE_` prefix right?** Is CSRF blocking a state-changing request? |

**The two 403 causes that trip up nearly everyone:**

**1. The `ROLE_` prefix.**
```java
.hasRole("ADMIN")           // checks for the authority "ROLE_ADMIN"   ← prefix ADDED
.hasAuthority("ADMIN")      // checks for the authority "ADMIN"        ← literal
```
Store `ROLE_ADMIN` in your authorities and use `hasRole("ADMIN")`, **or** store `ADMIN` and use `hasAuthority("ADMIN")`. Mixing them gives a 403 for a user who visibly "has the role." **This is the #1 Spring Security confusion**, and being able to state it crisply is worth real points.

**2. CSRF on a POST.** CSRF protection is **on by default**. A POST from a REST client without a CSRF token gets a 403 that looks like an authorization failure. For a stateless token-authenticated API you disable CSRF — see §6.

### Debugging tools — learn these now

```properties
logging.level.org.springframework.security=DEBUG
logging.level.org.springframework.security.web.FilterChainProxy=TRACE
```

**With `DEBUG` enabled, Spring Security prints the entire filter chain at startup**, in order, for each `SecurityFilterChain`. Read it. It's the single fastest way to answer "is my custom filter actually in the chain, and where?"

```java
@Bean
SecurityFilterChain chain(HttpSecurity http) throws Exception {
    // Spring Security 6.4+: a huge debugging aid
    http.with(new org.springframework.security.config.debug.SecurityDebugBeanFactoryPostProcessor(), c -> {});
    // or simply:
    // @EnableWebSecurity(debug = true)
}
```

`@EnableWebSecurity(debug = true)` logs each request's journey through the chain — which filter did what. **Never in production** (it logs request details), but invaluable locally.

---

## 5. Password encoding

```java
@Bean
PasswordEncoder passwordEncoder() {
    return PasswordEncoderFactories.createDelegatingPasswordEncoder();   // ✅ the right answer
}
```

**Why the delegating encoder specifically:** it stores the algorithm as a prefix in the hash:
```
{bcrypt}$2a$10$N9qo8uLOickgx2ZMRZoMye...
{argon2}$argon2id$v=19$m=16384,t=2,p=1$...
```
So it can **verify old hashes while writing new ones with a stronger algorithm** — algorithm migration without forcing a password reset. That's a real operational capability, and citing it beats naming an algorithm.

| Encoder | Verdict |
|---|---|
| **BCrypt** | ✅ Solid default. Strength 10–12 (each +1 doubles the cost). Note: **truncates input at 72 bytes** |
| **Argon2id** | ✅ Current best practice — memory-hard, resists GPU/ASIC attacks. The OWASP recommendation |
| **SCrypt** | ✅ Also memory-hard, fine |
| **PBKDF2** | ⚠️ Acceptable, FIPS-approved, but weakest of the four against dedicated hardware |
| `NoOpPasswordEncoder` | ❌ Deprecated. Tests only, never anywhere else |
| MD5 / SHA-1 / plain SHA-256 | ❌❌ **Not password hashes.** Fast by design = billions of guesses/second |

**The concept to be able to explain:** a password hash must be **deliberately slow and memory-hard**. SHA-256 is a *fast* hash — that's its purpose — so a GPU can try billions per second against a leaked database. BCrypt/Argon2 have a tunable work factor, adjustable as hardware improves, and include a **per-password salt** so identical passwords produce different hashes and rainbow tables are useless.

**Tuning guidance:** target ~250–500ms per hash on your production hardware. Too fast is insecure; too slow is a self-inflicted DoS (a login flood becomes a CPU exhaustion attack). Measure it, don't guess.

---

## 6. Configuring the filter chain

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity                                  // enables @PreAuthorize  [Lesson 28]
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtFilter;
    private final VigilAuthenticationEntryPoint entryPoint;
    private final VigilAccessDeniedHandler deniedHandler;

    @Bean
    SecurityFilterChain apiFilterChain(HttpSecurity http) throws Exception {
        return http
            .securityMatcher("/api/**")                         // this chain handles /api/** only
            .csrf(AbstractHttpConfigurer::disable)              // ✅ safe: stateless + Bearer tokens
            .cors(Customizer.withDefaults())                    // ★ required, see Lesson 17
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/v1/status/**").permitAll()
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())                  // ★ deny-by-default
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
            .exceptionHandling(e -> e
                .authenticationEntryPoint(entryPoint)           // 401 body
                .accessDeniedHandler(deniedHandler))            // 403 body
            .headers(h -> h
                .frameOptions(FrameOptionsConfig::deny)
                .contentSecurityPolicy(csp -> csp.policyDirectives("default-src 'self'"))
                .httpStrictTransportSecurity(hsts -> hsts
                        .includeSubDomains(true).maxAgeInSeconds(31536000)))
            .build();
    }

    @Bean
    @Order(2)                                                    // ★ ordering matters
    SecurityFilterChain publicFilterChain(HttpSecurity http) throws Exception {
        return http
            .securityMatcher("/status/**", "/css/**", "/js/**")   // server-rendered status pages
            .authorizeHttpRequests(auth -> auth.anyRequest().permitAll())
            .csrf(AbstractHttpConfigurer::disable)
            .build();
    }
}
```

### Multiple chains and ordering

`FilterChainProxy` picks the **first** chain whose `securityMatcher` matches, and runs only that one. So:
- **Order chains from most specific to least specific.** A chain with no matcher matches everything and must be last.
- A common bug: a catch-all chain declared first, so your `/api/**` chain never runs. Symptom: your JWT filter appears not to exist.

### Rules for `authorizeHttpRequests`

**Rules are evaluated in order, first match wins.** So specific before general:
```java
// ❌ WRONG — anyRequest() matches everything, so the admin rule is dead code
.anyRequest().authenticated()
.requestMatchers("/api/v1/admin/**").hasRole("ADMIN")

// ✅ RIGHT
.requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
.anyRequest().authenticated()
```

**Always end with `.anyRequest().authenticated()` or `.denyAll()`.** Deny-by-default means a new endpoint added next year is protected until someone deliberately opens it. **Allow-by-default means a forgotten rule is a data breach.** This is the single most important line in a security config.

### The deprecated API — recognize it

```java
// ❌ Spring Security 5 style, REMOVED in 6.x
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Override protected void configure(HttpSecurity http) { ... }
}
```
`WebSecurityConfigurerAdapter` was deprecated in 5.7 and **removed in Spring Security 6** (Boot 3). The component-based `SecurityFilterChain` bean style above is current. Also: `antMatchers`/`mvcMatchers` → `requestMatchers`; `authorizeRequests` → `authorizeHttpRequests`. **Seeing `WebSecurityConfigurerAdapter` in a tutorial tells you it's pre-2022** — a useful dating heuristic, and a likely interview question.

### Security headers you get, and why

| Header | Protects against |
|---|---|
| `X-Content-Type-Options: nosniff` | MIME sniffing attacks |
| `X-Frame-Options: DENY` | Clickjacking |
| `Strict-Transport-Security` | Protocol downgrade / SSL stripping |
| `Cache-Control: no-store` (on secured responses) | Sensitive data in a shared browser cache |
| `Content-Security-Policy` | XSS (**you must set this — it's not on by default**) |
| `Referrer-Policy` | Leaking URLs (with tokens) to third parties |

Spring Security sets most of these automatically. **CSP is the notable exception** and the most powerful one — configure it.

---

## 7. CSRF — when you need it and when you don't

**The attack:** you're logged into `vigil.io`. You visit `evil.com`, which contains:
```html
<form action="https://vigil.io/api/v1/monitors/123" method="POST">
  <input type="hidden" name="_method" value="DELETE">
</form>
<script>document.forms[0].submit()</script>
```
The browser **automatically attaches your `vigil.io` cookies**. The request is authenticated. Your monitor is deleted. You never saw it happen.

**The defence:** require an unguessable token that `evil.com` cannot read (because the Same-Origin Policy stops it reading your pages) but your own pages can include.

**When you need CSRF protection:**
- ✅ **Cookie-based** or session-based authentication with browser clients
- ✅ Server-rendered forms (Vigil's status page admin, if it uses sessions)

**When you can disable it:**
- ✅ **Stateless APIs using the `Authorization: Bearer` header.** The reason is precise: **browsers don't attach `Authorization` headers automatically.** The attacker's forged request carries no token, so it's simply unauthenticated. The vulnerability CSRF addresses doesn't exist.
- ✅ Non-browser clients only (mobile, service-to-service)

**⚠️ The critical caveat: "we use JWT" is NOT sufficient justification to disable CSRF.** If you store that JWT in a **cookie**, the browser sends it automatically and **you are fully CSRF-vulnerable again.** The protection comes from the *transport mechanism* (a header you must explicitly set), not from the token format.

**Stating that distinction is one of the strongest security answers you can give**, because a very large number of developers disable CSRF while storing tokens in cookies and believe they're safe.

If you do use cookies (which has real XSS advantages — see Lesson 26), keep CSRF on:
```java
.csrf(csrf -> csrf
    .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())   // for SPA double-submit
    .csrfTokenRequestHandler(new SpaCsrfTokenRequestHandler()))
```
And set `SameSite=Lax` or `Strict` on the auth cookie — which mitigates most CSRF by itself, though it's defence in depth rather than a replacement.

---

## 8. Build this — Vigil's authentication

**1. Add security and observe the default.**
```xml
<dependency><groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-security</artifactId></dependency>
```
Restart with **nothing else changed**. Every endpoint now requires auth, and the log prints a generated password:
```
Using generated security password: 8e4f2a1b-...
```
Try `curl localhost:8080/api/v1/monitors` → 401. Then `curl -u user:<password> ...` → 200. **One dependency, and your app is locked down. That's `@ConditionalOnMissingBean` at work** (Lesson 11) — `SecurityAutoConfiguration` provides a default only because you haven't.

**2. Print the filter chain.** Set `logging.level.org.springframework.security=DEBUG` and restart. **Read the printed chain and match it against the §1 diagram.** Find `ExceptionTranslationFilter` and `AuthorizationFilter` and note their positions. Do this properly — it's the foundation of every future debugging session.

**3. Real users from the database.** Implement `User`/`Membership` entities, `VigilUserDetails`, and `VigilUserDetailsService`. Add the delegating `PasswordEncoder`. Seed two users (one `ROLE_USER`, one `ROLE_ADMIN`) with hashed passwords via an `ApplicationRunner`.

**4. Configure the chain** from §6: public `/api/v1/auth/**` and `/api/v1/status/**`, `hasRole("ADMIN")` on `/api/v1/admin/**`, `anyRequest().authenticated()`.

**5. ⭐ Reproduce the `ROLE_` prefix bug.** Store authorities as `"ADMIN"` (no prefix) and use `.hasRole("ADMIN")`. **Get a 403 as an admin user.** Read the DEBUG log — you'll see it looking for `ROLE_ADMIN`. Then fix it two ways (add the prefix when building authorities; or switch to `hasAuthority`). **This is the most common Spring Security bug; manufacture it once and you'll diagnose it instantly forever.**

**6. Reproduce the CSRF 403.** Leave CSRF enabled and POST to a protected endpoint with curl. **Get a 403 that looks like an authorization failure.** Read the DEBUG log to see `CsrfFilter` reject it. Then disable CSRF and confirm the POST works. **Now you'll never spend an hour on this.**

**7. Custom 401/403 bodies.** Implement `AuthenticationEntryPoint` and `AccessDeniedHandler` returning `ProblemDetail` JSON consistent with Lesson 16's error format. Verify both. **Note that these are the only error paths that never reach your `@ControllerAdvice`** — because they happen in the filter chain, before `DispatcherServlet`. That's a genuinely non-obvious detail worth knowing.

**8. Prove `SecurityContextHolder` doesn't cross threads.**
```java
@GetMapping("/who")
public String who() { return SecurityContextHolder.getContext().getAuthentication().getName(); }

@Async
public void asyncWho() {
    var auth = SecurityContextHolder.getContext().getAuthentication();
    log.info("async auth: {}", auth);       // null!
}
```
Then fix it with `DelegatingSecurityContextTaskExecutor` and confirm the identity propagates. **Same ThreadLocal lesson, third context** (after MDC and request scope).

**9. Verify password hashing.** Print a BCrypt hash of `"password"` twice — **different values**, because of the per-password salt. Then time `passwordEncoder.encode(...)` at strength 10 vs 12 and note the doubling. Decide what you'd use in production and why.

**10. Deny-by-default, proven.** Add a brand-new endpoint with no security rule. Confirm it requires authentication because of `anyRequest().authenticated()`. Then change that line to `permitAll()` and confirm the new endpoint is now **publicly exposed by accident.** That contrast is why deny-by-default is non-negotiable.

---

## 9. Interview traps

**Q1. How does Spring Security work?** *(the flagship question)*
It's a single servlet filter (`DelegatingFilterProxy` → `FilterChainProxy`) that selects one `SecurityFilterChain` per request and runs an ordered list of focused filters: context loading, headers, CORS, CSRF, then authentication filters that build an `Authentication` and place it in the `SecurityContextHolder`, then `ExceptionTranslationFilter`, and finally `AuthorizationFilter`. All of this happens **before** `DispatcherServlet`, so a controller only executes for authenticated, authorized requests.

**Q2. What's `DelegatingFilterProxy` for?**
The servlet container instantiates filters and knows nothing about Spring. `DelegatingFilterProxy` is a container-registered filter that looks up the Spring bean `springSecurityFilterChain` per request and delegates — bridging the servlet lifecycle and the Spring context, so security filters can be beans with injected dependencies.

**Q3. Explain the authentication flow.**
An authentication filter builds an unauthenticated token → `AuthenticationManager` (`ProviderManager`) picks a supporting `AuthenticationProvider` → `DaoAuthenticationProvider` calls `UserDetailsService.loadUserByUsername`, compares the password with `PasswordEncoder.matches`, and runs account checks → an authenticated `Authentication` with authorities is stored in the `SecurityContextHolder`.

**Q4. 401 vs 403, and how do you debug each?**
401 = not authenticated (missing/invalid/expired credentials, or the auth filter isn't in the chain). 403 = authenticated but not permitted. `ExceptionTranslationFilter` makes the distinction: `AuthenticationException` → 401, `AccessDeniedException` → 403 (or 401 if the user is anonymous). For a 403, check the **`ROLE_` prefix** and whether **CSRF** is blocking a state-changing request. Debug with `logging.level.org.springframework.security=DEBUG`.

**Q5. `hasRole` vs `hasAuthority`?**
`hasRole("ADMIN")` checks for the authority `ROLE_ADMIN` — it adds the prefix. `hasAuthority("ADMIN")` matches literally. Mixing the two produces a 403 for a user who obviously has the role — the most common Spring Security bug.

**Q6. What is CSRF, and when can you disable it?**
An attacker's site triggers a state-changing request to your app; the browser attaches cookies automatically, so it's authenticated. The defence is an unguessable token the attacker can't read. You may disable it for stateless APIs using `Authorization: Bearer`, because browsers don't attach that header automatically. **But if you store the token in a cookie, you're CSRF-vulnerable again** — the protection comes from the transport, not the token format.

**Q7. How should you store passwords?**
A delegating `PasswordEncoder` with BCrypt (strength 10–12) or Argon2id. It prefixes the algorithm into the hash so you can verify legacy hashes while writing stronger ones — algorithm migration without password resets. Never a fast hash (MD5/SHA-*): they're designed for speed, so a GPU tries billions of guesses per second. Password hashes must be deliberately slow, memory-hard, and per-password salted.

**Q8. Why doesn't `SecurityContextHolder` work in an `@Async` method?**
It's `ThreadLocal`-based and `@Async` runs on a different thread — the same reason MDC and request-scoped beans break. Fix with `DelegatingSecurityContextTaskExecutor` (not `MODE_INHERITABLETHREADLOCAL`, which leaks the first request's context into a pooled thread permanently).

**Q9. Should your `User` entity implement `UserDetails`?**
No. It couples your domain model to the security framework, and a lazily-loaded entity parked in the `SecurityContext` becomes detached — so touching an association throws `LazyInitializationException` from inside a filter. Wrap the entity in a small immutable `UserDetails` built at authentication time.

**Q10. Why must `.anyRequest().authenticated()` come last?**
Rules are evaluated in order and the first match wins, so a catch-all placed first makes every later rule dead code. Ending with `authenticated()` or `denyAll()` gives **deny-by-default**, so an endpoint added later is protected until someone deliberately opens it.

**Q11. What replaced `WebSecurityConfigurerAdapter`?**
`SecurityFilterChain` beans configured via `HttpSecurity` (component-based). `WebSecurityConfigurerAdapter` was deprecated in Spring Security 5.7 and **removed in 6.0** (Boot 3). Also `antMatchers`→`requestMatchers`, `authorizeRequests`→`authorizeHttpRequests`.

**Q12. How do you prevent user enumeration on login?**
Return an identical response for "no such user" and "wrong password." Spring's `DaoAuthenticationProvider` does this by default: it converts `UsernameNotFoundException` to `BadCredentialsException` (`hideUserNotFoundExceptions=true`) **and** performs a dummy password hash when the user is absent, so response timing doesn't leak the answer either.

**Q13. How do you handle multiple security configurations, e.g. an API and server-rendered pages?**
Multiple `SecurityFilterChain` beans, each with a `securityMatcher`, ordered with `@Order` from most specific to least. `FilterChainProxy` runs the **first** matching chain only — so a catch-all chain declared first silently disables the others.

---

## 10. Retention check

1. Draw the filter chain from memory, naming at least six filters in order.
2. Where exactly is the 401-vs-403 decision made, and what determines it?
3. Give the two most common causes of an unexpected 403.
4. Explain precisely why a Bearer-token API can disable CSRF, and the one condition that invalidates that reasoning.
5. Why is SHA-256 unsuitable for passwords? What are the two properties a password hash needs?
6. Why shouldn't your `User` entity implement `UserDetails`?

---

**Next:** [Lesson 26 — JWT & stateless authentication](26-jwt-and-stateless-auth.md) — access and refresh tokens, rotation, revocation, and the honest trade-offs nobody mentions.

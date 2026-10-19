# Lesson 27 — OAuth2, OIDC & social login

> **Why this lesson exists:** OAuth2 is genuinely confusing because its name says "auth" but it's an **authorization delegation** protocol, and everyone uses it for authentication (which is what OIDC actually is). Getting this distinction right, and being able to draw the authorization code flow with PKCE, puts you well ahead of most candidates. It's also how "Sign in with Google" works, which Vigil needs.

**Time:** ~70 minutes · **Prereq:** Lessons 25–26

---

## 1. The idea in one sentence

> **OAuth2 is about permissions and OIDC is about identity — "Sign in with Google" is OIDC riding on OAuth2's authorization code flow — and almost every confusion and every attack in this area comes from using an access token to answer a question only an ID token can answer.**

The access token says what its bearer may do, for an audience that isn't you. The ID token says who the user is, addressed to you, with an `aud` and a `nonce` you are required to check. Keep those two apart and the protocol stops being mysterious.

---

## 2. The distinction that clears up all the confusion

| | Answers | Protocol | Gives you |
|---|---|---|---|
| **OAuth 2.0** | *"Is this app allowed to do X on behalf of the user?"* | **Authorization** delegation | An **access token** for calling APIs |
| **OIDC** (OpenID Connect) | *"Who is this user?"* | **Authentication** (a layer *on top of* OAuth2) | An **ID token** (a JWT describing the user) |

**OAuth2 was never designed for login.** It was designed so that a photo-printing site could access your Google Photos **without you giving it your Google password.** That's the actual problem: delegated access with scoped permissions and no credential sharing.

Then everyone started using it for login anyway, doing it inconsistently and often insecurely (an access token proves you can *call an API*, not who you *are* — see §6). So **OIDC was standardized in 2014** to add a proper identity layer: an ID token with standard claims, a `/userinfo` endpoint, and discovery.

> **The soundbite:** *"OAuth2 is about permissions, OIDC is about identity. 'Sign in with Google' is OIDC, which is built on OAuth2's authorization code flow."* That single sentence resolves the confusion most candidates display.

### The four roles

```
  Resource Owner        the user (you)
  Client               the app requesting access (Vigil)
  Authorization Server issues tokens (Google, GitHub, Keycloak, Auth0)
  Resource Server      hosts the protected API (Google Photos API, or Vigil's own API)
```

Note that **Vigil is a client** when it offers "Sign in with Google," and a **resource server** when it validates its own JWTs (Lesson 26). Same application, two roles — a distinction worth being precise about.

---

## 3. The authorization code flow with PKCE

**This is *the* flow. Learn to draw it.** It's what you use for web apps, SPAs, and mobile apps — essentially everything now.

```
 User          Browser              Vigil (client)         Google (auth server)
  │              │                        │                        │
  │ click        │                        │                        │
  │ "Sign in ───▶│                        │                        │
  │  with        │─── GET /oauth2/        │                        │
  │  Google"     │    authorization/      │                        │
  │              │    google ────────────▶│                        │
  │              │                        │ generate:              │
  │              │                        │  · state (CSRF)        │
  │              │                        │  · code_verifier (rnd) │
  │              │                        │  · code_challenge      │
  │              │                        │    = S256(verifier)    │
  │              │◀── 302 redirect to ────│                        │
  │              │    accounts.google.com/o/oauth2/v2/auth?        │
  │              │      client_id=...&redirect_uri=...             │
  │              │      &response_type=code&scope=openid email     │
  │              │      &state=xyz&code_challenge=abc              │
  │              │      &code_challenge_method=S256                │
  │              │                                                 │
  │              │────────────────────────────────────────────────▶│
  │◀─── Google's login page + consent screen ──────────────────────│
  │ enter password, approve ───────────────────────────────────────▶│
  │              │                                                 │
  │              │◀── 302 to Vigil's redirect_uri?code=AUTH_CODE&state=xyz
  │              │                        │                        │
  │              │─── GET /login/oauth2/  │                        │
  │              │    code/google?code=..▶│                        │
  │              │                        │ ★ verify state matches │
  │              │                        │                        │
  │              │                        │─── POST /token ───────▶│
  │              │                        │    code=AUTH_CODE      │
  │              │                        │    client_id + SECRET  │
  │              │                        │    code_verifier ★     │
  │              │                        │                        │ verify
  │              │                        │                        │ S256(verifier)
  │              │                        │                        │ == challenge?
  │              │                        │◀── access_token +──────│
  │              │                        │    id_token (JWT) +    │
  │              │                        │    refresh_token       │
  │              │                        │                        │
  │              │                        │ validate id_token:     │
  │              │                        │  signature (JWKS),     │
  │              │                        │  iss, aud, exp, nonce  │
  │              │                        │                        │
  │              │                        │ find-or-create the     │
  │              │                        │ Vigil user, issue      │
  │              │                        │ VIGIL's OWN tokens     │
  │              │◀── Set-Cookie / redirect to the app ────────────│
```

### Why each security element exists

**The authorization code (not the token) comes back through the browser.** The code is single-use, short-lived (~60s), and useless without the client secret. If it leaks in a browser history entry, a `Referer` header, or a server log, the damage is bounded. **Returning a token directly through the browser — the old implicit flow — put a long-lived credential in the URL bar. That's why the implicit flow is deprecated.**

**`state` prevents CSRF on the callback.** Without it, an attacker could send a victim a crafted callback URL containing the *attacker's* authorization code, and the victim's Vigil session would be linked to the attacker's Google account (or vice versa — "login CSRF"). Spring Security generates and validates `state` automatically.

**`nonce` (OIDC) prevents ID token replay.** The client sends a random `nonce` in the authorization request; it must appear in the ID token. So an ID token captured from another session can't be replayed.

**PKCE (Proof Key for Code Exchange, RFC 7636) — the important one:**

```
1. Client generates a random code_verifier (43–128 chars)
2. Sends code_challenge = BASE64URL(SHA256(code_verifier)) in the authorization request
3. Sends the raw code_verifier when exchanging the code for a token
4. The auth server checks SHA256(verifier) == the challenge it stored
```

**The attack it prevents:** on mobile, redirect URIs use custom schemes (`myapp://callback`) which a *malicious app on the same device* can also register. It intercepts the authorization code — and without PKCE, a public client has no secret, so the stolen code is enough to get tokens. With PKCE, the attacker has the code but not the verifier (which never left the legitimate app), so the exchange fails.

**PKCE was created for public clients (mobile, SPA) but is now recommended for *all* clients**, including confidential ones with secrets — belt and braces. OAuth 2.1 makes it mandatory. **Knowing that PKCE is no longer "just for mobile" is a current-knowledge signal.**

---

## 4. The other flows — know which are dead

| Flow | Use | Status |
|---|---|---|
| **Authorization Code + PKCE** | Web apps, SPAs, mobile | ✅ **The answer for user-facing auth** |
| **Client Credentials** | Service-to-service, no user | ✅ **The answer for machine-to-machine** |
| **Device Code** | TVs, CLIs, input-constrained devices | ✅ Valid (`gh auth login` uses it) |
| **Refresh Token** | Getting a new access token | ✅ Valid |
| ~~Implicit~~ | SPAs (historically) | ❌ **Deprecated** — token in the URL fragment: leaks via history, Referer, logs; no refresh token |
| ~~Resource Owner Password Credentials~~ | Trusted first-party apps | ❌ **Deprecated** — the app handles the user's password, defeating OAuth's entire purpose; breaks MFA and federated login |

**Client credentials — for Vigil's own workers:**
```
POST /oauth2/token
grant_type=client_credentials&client_id=vigil-checker&client_secret=...&scope=checks:write
```
No user, no browser, no redirect. **This is how Vigil's check-worker service authenticates to the API** when the services are split (Module 10).

**"Which flows are deprecated and why?" is a common question**, and being able to explain that ROPC hands the user's password to the app — defeating the point of OAuth and breaking MFA — is the answer that shows understanding rather than memorization.

---

## 5. The ID token

```json
{
  "iss": "https://accounts.google.com",
  "sub": "110169484474386276334",       // ★ STABLE unique user id at this provider
  "aud": "your-client-id.apps.googleusercontent.com",
  "exp": 1753838400,
  "iat": 1753834800,
  "nonce": "n-0S6_WzA2Mj",
  "email": "user@example.com",
  "email_verified": true,               // ★ CHECK THIS
  "name": "Pat Doe",
  "picture": "https://lh3.googleusercontent.com/..."
}
```

**Two critical rules for using an ID token, and they're both about identity correctness:**

**1. Key users on `iss` + `sub`, never on `email`.**
```java
// ❌ WRONG
User user = userRepository.findByEmail(idToken.getEmail()).orElseGet(() -> createUser(...));
```
Emails change. Emails are reassigned (a corporate Google Workspace account can be deleted and recreated for a different person). And **on some providers, email isn't verified** — which makes it an account-takeover vector (§6). The `(iss, sub)` pair is the stable, provider-guaranteed identifier.

```java
// ✅ RIGHT
@Entity
@Table(uniqueConstraints = @UniqueConstraint(columnNames = {"provider", "provider_user_id"}))
public class FederatedIdentity {
    private String provider;              // "google"
    private String providerUserId;        // the `sub` claim
    @ManyToOne private User user;
}
```
This also lets one Vigil user link multiple providers (Google *and* GitHub), which is what real products do.

**2. Check `email_verified` before trusting the email for anything.**

**The account takeover this prevents:** suppose you match users by email. An attacker registers `victim@company.com` at a provider that doesn't verify emails, signs into your app with it, and is handed the victim's existing account. **This has been a real vulnerability in real products.** Trust an email only if the provider asserts it's verified — and even then, prefer `(iss, sub)` for identity and treat email as a display attribute.

**Being able to describe this attack is one of the strongest things you can say in an OAuth interview**, because it shows you think about identity as an attack surface.

### `id_token` vs `access_token` — the distinction to be precise about

| | ID token | Access token |
|---|---|---|
| Audience | **The client** (your app) | **The resource server** (the API) |
| Purpose | "Here's who the user is" | "Here's what the bearer may do" |
| Format | Always a JWT with standard claims | Opaque or JWT — **the client must not inspect it** |
| Your app should | Validate and read it | Forward it; never parse it |

**A client must never treat an access token as identity, and never introspect one that isn't addressed to it.** Access tokens can be opaque, their format can change without notice, and their audience is a different party.

---

## 6. Two attacks worth knowing by name

### The confused deputy / access token substitution attack

**The reason OIDC exists.** Suppose you authenticate users by calling a provider's `/userinfo` with an access token and trusting the result:

1. The user signs into `evil.com`, which is also a legitimate client of the same provider.
2. `evil.com` obtains an access token for that user.
3. `evil.com` presents *that token* to **your** app's "log me in with this token" endpoint.
4. Your app calls `/userinfo`, gets a valid identity, and logs the attacker in as that user.

**The token was issued for `evil.com`, not for you — but nothing in an access token forces you to check that.** The ID token fixes it precisely because it has an `aud` claim you are *required* to validate against your own client ID.

**Which yields the rule: authenticate with the ID token (validating `aud` and `nonce`), not by dereferencing an access token.** And never build a "log in with this token" endpoint that accepts a client-supplied token.

### Open redirect via `redirect_uri`

If the authorization server allows loose `redirect_uri` matching, an attacker crafts an authorization request with their own redirect and receives the code. **Defence: register exact redirect URIs — full string match, no wildcards, no path-prefix matching.** This is configured at the provider, and it's the one setting people get lazy about.

---

## 7. Spring Security implementation

### As an OAuth2 client (social login)

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
```

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: [openid, email, profile]
            # redirect-uri defaults to {baseUrl}/login/oauth2/code/{registrationId}
          github:
            client-id: ${GITHUB_CLIENT_ID}
            client-secret: ${GITHUB_CLIENT_SECRET}
            scope: [read:user, user:email]
```

**Google, GitHub, Facebook and Okta have built-in `CommonOAuth2Provider` defaults**, so endpoints, JWKS URIs and user-info attribute names are pre-configured. For anything else:

```yaml
        provider:
          keycloak:
            issuer-uri: https://auth.vigil.io/realms/vigil     # ★ discovery does the rest
```

**`issuer-uri` triggers OIDC Discovery**: Spring fetches `{issuer}/.well-known/openid-configuration` and learns the authorization endpoint, token endpoint, JWKS URI, and supported algorithms automatically. **One line replaces six.** Nice thing to know.

```java
@Bean
SecurityFilterChain webChain(HttpSecurity http) throws Exception {
    return http
        .securityMatcher("/", "/login/**", "/oauth2/**", "/app/**")
        .authorizeHttpRequests(a -> a
            .requestMatchers("/", "/login/**", "/oauth2/**").permitAll()
            .anyRequest().authenticated())
        .oauth2Login(oauth -> oauth
            .loginPage("/login")
            .userInfoEndpoint(u -> u.oidcUserService(vigilOidcUserService))   // ★ your hook
            .successHandler(oauth2SuccessHandler))                            // ★ issue OUR tokens
        .logout(l -> l.logoutSuccessUrl("/"))
        .build();
}
```

**Spring Security handles automatically:** the `/oauth2/authorization/{registrationId}` initiation endpoint, `state` generation and validation, **PKCE** (for public clients, and configurable for confidential ones), the `/login/oauth2/code/{registrationId}` callback, the token exchange, ID token validation including signature/`iss`/`aud`/`exp`/`nonce`, JWKS fetching and caching, and `OAuth2AuthorizedClient` storage for later API calls.

**That list is exactly why you don't hand-roll this.** Every item is a place to introduce a vulnerability.

### The find-or-create hook — where your logic goes

```java
@Service
@RequiredArgsConstructor
public class VigilOidcUserService extends OidcUserService {

    private final UserRepository userRepository;
    private final FederatedIdentityRepository identities;
    private final OrganizationService organizations;

    @Override
    @Transactional
    public OidcUser loadUser(OidcUserRequest request) throws OAuth2AuthenticationException {
        OidcUser oidcUser = super.loadUser(request);       // ★ let Spring do the protocol work

        String provider = request.getClientRegistration().getRegistrationId();
        String providerUserId = oidcUser.getSubject();      // the `sub` claim
        String email = oidcUser.getEmail();
        Boolean emailVerified = oidcUser.getEmailVerified();

        // ★ Never trust an unverified email
        if (!Boolean.TRUE.equals(emailVerified)) {
            throw new OAuth2AuthenticationException(
                    new OAuth2Error("email_not_verified"),
                    "Provider did not verify this email address");
        }

        User user = identities.findByProviderAndProviderUserId(provider, providerUserId)
                .map(FederatedIdentity::getUser)
                .orElseGet(() -> linkOrCreate(provider, providerUserId, email, oidcUser));

        return new VigilOidcUser(oidcUser, user);            // wrap, carrying our own user id
    }

    private User linkOrCreate(String provider, String providerUserId, String email, OidcUser o) {
        // Link to an existing local account ONLY when that account's email is itself verified.
        User user = userRepository.findByEmailIgnoreCase(email)
                .filter(User::isEmailVerified)
                .orElseGet(() -> {
                    User created = userRepository.save(User.fromOidc(email, o.getFullName()));
                    organizations.createDefaultOrganization(created);   // Vigil: every user gets an org
                    return created;
                });
        identities.save(new FederatedIdentity(provider, providerUserId, user));
        return user;
    }
}
```

**The `.filter(User::isEmailVerified)` on the linking path is the subtle one:** linking a federated identity to an existing local account by email is only safe if *both* sides verified that email. Otherwise you've built the takeover from §6 in reverse. **This is exactly the kind of detail that separates a working implementation from a correct one.**

### Bridging OAuth2 login to Vigil's own JWTs

Vigil's API is stateless with its own tokens (Lesson 26). So after a successful social login, we mint **our** tokens:

```java
@Component
@RequiredArgsConstructor
public class OAuth2SuccessHandler implements AuthenticationSuccessHandler {

    private final TokenService tokenService;

    @Override
    public void onAuthenticationSuccess(HttpServletRequest req, HttpServletResponse res,
                                        Authentication auth) throws IOException {
        VigilOidcUser principal = (VigilOidcUser) auth.getPrincipal();
        TokenPair tokens = tokenService.issue(principal.vigilUser(),
                                              principal.defaultOrgId(), null);

        res.addHeader(HttpHeaders.SET_COOKIE, refreshCookie(tokens.refreshToken()).toString());
        res.sendRedirect("/app#access_token=" + tokens.accessToken());
        // ⚠️ In production, prefer a short-lived one-time handoff code over a URL fragment.
    }
}
```

**This "federate in, issue our own tokens" pattern is the standard architecture** and worth naming: the provider authenticates the *human*; your system issues the credential that authorizes access to *your* resources. It means your API has one token format regardless of how the user logged in — Google, GitHub, or password.

### Calling a provider's API afterwards

```java
@GetMapping("/import/github-repos")
public List<RepoResponse> importRepos(
        @RegisteredOAuth2AuthorizedClient("github") OAuth2AuthorizedClient client) {
    return restClient.get()
            .uri("https://api.github.com/user/repos")
            .header("Authorization", "Bearer " + client.getAccessToken().getTokenValue())
            .retrieve().body(new ParameterizedTypeReference<>() {});
}
```
Spring stores the authorized client (with its refresh token) and refreshes it transparently. **A genuinely nice Vigil feature: import monitors from a user's GitHub repositories' deployment URLs.**

---

## 8. Identity providers, and the build-vs-buy question

| Provider | Notes |
|---|---|
| **Keycloak** | Open source, self-hosted, full-featured (OIDC, SAML, federation, MFA, admin UI). ✅ **Great for learning and for on-prem** |
| **Auth0 / Okta** | SaaS, excellent DX, generous free tier, gets expensive at scale |
| **AWS Cognito** | Cheap, integrates with AWS IAM, notoriously awkward DX |
| **Google/GitHub directly** | Fine when you only need social login, no user store of your own |
| **Spring Authorization Server** | Spring's own OAuth2/OIDC **server** — for when you must *be* the provider |

**The build-vs-buy answer that shows judgment:**

> *"I'd use a managed provider or Keycloak rather than building an authorization server. Implementing OAuth2/OIDC correctly means PKCE, state, nonce, JWKS rotation, consent, token introspection, MFA, and account recovery — each a place to introduce a vulnerability, and all of it needing maintenance as specs evolve. I'd build one only if I were selling identity as a product, or had a hard requirement no provider met. **Spring Authorization Server** exists for that case."*

**When you *do* need your own** (and Vigil arguably does, since it issues its own API tokens): note that **issuing simple JWTs for your own API is not the same as being an OAuth2 authorization server.** Vigil does the former, which is much smaller in scope. Being precise about that boundary is a good sign.

---

## 9. Build & break — Google sign-in for Vigil

**1. Register an OAuth client.** Google Cloud Console → Credentials → OAuth client ID → Web application. Authorized redirect URI: `http://localhost:8080/login/oauth2/code/google`. **Note that Google requires an exact match** — the open-redirect defence from §6, enforced by the provider.

**2. Configure the client** with `spring-boot-starter-oauth2-client`, credentials from environment variables. Start the app and visit `/oauth2/authorization/google`. **With almost no code, you have working Google sign-in.**

**3. ⭐ Watch the flow.** Open DevTools → Network, preserve log, and click sign in. Then trace it against the §3 diagram:
- The redirect to Google — **find `state`, `code_challenge`, `code_challenge_method`, and `scope` in the query string**
- Google's consent screen
- The callback to `/login/oauth2/code/google?code=...&state=...`
- Vigil's redirect to the app

**Copy the `code_challenge` value. Then dump the `code_verifier` from Spring's `AuthorizationRequestRepository` and verify that `BASE64URL(SHA256(verifier)) == challenge` yourself.** Doing that computation once makes PKCE concrete rather than a slogan.

**4. Inspect the ID token.** Log the `OidcUser`'s ID token and paste it into jwt.io. Find `iss`, `sub`, `aud`, `nonce`, `email`, `email_verified`. **Confirm `aud` equals your client ID** — that's the confused-deputy defence, visible.

**5. `FederatedIdentity` and find-or-create.** Implement §7's `VigilOidcUserService`, keyed on `(provider, providerUserId)`. Verify a second login reuses the same user. Then manually change the user's email in the database and log in again — **confirm the correct user is still found**, because you keyed on `sub`.

**6. Break `state`.** Intercept the callback and change the `state` parameter. **Read Spring's rejection.** Then reason about what would happen without that check.

**7. Reject unverified emails.** Add the `email_verified` check and throw `OAuth2AuthenticationException`. Simulate an unverified email (stub the service in a test) and confirm login is refused. **Write the test that documents the account-takeover you're preventing** — that test comment is a great thing for a reviewer to find.

**8. Add GitHub.** Note that GitHub is **OAuth2 only, not OIDC** — there's no ID token, so `OAuth2UserService` is used rather than `OidcUserService`, and you must call `/user` and `/user/emails` to get the email (which may be private). **That difference is a very good practical illustration of "OAuth2 ≠ OIDC."**

**9. Bridge to Vigil tokens.** Implement `OAuth2SuccessHandler` issuing your own access/refresh pair. Confirm the resulting access token works against `/api/v1/monitors`. **You now have three login paths — password, Google, GitHub — converging on one token format.**

**10. Optional but excellent: run Keycloak.**
```bash
docker run -p 8081:8080 -e KC_BOOTSTRAP_ADMIN_USERNAME=admin -e KC_BOOTSTRAP_ADMIN_PASSWORD=admin quay.io/keycloak/keycloak:26.0 start-dev
```
Create a realm and client, then configure Vigil with `issuer-uri: http://localhost:8081/realms/vigil`. **Fetch `/.well-known/openid-configuration` in a browser and read it** — you'll see exactly what discovery gives you. Then enable MFA in Keycloak and note that **Vigil needed zero changes** to support it. *That's* the argument for delegating identity, demonstrated.

### Explain out loud (60s)

1. OAuth2 vs OIDC in one sentence, and which one "Sign in with Google" is.
2. The authorization code flow with PKCE, naming what each of `code`, `state`, `nonce` and the verifier prevents.
3. The confused-deputy attack, and the single claim that stops it.
4. Why keying users on email is an account-takeover vector.
5. Which flows are dead, and the specific reason for each.

---

## 10. Production rules

| Do / Never | Why |
|---|---|
| **Authenticate with the ID token, validating `aud` and `nonce`** | An access token carries no obligation to check who it was issued for, which is exactly the confused-deputy attack |
| **Never dereference a client-supplied access token to log someone in** | A token minted for `evil.com` will happily answer `/userinfo`, and nothing in it stops you accepting the answer |
| **Never let a client inspect an access token** | It may be opaque, its format may change without notice, and its audience is a different party entirely |
| **Key users on `(iss, sub)`, never on email** | Emails change and get reassigned, and matching on them hands an existing account to whoever can register that address |
| **Check `email_verified` before trusting an email for anything** | A provider that doesn't verify addresses plus email-matching is a documented account-takeover path in real products |
| **Use authorization code + PKCE for every user-facing client** | PKCE started as a mobile defence and is now recommended for confidential clients too — OAuth 2.1 makes it mandatory |
| **Use client credentials for service-to-service, never a user's token** | Machine callers have no user; borrowing one makes the audit trail lie and ties a background job to someone's account |
| **Never use the implicit or password (ROPC) flows** | Implicit puts a long-lived credential in the URL bar; ROPC hands the user's password to the app, defeating OAuth and breaking MFA |
| **Register exact redirect URIs, never patterns** | A loose `redirect_uri` is an open redirect that leaks authorization codes, which is why providers enforce exact matching |
| **Let Spring generate and validate `state`; never disable it** | Without it an attacker can link a victim's session to the attacker's provider account via a crafted callback |
| **Prefer `issuer-uri` and OIDC Discovery over hand-listing endpoints** | One line fetches the authorization, token and JWKS endpoints and the supported algorithms, and keeps working through provider changes |
| **Remember GitHub is OAuth2, not OIDC** | There is no ID token, so you need `OAuth2UserService` and a `/user/emails` call — and the user's email may be private |

---

## 11. Interview traps

**Q1. OAuth2 vs OIDC?**
OAuth2 is an **authorization** framework: it lets an app act on a user's behalf with scoped permissions, without the user sharing credentials, and yields an **access token** for calling APIs. OIDC is an **authentication** layer on top of it, adding an **ID token** (a JWT with standard identity claims), a `/userinfo` endpoint, and discovery. "Sign in with Google" is OIDC.

**Q2. Walk through the authorization code flow.**
Client redirects to the auth server with `client_id`, `redirect_uri`, `scope`, `state`, and a PKCE `code_challenge` → user authenticates and consents → auth server redirects back with a single-use `code` and the `state` → client verifies `state`, then POSTs the code with its client secret and the `code_verifier` to the token endpoint → receives access token, ID token, and refresh token → validates the ID token's signature, `iss`, `aud`, `exp`, `nonce`.

**Q3. What is PKCE and what attack does it prevent?**
The client generates a random `code_verifier`, sends `SHA256(verifier)` as `code_challenge` up front, then sends the raw verifier when exchanging the code. It prevents authorization code interception — on mobile, a malicious app registering the same custom URI scheme could steal the code, and a public client has no secret to protect the exchange. Now recommended for **all** clients and mandatory in OAuth 2.1.

**Q4. Why is the implicit flow deprecated?**
It returned the access token directly in the URL fragment, exposing a long-lived credential to browser history, `Referer` headers, and logs, with no refresh token and no client authentication. Authorization code + PKCE achieves the same for SPAs while keeping the token out of the URL.

**Q5. Why is the password grant (ROPC) deprecated?**
The application handles the user's actual password, defeating OAuth's core purpose (delegated access without credential sharing). It also breaks MFA, federated login, and any provider-side risk checks.

**Q6. What does `state` protect against?**
CSRF on the callback. Without it, an attacker can deliver a crafted callback containing their own authorization code, linking the victim's session to the attacker's account (or the reverse) — "login CSRF."

**Q7. What does `nonce` protect against?**
ID token replay. The client's random `nonce` must appear in the returned ID token, so a token captured from a different session can't be reused.

**Q8. Why should you never identify users by email from an ID token?**
Emails change and get reassigned, and some providers don't verify them — so matching on email allows account takeover (register the victim's email at a lax provider, sign in, receive their account). Key on `(iss, sub)`, store a `FederatedIdentity` row, and treat email as a display attribute you only trust when `email_verified` is true.

**Q9. `id_token` vs `access_token`?**
The ID token is audienced to **the client** and describes the user; you validate and read it. The access token is audienced to **the resource server** and describes permissions; it may be opaque and the client must not inspect it. Using an access token as proof of identity enables the confused-deputy attack.

**Q10. Describe the confused deputy attack.**
An app that authenticates by dereferencing a client-supplied access token can be fed a token issued to a *different* client. It calls `/userinfo`, gets a valid identity, and logs the attacker in as that user. Access tokens have no binding to your client; ID tokens do, via `aud` — which is precisely why OIDC exists.

**Q11. Which flow for service-to-service calls?**
Client credentials: `grant_type=client_credentials` with a client ID and secret, no user and no browser. Used for background workers and internal APIs.

**Q12. Would you build your own authorization server?**
No, by default. Correct OAuth2/OIDC means PKCE, state, nonce, JWKS rotation, consent, introspection, MFA and recovery — a large, security-critical, evolving surface. Use Keycloak or a managed provider; use **Spring Authorization Server** if you genuinely must be the provider. Note that issuing your own JWTs for your own API is a much smaller job than being an authorization server.

**Q13. GitHub login vs Google login — what differs?**
GitHub is OAuth2 only, with no OIDC ID token, so you use `OAuth2UserService` and call its `/user` and `/user/emails` endpoints (email may be private) rather than reading verified claims from a signed ID token. It's the clearest practical demonstration that OAuth2 alone is not an authentication protocol.

---

## 12. Retention check

1. Give the one-sentence distinction between OAuth2 and OIDC.
2. Draw the authorization code flow, naming `state`, `nonce`, and PKCE and what each prevents.
3. Why does the authorization *code* come back through the browser rather than the token?
4. Describe the email-based account takeover and the two defences.
5. Why must a client never treat an access token as identity?
6. What does `issuer-uri` alone give you, and how?

---

**Next:** [Lesson 28 — Authorization & hardening](28-authorization-and-hardening.md) — roles vs permissions, multi-tenant isolation, method security, and the OWASP issues that actually appear in Spring apps.

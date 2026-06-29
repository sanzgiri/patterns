# Federated Identity (Single Sign-On, OAuth 2.0, OIDC, SAML)

**Aliases:** Federated Identity, Single Sign-On (SSO), Identity Federation, External Authentication, OAuth, OpenID Connect (OIDC), SAML
**Category:** Security / Architecture
**Sources:**
[Microsoft Azure — Federated Identity pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/federated-identity) ·
[RFC 6749 — OAuth 2.0 Authorization Framework](https://datatracker.ietf.org/doc/html/rfc6749) ·
[RFC 7636 — Proof Key for Code Exchange (PKCE)](https://datatracker.ietf.org/doc/html/rfc7636) ·
[OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html) ·
[OAuth 2.0 Security Best Current Practice (RFC 9700)](https://datatracker.ietf.org/doc/html/rfc9700) ·
[SAML 2.0 specification](http://docs.oasis-open.org/security/saml/v2.0/) ·
*OAuth 2 in Action* (Justin Richer, Antonio Sanso — Manning 2017) ·
[Auth0, Okta, Microsoft, Google identity documentation](https://auth0.com/docs/)

---

## Problem

> [!TIP]
> **ELI5.** You're building an app and you need users to log in. The naïve approach: store usernames and passwords yourself, implement password resets, MFA, account lockouts, breach detection. That is a *huge* amount of security-critical code that is hard to get right and dangerous if you get it wrong (leaked password DBs make daily headlines). Instead, **federate**: redirect users to a trusted **identity provider** — Google, GitHub, Apple, Microsoft, Okta, your company's Active Directory — which handles authentication. The IdP gives you back a signed token proving who the user is. You never see their password. The IdP's security team has more resources than you do; users get one login across many apps; you focus on your actual product. This is the dominant pattern for nearly all modern web/mobile apps and enterprise SaaS.

The motivating context: every web/mobile app of the 2000s thought it needed its own user database. Each one stored passwords (sometimes hashed properly, often not), implemented its own MFA, reset flows, and breach detection. Most got at least some of it wrong. Result: years of credential leaks, password reuse attacks, account takeovers.

The federated alternative was pioneered by enterprise SSO (Kerberos, then SAML in the 2000s) and consumer SSO ("Sign in with Google/Facebook" in the 2010s) and standardized as **OAuth 2.0** (2012) and **OpenID Connect** (2014). Today, it's near-universal:

- **Consumer apps**: "Sign in with Google/Apple/GitHub" buttons.
- **Enterprise SaaS**: SSO via Okta/Azure AD/Google Workspace/OneLogin.
- **B2B integrations**: OAuth tokens for API access.
- **Mobile apps**: PKCE-based OAuth.
- **Government / regulated**: SAML-based federation (often legacy).

The pattern's benefits — security, user experience, operational simplicity — outweigh the costs (vendor dependency, complexity of the protocols) for nearly all use cases. The result is that "don't roll your own authentication" is now a security commandment.

## How it works

> [!TIP]
> **ELI5.** When the user clicks "Sign in with X," your app redirects them to provider X. Provider X authenticates them (their password, their MFA, all on the IdP's site, never touching your app). Provider X redirects back with a signed token. Your app verifies the signature (it's mathematical proof that the IdP did the authentication) and treats the user as logged in.

### The fundamental shape

The architectural picture, regardless of protocol:

![Federated identity architecture](../diagrams/svg/federated-identity.svg)

Three parties:

- **User** — has accounts at one or more identity providers.
- **Your application** (the **Relying Party** in spec language) — needs to know who the user is, but does not want to manage credentials itself.
- **Identity Provider (IdP)** — runs authentication: passwords, MFA, biometrics, anomaly detection, password resets, account recovery. Returns a token proving authentication.

The flow (generally):
1. User clicks "Log in with X" in your app.
2. App redirects to IdP with a request.
3. IdP authenticates the user (password, MFA, etc.).
4. IdP redirects back to your app with a token (or auth code that exchanges for a token).
5. App verifies the token (cryptographic signature against IdP's public key).
6. App treats user as authenticated.

The crucial property: **your app never sees the password**. It sees only the token, which is verifiably from the IdP.

### The protocols: OAuth 2.0, OIDC, SAML

Three major protocols in widespread use:

**OAuth 2.0** (RFC 6749, 2012) — *authorization* framework. Originally for "let app A access app B's data on user's behalf" (Twitter API access, etc.). Token = `access_token` that grants API permission.

**OpenID Connect (OIDC)** (2014) — *authentication* layer on top of OAuth 2.0. Adds an `id_token` that carries the user's identity. Combines the auth + authorization story. **This is what most consumer/SaaS apps use today.**

**SAML 2.0** (2005) — XML-based, primarily for enterprise SSO. Heavier and older than OAuth/OIDC, but still dominant in enterprise B2B integrations (especially older deployments). Token = signed XML "assertion."

Quick comparison:

| | OAuth 2.0 | OIDC | SAML 2.0 |
|---|---|---|---|
| Purpose | Authorization | Authentication (+ authz) | Authentication (+ authz) |
| Format | JSON, JWT | JSON, JWT (`id_token`) | XML assertion |
| Used for | API access | User login | Enterprise SSO |
| Web/Mobile | Native | Native | Painful |
| Modern default | + OIDC | ✅ Most common | Legacy enterprise |

For a green-field consumer or SaaS app: use OIDC. For enterprise SSO: must support SAML (many corporate customers require it). For pure API access: OAuth 2.0 access tokens.

### The OAuth 2.0 / OIDC authorization code flow (with PKCE)

The dominant flow today — used by web apps, SPAs, and mobile apps. PKCE (Proof Key for Code Exchange) protects against code-interception attacks.

The full sequence:

![OAuth authorization code flow](../diagrams/svg/oauth-flow.svg)

The detailed sequence:

1. User clicks "Sign in with Google."
2. App generates a `code_verifier` (random secret) and `code_challenge` = SHA256(verifier).
3. App redirects user's browser to the IdP's authorize endpoint with `client_id`, `redirect_uri`, `response_type=code`, `scope`, `code_challenge`, `state` (CSRF protection).
4. IdP authenticates the user (password + MFA + risk checks).
5. IdP asks the user to consent ("Allow YourApp to access your email?") on first login.
6. IdP redirects browser back to your app's callback URL with a short-lived `authorization_code`.
7. App's *backend* exchanges the code for tokens via a server-to-server call to the token endpoint, sending the `code_verifier` to prove this is the same flow that started. Receives:
   - `id_token` (JWT proving who the user is — OIDC).
   - `access_token` (lets you call APIs on user's behalf — OAuth).
   - `refresh_token` (long-lived; used to get new access_tokens without re-prompting).
8. App verifies the `id_token`'s signature against the IdP's published public key (JWKS endpoint).
9. App creates its own session for the user (cookie or app token).

The user experience is "click button → enter credentials at IdP → redirected back, logged in." The cryptographic complexity is invisible.

### Why authorization code + PKCE (and what NOT to use)

The protocol has historically allowed several "flows." The 2025 best practice (RFC 9700, OAuth 2.0 Security BCP):

| Flow | Use? | Notes |
|---|---|---|
| **Authorization Code + PKCE** | ✅ All cases | Modern best practice. |
| **Implicit flow** | ❌ Deprecated | Token in URL fragment; insecure. |
| **Resource Owner Password** | ❌ Deprecated | App sees user's password. |
| **Client Credentials** | ✅ Service-to-service | No user; service authenticates. |
| **Device Code** | ✅ TVs / CLI | For input-limited devices. |
| **Refresh Token** | ✅ Always | Use rotating refresh tokens. |

PKCE was originally for mobile/SPAs (no client secret possible) but is now **recommended for all clients** including server-side web apps. It's a small addition with significant security benefit.

### Tokens: what's in them

**ID Token (OIDC)** — a signed JWT with claims:

```json
{
  "iss": "https://accounts.google.com",       // who issued it
  "sub": "user-12345",                         // unique user ID at this IdP
  "aud": "your-app-client-id",                 // intended audience (your app)
  "exp": 1709999999,                           // expiry (Unix timestamp)
  "iat": 1709996399,                           // issued at
  "email": "alice@example.com",
  "email_verified": true,
  "name": "Alice",
  "picture": "https://..."
}
.SIGNATURE  // RS256 / ES256 against IdP's public key
```

Your app verifies: signature (against JWKS), `iss` matches expected IdP, `aud` matches your client ID, `exp` is in future, optionally `nonce` matches what you sent.

**Access Token (OAuth 2.0)** — opaque string or JWT that grants access to APIs:

- Used as `Authorization: Bearer <token>` header.
- Short-lived (typically minutes to an hour).
- Carries scopes ("read:email," "write:posts").
- API verifies it (introspection endpoint or JWT signature).

**Refresh Token** — long-lived (days to months); used to obtain new access tokens without user re-prompt; should be stored very securely; should rotate on each use (RFC 6749 BCP).

### Common patterns built on this

**Single Sign-On (SSO)**: one user authentication to the IdP allows access to many apps. Critical for enterprise; users hate per-app passwords.

**Social Login**: "Sign in with Google/Apple/GitHub" — uses OIDC against consumer IdPs.

**SAML SSO**: enterprise IT requires SAML for connecting to internal directories (Active Directory, etc.).

**Just-in-time provisioning**: first login creates user record; no separate sign-up.

**Identity federation chains**: your app trusts Okta; Okta trusts the customer's Active Directory; user logs in once at AD, gets to your app. Common in enterprise B2B.

**Step-up authentication**: user is logged in for general use, but for sensitive actions (transfer money, change password) the IdP is asked to re-authenticate with stronger factors.

**Token exchange (RFC 8693)**: convert one token for another (impersonation, delegation, on-behalf-of).

### IdP options

| Type | Examples |
|---|---|
| **Consumer IdPs** | Google, Apple, Microsoft (personal), GitHub, Facebook, Twitter/X |
| **Enterprise IdPs (cloud)** | Okta, Auth0 (now part of Okta), Microsoft Entra ID (formerly Azure AD), Google Workspace, OneLogin, Ping Identity, JumpCloud |
| **On-prem / hybrid IdPs** | Active Directory + ADFS, Keycloak, FreeIPA, Authelia, Authentik |
| **CIAM (Customer Identity)** | Auth0, Stytch, FusionAuth, AWS Cognito, Azure AD B2C, Firebase Auth, Supabase Auth |
| **Specialized** | LoginID, Magic Link, WorkOS (SSO-as-a-service for SaaS) |

Most apps use a CIAM provider (Auth0, Cognito, Stytch, etc.) for consumer auth and add SAML/OIDC support for enterprise customers.

### Security considerations

This is security-critical code; getting it wrong is bad. Common pitfalls:

- **Trusting unsigned tokens**: always verify the signature.
- **Wrong `aud` validation**: a token for app B accepted by app A.
- **Token leakage in URLs / logs / referrer headers**: never log tokens; use short-lived ones.
- **Insecure storage of refresh tokens**: encrypt at rest; rotate.
- **No PKCE on mobile/SPA**: code interception.
- **No state parameter**: vulnerable to CSRF on callback.
- **Implicit flow**: don't.
- **Insecure client secrets**: never ship secrets in public mobile/SPA code; use PKCE.
- **Phishing-resistant MFA missing**: hardware keys (FIDO2 / WebAuthn / passkeys) are the gold standard; SMS-OTP is the worst common option.
- **Session fixation / re-use**: regenerate session ID on login.
- **CSRF protection**: use `state` and `nonce` properly.
- **Open redirect**: validate `redirect_uri` strictly.

OWASP has extensive guidance; OAuth 2.0 Security BCP (RFC 9700) is required reading for anyone implementing this.

### Authentication vs authorization (don't confuse them)

Two often-conflated concepts:

- **Authentication (AuthN)**: *who is this user?* → IdP's job; produces identity token.
- **Authorization (AuthZ)**: *what can this user do?* → your app's job; uses identity + roles + policies.

OIDC = authentication. OAuth 2.0 = authorization (for API access). The pattern of "log in via IdP, get user's identity, your app decides what they can do" is universal.

Modern authorization frameworks (RBAC, ABAC, ReBAC) sit on top of authentication. Tools: Open Policy Agent (OPA), AuthZed (Zanzibar-style), Casbin, Permify.

### Federated vs centralized (own auth)

When *not* to federate:

- **Air-gapped systems** with no internet.
- **Sole-purpose systems** that own all their users (some games, some appliances).
- **Highly-regulated** environments requiring internal-only authentication (some government systems).
- **Distrust of all major IdPs** (rare but exists).

For these, build internal authentication carefully (or use self-hosted IdPs like Keycloak/Authelia). For everything else: federate.

### Trade-offs

Advantages:
- **Security**: outsource hard problem to specialists.
- **No password storage**: can't leak what you don't have.
- **MFA / passkeys / biometrics for free**: IdPs implement.
- **User experience**: one click sign-in; SSO across apps.
- **Account recovery**: handled by IdP.
- **Enterprise SSO**: needed for B2B sales.
- **Standards-based**: portable across IdPs.

Disadvantages:
- **External dependency**: IdP outage affects your login.
- **Protocol complexity**: OAuth/OIDC/SAML are sophisticated.
- **Cost**: enterprise IdPs charge per user.
- **Vendor lock-in**: harder to migrate if you've used proprietary features.
- **Privacy concerns**: IdP knows who logs into your app.
- **Account-linking edge cases**: same user with multiple IdPs.

### Anti-patterns

- **Rolling your own authentication** for a typical app.
- **Storing passwords in your DB** (when federation would work).
- **Trusting tokens without signature verification**.
- **Using deprecated flows** (implicit, password grant).
- **No MFA** (delegate to IdP and require it there).
- **Long-lived access tokens** (use refresh tokens; short access tokens).
- **Putting tokens in URLs / logs / referrer headers**.
- **Not supporting SSO** if you sell to enterprise.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **OAuth 2.0 only** | Authorization for APIs; no identity. |
| **OpenID Connect (OIDC)** | OAuth 2.0 + identity layer. |
| **SAML 2.0** | XML-based; enterprise SSO. |
| **WS-Federation** | Older Microsoft; mostly retired. |
| **Kerberos** | Legacy enterprise (still used in AD). |
| **OAuth Authorization Code + PKCE** | Modern web/mobile flow. |
| **OAuth Client Credentials** | Service-to-service. |
| **OAuth Device Code** | TVs / CLI. |
| **Passwordless / Magic Link** | Email-based authentication. |
| **Passkeys / WebAuthn / FIDO2** | Phishing-resistant; future direction. |
| **Just-in-time provisioning** | Auto-create user on first login. |
| **SCIM** | Cross-domain user provisioning. |
| **mTLS (mutual TLS)** | Certificate-based authentication for services. |
| **Federation chain** | Multi-level trust (your IdP → customer IdP). |
| **CIAM (Customer Identity Mgmt)** | Vendor category for consumer auth. |
| **Workforce IdP** | Vendor category for employee auth. |

## When NOT to use

- **Air-gapped systems** — no IdP reachable.
- **Single-use embedded systems** — overkill.
- **Compliance forbidding external IdPs** — rare.
- **For session management** — federated auth gives you identity, not session; build session on top.

---

## Real-world implementations

| Tool | Type |
|---|---|
| **Auth0** | CIAM + SSO; leading vendor |
| **Okta** | Workforce SSO leader |
| **Microsoft Entra ID (Azure AD)** | Enterprise + CIAM |
| **Google Workspace + Cloud Identity** | Google's IdP |
| **AWS Cognito** | AWS-native CIAM |
| **Firebase Auth** | Mobile-focused CIAM |
| **Stytch** | Modern CIAM; passwordless-first |
| **FusionAuth** | Self-hostable CIAM |
| **Keycloak** | Open-source IdP |
| **Authelia, Authentik** | Open-source self-host |
| **Ping Identity, OneLogin, JumpCloud** | Enterprise IdPs |
| **WorkOS** | SSO-as-a-service (helps SaaS add SSO quickly) |
| **Supabase Auth, Clerk, Magic** | Modern developer-focused |
| **Spring Security, Passport.js, NextAuth** | Client libraries |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **Google** | Sign in with Google (OIDC) — IdP for millions of apps. | ✅ Verified — universally known |
| **Apple** | Sign in with Apple (OIDC) — required on iOS apps offering social login. | ✅ Verified — Apple docs |
| **Microsoft** | Microsoft accounts + Entra ID (OIDC + SAML). | ✅ Verified — Microsoft docs |
| **GitHub** | OAuth IdP for many dev tools; SAML for enterprise. | ✅ Verified — GitHub docs |
| **Facebook/Meta** | Facebook Login (OAuth + OIDC). | ✅ Verified — Meta for Developers |
| **Twitter/X, LinkedIn** | OAuth IdPs. | ✅ Verified |
| **Almost every SaaS** | Both as IdP (own users) and Relying Party (other SSO). | ✅ Industry universal |
| **Enterprise SaaS (Slack, Zoom, Salesforce)** | All support SAML + SCIM for enterprise. | ✅ Verified — vendor docs |
| **Banking / regulated** | Heavy SAML use; some still on Kerberos / AD. | ✅ Industry standard |
| **Workforce SSO** | Okta / Entra ID dominant. | ✅ Industry standard |

---

## Further reading

- *OAuth 2 in Action* (Richer, Sanso — Manning 2017).
- OpenID Connect Core 1.0 specification.
- OAuth 2.0 Security Best Current Practice (RFC 9700) — required reading.
- *API Security in Action* (Madden — Manning 2020).
- Microsoft Identity Platform documentation.
- Auth0 Docs — extensive practical material.
- OWASP Authentication Cheat Sheet.
- *Solving Identity Management in Modern Applications* (Wilson, Hingnikar — Apress).
- FIDO Alliance and WebAuthn / Passkeys documentation.
- *Building Secure and Reliable Systems* (Google SRE security book).

---

*Diagram sources: [`../diagrams/src/federated-identity.d2`](../diagrams/src/federated-identity.d2), [`../diagrams/src/oauth-flow.d2`](../diagrams/src/oauth-flow.d2).*

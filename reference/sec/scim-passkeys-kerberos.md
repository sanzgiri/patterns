# SCIM, Passkeys (FIDO2/WebAuthn), and Kerberos

**Aliases:** SCIM (System for Cross-domain Identity Management), Just-in-Time provisioning, User Provisioning Protocol; FIDO2, WebAuthn, Passkey, CTAP2, Resident Credential, Discoverable Credential, Phishing-Resistant MFA; Kerberos, KDC, TGT, AD Authentication, Heimdal/MIT Kerberos
**Category:** Security / Identity
**Sources:**
[RFC 7644 — SCIM Protocol](https://datatracker.ietf.org/doc/html/rfc7644) ·
[RFC 7643 — SCIM Core Schema](https://datatracker.ietf.org/doc/html/rfc7643) ·
[W3C — Web Authentication Level 3 (WebAuthn)](https://www.w3.org/TR/webauthn-3/) ·
[FIDO Alliance — Passkey specifications](https://fidoalliance.org/passkeys/) ·
[RFC 4120 — The Kerberos Network Authentication Service (V5)](https://datatracker.ietf.org/doc/html/rfc4120) ·
[Microsoft — Active Directory + Kerberos documentation](https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview) ·
[Apple, Google, Microsoft passkeys product documentation (2022 joint announcement)](https://www.apple.com/newsroom/2022/05/apple-google-and-microsoft-commit-to-expanded-support-for-fido-standard/) ·
[NIST SP 800-63B (Authenticator Assurance)](https://pages.nist.gov/800-63-3/sp800-63b.html) ·
*Identity and Data Security for Web Development* (Sullivan & Liu, O'Reilly 2016)

---

## Problem

Three orthogonal identity problems that often confuse people because they're all "identity":

**Problem 1 (SCIM)**: *user-lifecycle drift across many apps.* A typical enterprise has 50+ SaaS apps. New hires need accounts in all of them; terminations need to be revoked everywhere immediately (or you have ex-employees still in Slack a year later — real, documented, expensive). Doing this manually is broken; doing it with per-app scripts is brittle. SCIM standardizes the "push user changes from IdP to apps" REST protocol so it works the same way across vendors.

**Problem 2 (Passkeys / FIDO2)**: *passwords (and even most MFA) are phishable.* Phishing is the #1 root cause of breaches. SMS 2FA is SIM-swappable; TOTP can be relayed in real-time by a phishing site; even push-notification MFA gets bypassed by user fatigue (push-bombing). Need an auth method that is **cryptographically bound to the origin** so a phishing site simply cannot acquire a usable credential, no matter what the user does. Passkeys/FIDO2 deliver this.

**Problem 3 (Kerberos)**: *SSO inside the enterprise perimeter.* Before web SSO existed, enterprises needed a way for one login to grant access to many internal services (file shares, printers, databases, internal apps) without sending the password to every service. Kerberos (1980s, MIT) solved this with cryptographic tickets and became the foundation of Active Directory authentication — still in use at most large enterprises.

These three are **complementary, not competitive**: SCIM handles *who exists*, passkeys handle *how they prove it*, Kerberos and OIDC (covered in [federated identity](federated-identity.md)) handle *how that proof gets accepted by services*. Modern enterprises use all four.

> [!TIP]
> **ELI5 (SCIM).** When your company hires someone, an HR system tells the IdP, and the IdP automatically creates accounts in all 50 SaaS apps using a standard "user CRUD" REST API called SCIM. When the person is fired, one click in HR turns off access everywhere within minutes — instead of someone manually disabling accounts in 50 places and missing some.

> [!TIP]
> **ELI5 (Passkeys).** Your device generates a key pair for each site. The site keeps the public key; the private key never leaves the device. To log in, the site challenges your device, your device signs *only if the site's origin matches what the passkey was registered for*, and you approve with your fingerprint/face. A phishing site can't get a signature because the origin doesn't match. No password to leak. No SMS to intercept. No code to type.

> [!TIP]
> **ELI5 (Kerberos).** You log in once and get a "master ticket" (TGT). When you want to use the file server, you trade the master ticket for a short-lived "file server ticket." You show that to the file server and it lets you in — and never sees your password. The file server can also prove its identity to you (mutual auth). Time has to be in sync across all machines or tickets get rejected. This is how Active Directory authenticates everything inside a Windows-dominated enterprise network.

## How they work

### SCIM: user lifecycle protocol

> [!TIP]
> **ELI5.** SCIM is just a standardized REST API for "create user, update user, deactivate user, create group, add member to group." Every SaaS that wants to be enterprise-ready implements it. IdPs (Okta, Azure AD, Google Workspace, OneLogin) speak it to push user changes outward.

The full picture of all three:

![Identity adjuncts: SCIM, passkeys, Kerberos](../diagrams/svg/identity-adjuncts.svg)

SCIM endpoints (RFC 7644):

```
POST   /scim/v2/Users           # create user
GET    /scim/v2/Users/{id}      # read user
PUT    /scim/v2/Users/{id}      # replace user
PATCH  /scim/v2/Users/{id}      # partial update (e.g., {active: false})
DELETE /scim/v2/Users/{id}      # delete (most apps soft-delete via active=false)

POST   /scim/v2/Groups          # create group
PATCH  /scim/v2/Groups/{id}     # add/remove members

GET    /scim/v2/Schemas         # discover what attributes this app supports
```

The schema is JSON; core attributes are standard (`userName`, `name.givenName`, `emails`, `active`, `groups`), and apps can extend with their own. The IdP polls or is webhook-driven from HR systems (Workday, BambooHR, SAP), then pushes deltas to each SCIM-enabled SaaS.

Why this matters in practice:
- **Onboarding**: new hire shows up on day 1 with all accounts pre-created.
- **Offboarding**: terminations close all access in minutes, not days.
- **Role changes**: promotion from contractor to FTE updates entitlements across all apps.
- **Audit**: one IdP-driven place to query "who has access to what."
- **Compliance**: SOX/SOC 2 evidence that user provisioning is controlled.

**Just-in-Time (JIT) provisioning** is a lighter alternative: instead of pre-pushing user data, the app creates a user on their first SSO login (using SAML/OIDC claims). JIT is easier to set up but worse for offboarding (deprovision still needs SCIM or manual cleanup).

### FIDO2 / WebAuthn / Passkeys: phishing-resistant auth

The terminology stack (often confused):

- **FIDO2**: the FIDO Alliance umbrella for the next-gen authentication standards.
- **WebAuthn**: the W3C browser API (`navigator.credentials.create / get`).
- **CTAP2**: Client to Authenticator Protocol — how browsers talk to external authenticators (USB, NFC, BLE).
- **Passkey**: marketing name for FIDO2 credentials that are *discoverable* (the site doesn't need to know your username first) and often *synced* across your devices via cloud (iCloud Keychain, Google Password Manager, Microsoft Account).

#### Registration flow

1. Site (relying party) sends `PublicKeyCredentialCreationOptions` (challenge, user info, RP ID = origin).
2. Browser invokes platform authenticator (Touch ID, Windows Hello) or external (YubiKey).
3. Authenticator generates a fresh key pair scoped to this RP ID.
4. User authorizes with biometric / PIN (locally; secret never leaves device).
5. Browser returns the public key and a signed attestation to the site.
6. Site stores the public key against the user record.

#### Authentication flow

1. Site sends `PublicKeyCredentialRequestOptions` (challenge, RP ID).
2. Browser asks authenticator to sign the challenge.
3. **Critical**: the authenticator only signs if the calling origin matches the RP ID the key was registered for. This is enforced by the browser/OS — not by the user.
4. User authorizes with biometric / PIN.
5. Browser returns signed challenge to the site.
6. Site verifies signature with stored public key.

#### Why this is phishing-resistant

A phishing site `g1thub.com` cannot get a valid signature because:
- The browser binds the challenge to the actual origin (`g1thub.com`), not what the user typed.
- The authenticator refuses to use the `github.com` key for a different RP ID.
- There is no password the user could be tricked into typing.

This is a *structural* defense, not a "be smarter" defense. It works even if the user is fully tricked.

#### Synced vs device-bound passkeys

- **Synced passkeys** (Apple, Google, Microsoft): the private key is end-to-end encrypted to the user's cloud account and roams across their devices. Usability win; trade-off is the cloud account is now a key recovery surface.
- **Device-bound (security key)**: private key never leaves the YubiKey/etc. Highest assurance. Lose the key, lose the account (unless you registered multiple).

Most consumer deployments use synced passkeys; high-security (admin accounts, GitHub root, etc.) use device-bound.

#### Adoption snapshots (2024)

- Apple, Google, Microsoft all support passkeys in their OSes and browsers.
- Major sites: Google, Microsoft, GitHub, Amazon, eBay, PayPal, Best Buy, Cloudflare, Shopify, Adobe — all support passkeys.
- WebAuthn is supported in all major browsers (Chrome, Safari, Firefox, Edge).
- Most enterprise IdPs (Okta, Azure AD, Auth0, Google Workspace) support passkeys / WebAuthn as MFA.

### Kerberos: enterprise SSO inside the perimeter

Kerberos has been around since the mid-1980s (Project Athena at MIT) and is the foundation of Active Directory authentication. It pre-dates the web and was designed for trusted LANs.

#### The actors

- **Client** (user's workstation).
- **Service** (file server, database, web app).
- **KDC (Key Distribution Center)**: trusted third party. Composed of:
  - **AS (Authentication Server)**: issues TGTs.
  - **TGS (Ticket Granting Server)**: issues service tickets in exchange for a TGT.

In Active Directory: every domain controller is a KDC.

#### Flow (simplified)

1. User logs in → AS issues a **TGT** (Ticket-Granting Ticket), encrypted with the user's long-term key (derived from password). User decrypts → proves identity.
2. User wants to access `fileserver` → presents TGT to TGS, requests a service ticket for `fileserver`.
3. TGS issues a **service ticket** for `fileserver`, encrypted with the fileserver's key.
4. User presents service ticket to `fileserver`. The fileserver decrypts (it knows its own key) and trusts the contents.
5. `fileserver` can authenticate back to the user (mutual auth).

Critical: the fileserver **never sees the user's password**. The KDC distributes encrypted tickets; each service only decrypts what's for itself.

#### Properties

- **SSO inside the realm**: one login, many services.
- **Mutual auth**: both sides prove identity.
- **No password on the wire** (after initial login).
- **Time-bound tickets** (usually 10 hours TGT, shorter for service tickets).
- **Works offline-from-internet**: great for corporate LAN.

#### Limitations

- **Clock skew sensitive**: tickets are time-bound; >5 minutes skew typically rejects. Network Time Protocol is mandatory.
- **Cross-realm trust**: federating two Kerberos realms is complex.
- **Not internet-scale**: assumes all services are on the same LAN / trusted network with KDC reachability. For internet, use OIDC ([federated identity](federated-identity.md)).
- **KDC is a single point of trust**: compromise = realm compromise. AD compromise (Golden Ticket attack) is a serious enterprise concern.
- **No native consent / scope**: unlike OAuth, there's no "I delegate scope X to this client" — tickets are typically all-or-nothing.

#### When Kerberos in 2024+

Still widely used:
- Active Directory inside enterprise networks.
- HDFS / Hadoop / Spark internal auth.
- SSH with GSSAPI for "no-password SSH inside a realm."
- Cross-platform internal services using Heimdal / MIT Kerberos.

Increasingly *combined* with:
- OIDC for external/cloud apps (Azure AD bridges).
- Passkeys/WebAuthn for the initial domain login (Windows Hello for Business).
- SCIM for external SaaS provisioning.

## How they fit together

A modern enterprise identity stack:

1. **Workday (HRIS)** is the source of truth for "who is an employee."
2. **Workday → SCIM → IdP**: when someone is hired, the IdP gets a user record.
3. **IdP (Azure AD / Okta) → SCIM → SaaS apps**: user is provisioned in Slack, Zoom, GitHub, Salesforce, Notion, etc.
4. **User logs in to IdP** with **passkey** (phishing-resistant).
5. **IdP issues OIDC tokens** to SaaS apps (web SSO; see [federated identity](federated-identity.md)).
6. **IdP issues Kerberos tickets** for AD-joined internal resources (file shares, internal apps).
7. **Termination**: Workday flips active=false → SCIM cascade revokes everywhere within minutes.

Each piece does one job. None can substitute for another. Together they cover the lifecycle, authentication, and authorization across both cloud and internal services.

### Common confusions

- **"SCIM is auth."** No — SCIM is provisioning. Auth is separate (OIDC, SAML, Kerberos, passkeys).
- **"Passkeys replace SSO."** No — passkeys are how you authenticate to the IdP. The IdP still does SSO to apps.
- **"Kerberos is dead."** No — it's still the foundation of AD-authenticated internal services at virtually every enterprise.
- **"WebAuthn and FIDO2 and passkeys are different things."** They're layers of the same stack: FIDO2 = the umbrella, WebAuthn = the browser API, passkey = the marketing name for the user-friendly form.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **SCIM 1.1** | Older version, less common; use SCIM 2.0 (RFC 7644) |
| **JIT provisioning** | App creates user on first SSO login; complement to SCIM |
| **Cross-Domain Identity Management** | The full name SCIM expands to |
| **U2F (FIDO1)** | Older standard; second-factor only; superseded by FIDO2 |
| **WebAuthn** | The W3C browser API for FIDO2 |
| **CTAP2** | Authenticator-side protocol |
| **Synced passkey** | iCloud Keychain / Google / Microsoft Authenticator |
| **Device-bound credential** | YubiKey, TPM-bound |
| **Resident / discoverable credential** | Site doesn't need username first; user picks from list |
| **MIT Kerberos** | Original reference implementation |
| **Heimdal Kerberos** | BSD-licensed alternative |
| **MS Active Directory Kerberos** | Microsoft's productized version |
| **GSSAPI** | Generic API on top of Kerberos (used by SSH, etc.) |
| **SPNEGO** | Negotiate Kerberos vs NTLM in HTTP headers |
| **[Federated Identity](federated-identity.md)** | OAuth/OIDC/SAML for cloud SSO |
| **[Gatekeeper / Valet Key](gatekeeper-valet-key.md)** | Complementary security patterns |

## When NOT to use

**SCIM**:
- Tiny org with few apps — manual is fine.
- Apps don't support SCIM and won't.
- Use JIT provisioning if pre-provisioning is overkill.

**Passkeys**:
- Apps that absolutely must work without a registered device (rare).
- Headless / non-interactive auth (use OAuth client credentials).
- Where you can't ensure browser+OS support (very old environments).

**Kerberos**:
- Internet-facing apps (use OIDC).
- Stateless microservice mesh (use mTLS + JWT).
- Cross-org federation (use OIDC/SAML).
- Where time sync isn't reliable.

---

## Real-world implementations

| Tool | Type |
|---|---|
| **Okta, Azure AD, Google Workspace, OneLogin, JumpCloud, Auth0** | IdPs with SCIM + WebAuthn + OIDC |
| **Workday, BambooHR, SAP SuccessFactors, Rippling** | HRIS systems that drive SCIM |
| **Slack, Zoom, GitHub, GitLab, Salesforce, Notion, Atlassian** | Apps that consume SCIM |
| **YubiKey, Titan Key, Feitian, SoloKey** | FIDO2 hardware keys |
| **iCloud Keychain, Google Password Manager, Microsoft Authenticator, 1Password, Dashlane, Bitwarden** | Passkey managers |
| **MIT Kerberos, Heimdal, Microsoft AD** | Kerberos implementations |
| **SSSD (Linux)** | Kerberos + LDAP integration |
| **Apache mod_auth_kerb, mod_auth_gssapi** | Web server Kerberos integration |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **Most Fortune 500 enterprises** | Active Directory + Kerberos for internal SSO. | ✅ Industry universal |
| **Apple, Google, Microsoft** | Joint 2022 commitment to passkeys; rolled out across iOS/Android/Windows. | ✅ Verified — joint press release |
| **GitHub** | Passkey support since 2023. | ✅ Verified — GitHub blog |
| **Cloudflare** | All employees use FIDO2 hardware keys; documented phishing-resistant. | ✅ Verified — Cloudflare blog (2022 SMS phishing incident response) |
| **Amazon, eBay, PayPal, Best Buy, Shopify, Adobe** | Passkey support in production. | ✅ Verified — vendor product pages |
| **Okta, Azure AD, Google Workspace** | Drive SCIM to most enterprise SaaS apps. | ✅ Vendor docs |
| **Banks, healthcare** | Heavy Kerberos + AD; cautious passkey adoption underway. | ⚠ Industry pattern |
| **MIT** | Birthplace of Kerberos; still uses it. | ✅ Historical / documented |

---

## Further reading

- RFC 7643, 7644 — SCIM Core Schema & Protocol.
- W3C WebAuthn Level 3 specification.
- FIDO Alliance — Passkey design and adoption guidance.
- RFC 4120 — Kerberos V5.
- *Kerberos: The Definitive Guide* (Garman, O'Reilly 2003).
- *Identity and Data Security for Web Development* (Sullivan & Liu, O'Reilly 2016).
- NIST SP 800-63B — Digital Identity Guidelines.
- Microsoft AD documentation — Kerberos in practice.
- Google Identity passkey guide.
- Apple WWDC sessions on passkeys (2022, 2023, 2024).
- *Modern Authentication* (Diogenes & Souppaya, 2023).

---

*Diagram sources: [`../diagrams/src/identity-adjuncts.d2`](../diagrams/src/identity-adjuncts.d2).*

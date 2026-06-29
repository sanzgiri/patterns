# Gatekeeper & Valet Key

**Aliases:** Gatekeeper, Security Gateway, API Gateway (overlapping), Bastion, Front-End-for-Security; Valet Key, Pre-Signed URL, Shared Access Signature (SAS), Capability Token, Token-Based Access
**Category:** Security
**Sources:**
[Microsoft Azure — Gatekeeper pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/gatekeeper) ·
[Microsoft Azure — Valet Key pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/valet-key) ·
[AWS — Presigned URLs documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html) ·
[Azure Storage Shared Access Signatures](https://learn.microsoft.com/en-us/azure/storage/common/storage-sas-overview) ·
[GCP — Cloud Storage signed URLs](https://cloud.google.com/storage/docs/access-control/signed-urls) ·
[OWASP — Application Security Verification Standard](https://owasp.org/www-project-application-security-verification-standard/) ·
*Building Secure and Reliable Systems* (Adkins et al., O'Reilly 2020)

---

## Problem

Two distinct but related questions that come up constantly:

**Problem 1 (Gatekeeper)**: *How do we expose internal services to the internet without putting their full code on the perimeter?* Business logic services are large, complex, and full of attack surface. Exposing them directly to the open internet means any bug in any service is an internet-exploitable vulnerability.

**Problem 2 (Valet Key)**: *How do we let clients upload/download large files to/from storage without proxying every byte through our app?* The naive "client → app → storage" path doubles bandwidth, ties up app memory and time, and doesn't scale. But just handing clients permanent storage credentials would let them do anything.

> [!TIP]
> **ELI5 (Gatekeeper).** Put a small, hardened, dedicated security service at the edge. Its only job is to validate, authenticate, authorize, and forward — no business logic, no DB access, no secrets. If attackers compromise the gatekeeper, they're still locked out of the real services and data. It's the "moat and drawbridge" model: keep the interesting stuff inside the castle.

> [!TIP]
> **ELI5 (Valet Key).** Like leaving your car with a valet: you give them a key that opens the door and starts the engine but doesn't open the glove compartment or the trunk, and it works only for the next 30 minutes. In cloud terms: your app generates a short-lived, narrowly-scoped credential ("you may PUT to s3://uploads/abc-123.jpg for the next 5 minutes, max 10MB, image/jpeg only") and gives it to the client. The client uploads directly to storage. Storage validates the credential. No proxying through your app.

Both patterns are about **separation of concerns for security**: keep the security-critical, perimeter-facing logic small and well-audited, and grant temporary, scoped permission for specific operations rather than broad, long-lived access.

## How the Gatekeeper works

> [!TIP]
> **ELI5.** A small public-facing service does all the dangerous stuff (parse untrusted input, check auth tokens, enforce rate limits). It then forwards already-cleaned, already-validated, already-authorized requests to the real business services on a trusted network. If the gatekeeper is breached, the attacker is still outside the real systems.

### Structure

The fundamental shape:

![Gatekeeper pattern](../diagrams/svg/gatekeeper-pattern.svg)

The gatekeeper is a dedicated service that sits between the public internet and the trusted internal services. Its responsibilities (and *only* its responsibilities):

- **TLS termination**: decrypt incoming HTTPS.
- **Input validation**: JSON schema check, size limits, content-type, character encoding.
- **Authentication**: validate token signature, check expiry, verify issuer.
- **Authorization**: check that the authenticated principal can perform this action.
- **Rate limiting**: throttle abusive clients (per IP, per user, per endpoint).
- **Block known-bad**: WAF rules, IP reputation, geo-blocking, signature-based attacks (SQLi, XSS).
- **Sanitization / normalization**: strip dangerous bytes, normalize URL encoding, canonicalize.
- **Audit logging**: every request logged for forensic review.
- **Forwarding**: pass cleaned request to the appropriate internal service.

What the gatekeeper does **not** do:

- **No business logic**: that lives in the actual services.
- **No database access**: doesn't talk to the DB directly.
- **No long-lived secrets**: doesn't hold customer data, payment keys, etc.

The principle: if the gatekeeper is exploited, the blast radius is contained. The attacker has compromised a small, narrow service that *doesn't have access to the things they want* (database, customer data, business secrets). Defense in depth means subsequent layers still enforce security too.

### Variants

- **API Gateway** ([page](../comm/api-gateway.md)) — a gatekeeper that also does routing, aggregation, and protocol translation. Most API gateways are gatekeepers + more.
- **Web Application Firewall (WAF)** — a more network-layer gatekeeper (Cloudflare WAF, AWS WAF, Akamai). Often *in front* of an API gateway, providing additional protection.
- **Service mesh sidecar** ([page](../comm/sidecar.md)) — gatekeeper per service rather than at the edge. Validates mTLS, enforces policies.
- **Bastion host** — gatekeeper for SSH / admin access; you SSH to the bastion, then from there to internal hosts. Sometimes called "jump host."
- **Backend-for-Frontend (BFF)** ([page](../comm/bff.md)) — a gateway scoped to a specific client (web, iOS, Android). Often acts as a per-client gatekeeper.

The "gatekeeper" as Microsoft describes it is the *abstract* role; in practice it manifests in many of these specific tools.

### Defense in depth

The gatekeeper is one layer; it must not be the *only* layer:

- **Network**: VPC isolation, security groups, firewalls.
- **Edge**: CDN + WAF.
- **Gateway**: gatekeeper (this pattern).
- **Service mesh**: per-service sidecar (mTLS, policies).
- **Service**: each service still validates auth (in case gateway is bypassed).
- **Data**: encryption at rest, ACLs on tables, query-level authorization.
- **Application logic**: business rules enforced in code.

This is sometimes called "belt and braces" — multiple independent layers, so the failure of any one isn't catastrophic.

### Why bother (vs putting validation in services)?

A natural question: why not let every service handle its own validation/auth? Answers:

- **Smaller, audited security surface**: the gatekeeper's code is tiny vs all services combined.
- **One place to update security policy**: change WAF rules, rate limits, auth integration in one place.
- **Easier to harden one service**: minimal dependencies, fewer privileges, fewer libraries.
- **Reduce service complexity**: business services don't need to deal with adversarial input.
- **Better operational separation**: security team owns the gatekeeper; product teams own services.

The pattern is essentially "single responsibility principle" applied to security.

### Trade-offs (Gatekeeper)

Advantages:
- **Bounded attack surface**: small audited security perimeter.
- **Centralized security policy**: one place to update.
- **Simpler service code**: doesn't need to defend against adversaries.
- **Defense in depth foundation**.

Disadvantages:
- **Additional hop / latency**.
- **Single point of failure** if not made redundant.
- **Encourages "trust internal network" mistake** — services *should* still check auth.
- **Can become a development bottleneck** — every API change touches the gatekeeper.

## How the Valet Key works

> [!TIP]
> **ELI5.** Instead of being the proxy for every upload/download, your app generates a special URL that includes a cryptographic signature. The signature says "this URL is valid for this exact resource, this exact operation, until this exact time." Client uses the URL to talk directly to storage. Storage validates the signature and enforces the limits. Your app stays out of the data path.

### The flow

The architectural picture:

![Valet key pattern](../diagrams/svg/valet-key.svg)

The sequence:

1. Client asks your app: "I want to upload `my-file.jpg` (5 MB)."
2. App **authorizes** the request: is this user allowed? Have they hit their quota? Is the content type permitted?
3. App generates a **presigned URL** via the storage SDK. The URL embeds:
   - The specific object key (`uploads/abc-123.jpg`).
   - The HTTP method (`PUT`).
   - An expiry (5 minutes from now).
   - Optional constraints: max size, content-type, content-MD5, IP range.
   - A signature computed using the app's storage credentials.
4. App returns the URL to the client.
5. Client uploads (or downloads) **directly** to/from storage at that URL.
6. Storage validates the signature, checks expiry, enforces constraints; accepts or rejects.

The cryptography is done by the cloud SDK; your code never handles the signing math.

### Why this matters

For uploads of any non-trivial size:

- **Bandwidth**: client → app → storage doubles your egress. Client → storage halves it. At scale this is massive cost savings.
- **App memory / CPU**: streaming GB-sized uploads through your app servers is expensive and brittle.
- **Latency**: direct upload is faster (one network hop, not two).
- **Reliability**: cloud storage is more reliable than your app; fewer moving parts.
- **Concurrency**: you can handle many concurrent uploads without burning app resources.

For downloads:
- Same bandwidth and resource arguments.
- Plus: clients can get fast CDN/edge delivery (CloudFront, etc.) without proxying.

### Properties a valet key MUST have

- **Short-lived**: minutes, not days. The shorter the better — keys leak.
- **Scoped to one resource**: this exact object key, not the bucket.
- **Scoped to one operation**: PUT only, or GET only — not both.
- **Constrained**: content type, size limit, IP range when possible.
- **Not revocable before expiry**: keep expiry short.
- **Treated as a secret**: don't log full presigned URLs (they contain the signature).
- **Not cacheable / shareable**: any sharing is a security event.

### Real implementations

| Service | Mechanism |
|---|---|
| **AWS S3** | Presigned URLs (canonical example); STS for broader temporary credentials |
| **Azure Blob Storage** | Shared Access Signatures (SAS) — fine-grained, per-blob or container |
| **GCP Cloud Storage** | Signed URLs (V4 signing) |
| **Cloudflare R2** | Presigned URLs (S3-compatible) |
| **Backblaze B2** | Application keys + presigned URLs |
| **MinIO** | Presigned URLs (S3-compatible) |
| **AWS STS AssumeRole** | Short-lived AWS credentials (broader scope than presigned URLs) |
| **OAuth 2.0 scope-limited tokens** | The same pattern at the API layer (RFC 6749) |
| **CDN signed URLs** | CloudFront signed URLs, Akamai tokenized URLs |
| **Database row-level access tokens** | More exotic; Postgres RLS + JWT |

### Common patterns built on valet keys

**Direct upload from browser/mobile**: app generates presigned URL, client uploads to S3 directly. Standard for any media-heavy app (image hosts, video platforms, document management).

**Time-limited downloads**: paid content with expiring URLs (Vimeo, Bandcamp, paywalled documents). Without the valet key, you'd need to proxy or hand out permanent links.

**File-sharing with TTL**: WeTransfer, Dropbox shared links, "share this for 7 days."

**Webhook signatures**: similar idea — short-lived signature that proves authenticity.

**Pre-authorized API access**: Stripe's `ephemeralKey` for mobile, Shopify's storefront access tokens.

### Anti-patterns

- **Long-lived presigned URLs** (days/weeks): they leak, get cached, shared.
- **Overly broad scope**: a presigned URL for `*` instead of one object.
- **No constraints**: presigned URL that accepts any content-type / any size.
- **Logging full presigned URLs**: the signature is in the URL.
- **Embedding presigned URLs in emails forever**: they get forwarded.
- **Using presigned URLs for state changes** with no replay protection.
- **Treating storage credentials as "internal-only safe"**: any leak gives broad access — narrow valet keys are better.
- **Using app's master credentials in client code**: never; that's the failure mode this pattern prevents.

### Trade-offs (Valet Key)

Advantages:
- **Massive bandwidth / cost savings** for large objects.
- **Better client experience**: direct, fast, less proxying.
- **App stays out of data path**: smaller, simpler.
- **Scoped, expiring credentials**: better than long-lived bearer tokens.
- **Native cloud-storage support**: minimal custom code.

Disadvantages:
- **App still must enforce authorization** before issuing the key.
- **Storage-tier validation only**: if storage doesn't enforce a constraint, you can't.
- **Auditing harder**: client → storage is not visible to your app.
- **Limited revocation**: short expiry is your only revoke.
- **Vendor-specific signing**: not strictly portable across cloud providers.

### Auditing direct uploads / downloads

Since your app isn't in the data path, you lose visibility unless you instrument:

- **Storage access logs**: S3 server access logs, CloudTrail data events, Azure Blob diagnostic logs, GCP audit logs.
- **Notification on upload**: S3 event notifications → Lambda / SQS → log to your audit pipeline.
- **Pre-upload tracking**: log the *issuance* of the valet key (this is auditable).

The pattern shifts auditing to the storage tier; design accordingly.

### Combining gatekeeper and valet key

A common combination:
1. Client → Gatekeeper → App: "I want to upload X."
2. App authorizes, generates valet key.
3. App returns valet key to client.
4. Client → Storage (direct, with valet key).
5. Storage event notification → Audit pipeline.

Gatekeeper enforces who can ask; valet key enforces what they can actually do.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **Gatekeeper** | Generic security perimeter. |
| **WAF (Web App Firewall)** | Network-layer gatekeeper. |
| **API Gateway** ([page](../comm/api-gateway.md)) | Gatekeeper + routing + aggregation. |
| **BFF** ([page](../comm/bff.md)) | Per-client gatekeeper. |
| **Service mesh sidecar** ([page](../comm/sidecar.md)) | Per-service gatekeeper. |
| **Bastion host** | SSH-specific gatekeeper. |
| **Valet Key** | Short-lived storage credential. |
| **Presigned URL** | The cloud-storage implementation. |
| **SAS (Shared Access Signature)** | Azure's version. |
| **STS AssumeRole** | AWS short-lived credentials (broader scope). |
| **Capability-based security** | The academic principle behind valet keys. |
| **OAuth scope-limited token** | API-layer valet key. |
| **CDN signed URL** | Valet key for cached content. |
| **Webhook signature** | Sender-signed payload. |
| **[Federated Identity](federated-identity.md)** | Complementary: who's calling? |
| **[Throttling](../res/throttling.md)** | Often enforced by the gatekeeper. |

## When NOT to use

**Gatekeeper:**
- **Internal-only services** with no external exposure.
- **Tiny services** where a separate gatekeeper is overhead.
- **As the only security layer** — must combine with defense-in-depth.

**Valet Key:**
- **Small payloads** — proxying through your app is fine for <1MB JSON.
- **Need strict per-operation auditing in real time** — direct access bypasses your app.
- **Need real-time content inspection** (virus scanning, content moderation) — would need to scan after upload via event notification.
- **No support from the storage backend** — older / self-hosted backends may not support signed URLs.

---

## Real-world implementations

| Tool | Type |
|---|---|
| **Cloudflare** | WAF + gatekeeper at edge |
| **AWS WAF, AWS Shield** | Cloud WAF |
| **Azure Front Door, App Gateway** | Cloud WAF + gateway |
| **GCP Cloud Armor** | Cloud WAF |
| **Akamai, Imperva** | Commercial WAF |
| **Kong, Apigee, AWS API Gateway, Azure API Mgmt** | API gateways with gatekeeper features |
| **Envoy + Istio** | Service mesh gatekeeper |
| **AWS S3 presigned URLs** | Valet key |
| **Azure Blob SAS** | Valet key |
| **GCP Cloud Storage signed URLs** | Valet key |
| **CloudFront signed URLs / cookies** | CDN valet key |
| **Stripe ephemeralKey** | API-layer valet key |
| **Auth0/Okta token services** | OAuth scoped tokens |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **Cloudflare** | WAF + gatekeeper at edge for millions of sites. | ✅ Verified — Cloudflare docs |
| **Netflix, YouTube, Instagram** | Presigned URLs for media upload/download. | ⚠ Universal practice; not always specifically documented |
| **Dropbox, WeTransfer, Box, Google Drive** | Presigned URLs / valet keys for sharing. | ✅ Verified — vendor docs / engineering blogs |
| **AWS** | S3 presigned URLs are the canonical implementation. | ✅ Verified — AWS docs |
| **Microsoft** | Azure SAS is universally used in Azure-based systems. | ✅ Verified — Azure docs |
| **Stripe** | API gateway + ephemeral keys for mobile clients. | ✅ Verified — Stripe docs |
| **Shopify** | Storefront access tokens (OAuth scope-limited). | ✅ Verified — Shopify docs |
| **Most modern SaaS** | Direct upload to S3-class storage via presigned URLs. | ✅ Industry standard |
| **Banks / financial** | Heavy use of API gateways as gatekeepers. | ✅ Universal in regulated industries |

---

## Further reading

- Microsoft Azure — Gatekeeper, Valet Key patterns.
- *Building Secure and Reliable Systems* (Google SRE security book, 2020).
- *API Security in Action* (Madden, Manning 2020).
- OWASP Application Security Verification Standard (ASVS).
- AWS S3 presigned URL guide and security best practices.
- *Capability-Based Computer Systems* (Henry Levy, 1984) — academic background.
- *Designing Web APIs* — chapter on gateway patterns.
- AWS, Azure, GCP cloud security pillars / documentation.
- *Microservices Security in Action* (Siriwardena, Dias — Manning 2020).

---

*Diagram sources: [`../diagrams/src/gatekeeper-pattern.d2`](../diagrams/src/gatekeeper-pattern.d2), [`../diagrams/src/valet-key.d2`](../diagrams/src/valet-key.d2).*

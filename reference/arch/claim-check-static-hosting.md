# Claim Check & Static Content Hosting

**Aliases:** Claim Check, Reference-Based Messaging, Large-Payload Externalization; Static Content Hosting, Static Asset Offload, Origin Storage, CDN Origin
**Category:** Communication / Architecture / Scale
**Sources:**
[Microsoft Azure — Claim-Check pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/claim-check) ·
[Microsoft Azure — Static Content Hosting pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/static-content-hosting) ·
[Enterprise Integration Patterns (Hohpe & Woolf, 2003) — *Store-in-Library* / Claim Check](https://www.enterpriseintegrationpatterns.com/patterns/messaging/StoreInLibrary.html) ·
[AWS — Amazon SQS Extended Client Library for Java (S3 large-payload pattern)](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-s3-messages.html) ·
[Jamstack documentation](https://jamstack.org/) ·
[AWS S3 + CloudFront, GCP Cloud Storage + Cloud CDN, Azure Blob + Azure CDN docs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/website-hosting-custom-domain-walkthrough.html)

---

## Problem

Two distinct "stop putting data in the wrong place" patterns:

**Problem 1 (Claim Check)**: *Large payloads on a message bus break everything.* A 50MB image in a Kafka message exceeds default size limits, causes serialization stalls, multiplies storage cost (every replica, every replay, every dead-letter copy), and forces every consumer to hold the payload even when they don't need it. Message brokers are tuned for many small messages, not few big ones.

**Problem 2 (Static Content Hosting)**: *Serving images, CSS, and JS from your app server is wasteful.* The same 100KB CSS file is fetched a million times per day, each time consuming an app server thread and bandwidth. App servers should handle dynamic responses; static assets belong elsewhere.

Both have the same shape: **don't store/move data through the wrong tier**. Use the cheap, scalable, purpose-built tier (blob storage + CDN) for data that fits its profile, and keep the expensive, complex tier (app servers, message brokers) for what only it can do.

> [!TIP]
> **ELI5 (Claim Check).** Like checking your coat at a restaurant: you don't carry it to your table. The coat check gives you a small token; you hand the token back to get the coat. In messaging: store the large payload in a blob store, put just a reference (the "claim check") in the message. Consumers fetch the payload only if/when they need it.

> [!TIP]
> **ELI5 (Static Content Hosting).** Don't have your app servers serve images and JavaScript. Put those files in cheap object storage (S3, Azure Blob, GCS), put a CDN in front, and let the CDN handle 99% of requests at edge servers near users. Your app servers only handle dynamic content. Cheaper, faster, more scalable.

## How Claim Check works

> [!TIP]
> **ELI5.** Producer uploads the big thing to S3/blob storage, gets back a reference. Producer publishes a small message containing only the reference. Consumers receive small messages quickly; if they actually need the big payload, they fetch from blob storage. Message bus stays fast; broker stays small; replay is cheap; dead letters are tiny.

### The mechanism

The pattern in detail:

![Claim check pattern](../diagrams/svg/claim-check.svg)

The flow:

1. **Producer uploads the payload** to blob storage (S3, Azure Blob, GCS, MinIO). Gets back the storage key.
2. **Producer publishes a small message** to the bus containing:
   - Event type and metadata.
   - Reference to the blob (URI, key, bucket).
   - Hash for integrity check (SHA-256 commonly).
   - Size, content-type.
   - Correlation ID.
   - Any small fields useful for filtering/routing without fetching the blob.
3. **Bus** delivers the small message efficiently. Fanout to multiple consumers is cheap.
4. **Consumer receives the small message**. Decides whether it needs the payload:
   - **Filtering consumer**: only some events are relevant; reject quickly without fetching.
   - **Header-only consumer**: needs only metadata; never fetches.
   - **Full consumer**: fetches the payload from blob storage; processes; acks.
5. **Cleanup**: blob lifecycle policy deletes after retention period, or explicit cleanup after all consumers process.

A typical message:

```json
{
  "event_type": "ImageUploaded",
  "blob_ref": "s3://my-bucket/images/img-abc-123.jpg",
  "size_bytes": 52428800,
  "sha256": "a1b2c3...",
  "content_type": "image/jpeg",
  "uploaded_by": "alice",
  "request_id": "req-7f3a-bc12",
  "uploaded_at": "2024-01-15T10:23:45Z"
}
```

A few hundred bytes instead of 50MB.

### When to use

**Use claim check when**:
- Payload size > typical message size — rule of thumb >256KB; certainly >1MB.
- Message broker has hard size limits (Kafka default 1MB; AWS SQS hard 256KB; Azure Service Bus 256KB-1MB by tier).
- Fanout is wide — N consumers means N× cost without claim check.
- Not every consumer needs full payload — many can filter on metadata.
- Replay is needed — replaying messages with embedded blobs is expensive.
- Long-lived payloads — blob retention differs from message retention.

**Don't use when**:
- Payloads are small — just embed them inline.
- Every consumer always needs full payload immediately — extra fetch hop is wasted.
- Strict atomicity needed — blob and message can technically diverge (rare in practice if you upload-then-publish).
- The blob store is unavailable on consumer side — adds a dependency.

### Variations and gotchas

**Atomicity**: in principle, the blob upload could succeed but the message publish could fail, or vice versa. Mitigations:
- Upload blob first, then publish (orphan blob from failed publish is cheaper than failed message claiming missing blob).
- Use [transactional outbox](../data/outbox.md) for the message publish if the producer has a DB.
- Periodic cleanup of orphan blobs (lifecycle policy).
- Hash check on consumer fetch detects corruption.

**Caching**: many consumers fetching the same blob simultaneously creates a stampede on the blob store. CDN in front of the blob store helps.

**Versioning**: if blobs are mutable, versioned blob IDs (or never-mutate convention) prevent confusion.

**Security**: blob references in messages give read access to anyone reading the message. Either:
- Bucket is internal-net only (consumers within trust boundary).
- Use [valet keys](../sec/gatekeeper-valet-key.md) — message includes a presigned URL with limited TTL.
- Sign or encrypt the reference itself.

**Cost analysis**:
- Without claim check: 50MB × N consumers × replication factor × replay = enormous bus storage cost.
- With claim check: 50MB once in blob (single copy or cheap replicated), small messages in bus.
- Trade-off: blob storage cost + GET request cost vs bus storage savings + reduced bandwidth.

In nearly every case where payloads exceed 100KB and fanout exists, claim check wins on cost and performance.

### AWS SQS Extended Client (the transparent implementation)

AWS published the **SQS Extended Client Library for Java** that does claim check automatically: send a message normally; if >256KB, it transparently uploads to S3 and replaces the body with a reference. Receive normally; library fetches from S3 if needed. The pattern is invisible to application code.

Similar libraries exist for other languages and brokers. Apache Camel has the **Claim Check EIP** built in. Spring Cloud Stream has S3-backed binders.

### Real-world uses

- **Image / video processing pipelines**: upload media → claim check → multiple processors (thumbnailer, transcoder, ML tagger) each fetch from S3.
- **Document processing**: PDF/document analysis pipelines pass references, not bytes.
- **Audit / event-sourcing with large events**: detailed event payloads in S3; small references in event log.
- **Email systems**: email body and attachments in storage; message contains reference (SES, SendGrid backend).
- **Webhooks with bodies**: webhook event log holds reference; bodies in blob.
- **Big-data pipelines**: Kafka messages reference HDFS/S3 paths for big records.

## How Static Content Hosting works

> [!TIP]
> **ELI5.** Build your CSS/JS/images into static files during deploy. Upload them to object storage (S3, Azure Blob, GCS). Put a CDN in front so they're served from edge servers near users. Your app servers don't touch them; they just serve HTML with `<script src="https://cdn.example.com/...">` pointing to the CDN. App servers handle the few dynamic requests; CDN handles the millions of static requests for free or pennies.

### The architecture

The picture:

![Static content hosting](../diagrams/svg/static-content-hosting.svg)

The flow:

1. **Build/deploy pipeline** generates static assets (compiled CSS/JS, optimized images, fonts).
2. Pipeline **uploads** to a blob store (S3, Azure Blob, GCS, R2). Filenames use content hashes for cache-bust (`main.a3f8c9.js`).
3. **CDN** (CloudFront, Cloudflare, Fastly, Akamai, Azure CDN) caches at edge with the blob store as origin.
4. **Client** requests HTML from your app; HTML references CDN URLs for assets. Browser fetches assets from CDN — usually <50ms global.
5. **App server** handles only dynamic content (API responses, SSR HTML). Static load is offloaded entirely.

### Why this wins

- **Cost**: object storage at $0.023/GB/month + CDN at fractions of a cent per request is dramatically cheaper than app-server bandwidth and threads.
- **Latency**: CDN edge delivery from a PoP near the user beats your origin server in any geography.
- **Scale**: object storage and CDN scale infinitely; app servers are bounded.
- **Reliability**: object store + CDN have 99.99%+ availability independently of your app.
- **Deploy decoupling**: deploying static assets = blob upload; doesn't require app restart.
- **Cache busting**: content-hash filenames mean assets are immutable; can be cached forever at edge.
- **Reduced app surface**: fewer code paths to maintain; less attack surface.

### Variants

**Single-Page Application (SPA)**: HTML + JS shell from CDN; JS makes API calls to backend. React, Vue, Angular apps deployed entirely to CDN. Build outputs to S3 + CDN; backend separate.

**Server-rendered + static**: app generates dynamic HTML; static assets (images, CSS, JS bundles) come from CDN.

**Static-site generator**: Hugo, Jekyll, Gatsby, Astro, Eleventy generate static HTML at build time; entire site lives on CDN. No backend at all for many use cases (marketing sites, documentation).

**Jamstack**: marketing term for the pattern: prebuilt static + dynamic via APIs + edge functions. Cloudflare Pages, Vercel, Netlify, AWS Amplify all built around this.

**Edge functions**: compute at the CDN edge for personalization without a full backend round trip. Cloudflare Workers, Lambda@Edge, Vercel Edge Functions, Fastly Compute@Edge, Deno Deploy.

**Image optimization pipelines**: store original in blob; CDN/edge service resizes/optimizes on the fly (Cloudflare Images, AWS Image Optimizer, Imgix, Cloudinary). User requests `image.jpg?w=400` and gets a 400px version cached at edge.

### Cache strategies

- **Immutable assets** (content-hashed filenames): `Cache-Control: public, max-age=31536000, immutable`. Cached forever; never re-fetched.
- **HTML** (changes per release): short TTL or `must-revalidate`; CDN can cache briefly.
- **API responses**: usually `no-cache` or stale-while-revalidate.
- **Origin policies**: blob store has its own cache headers; CDN respects them.

### Anti-patterns

- **Serving static from app servers** at any scale.
- **Long TTLs on mutable HTML** — users see stale pages.
- **No content hashing** — cache busting requires URL changes.
- **Same-origin for static and dynamic** when CDN could split them.
- **Not using a CDN** — origin alone is global latency by physics.
- **Manual S3 uploads instead of build pipeline** — drift, errors.

### Costs of static content hosting

Almost universally favorable. Typical numbers (2024):
- S3 storage: $0.023/GB/month.
- S3 GET requests: $0.0004 per 1,000.
- CloudFront delivery: ~$0.085/GB to internet (varies by region; tiered).
- Cloudflare bandwidth: often included in plan or much cheaper.

Compare to:
- App server bandwidth (EC2): outbound to internet $0.09/GB.
- App server thread serving static content: blocks the thread for milliseconds while it could be doing real work.
- App server replication: every app server has the static content; bigger images, bigger deploys.

For any site with non-trivial traffic, static-content offload pays for itself in days.

### Trade-offs (Static Content Hosting)

Advantages:
- **Cheap and fast** at any scale.
- **Independent deployment** of static assets.
- **Global delivery** via CDN.
- **Reduced app server load** and complexity.
- **Standard, well-supported** pattern.

Disadvantages:
- **Two systems to manage** (blob + CDN).
- **Cache invalidation** is a real concern (use content hashing).
- **Cross-origin issues**: CDN and API on different origins need CORS.
- **Versioning** of assets must be coordinated with HTML references.
- **Limited dynamic behavior** (mitigated by edge functions).

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **Claim Check (Hohpe & Woolf)** | Original EIP; same concept. |
| **Reference Token in message** | Same pattern; different name. |
| **AWS SQS Extended Client** | Transparent claim check. |
| **Apache Camel Claim Check EIP** | Framework support. |
| **Event-sourcing with externalized large events** | Variant for event stores. |
| **Static Content Hosting** | Original Azure pattern. |
| **Single-Page App (SPA)** | Variant: HTML+JS shell on CDN. |
| **Static-Site Generator (SSG)** | Variant: full static site. |
| **Jamstack** | Marketing umbrella. |
| **Edge functions** | Hybrid: static + edge compute. |
| **Image optimization on the fly** | Variant for transformed images. |
| **[CDN](../scale/cdn.md)** | The delivery layer this pattern relies on. |
| **[Valet Key](../sec/gatekeeper-valet-key.md)** | Complementary for direct client access. |
| **[API Gateway](../comm/api-gateway.md)** | Routes dynamic requests; static is bypassed. |

## When NOT to use

**Claim Check:**
- Payloads are small — extra hop and code complexity not worth it.
- All consumers always need full payload immediately — wasted indirection.
- No reliable blob store available.

**Static Content Hosting:**
- Truly dev / hobbyist sites — overkill.
- Heavily-personalized HTML with little static surface.
- Where regulatory constraints require all data through your servers.

---

## Real-world implementations

| Tool | Type |
|---|---|
| **AWS SQS Extended Client Library** | Transparent claim check |
| **Apache Camel** | Claim Check EIP |
| **AWS S3 + CloudFront** | Static hosting + CDN |
| **Azure Blob + Azure CDN / Front Door** | Static hosting + CDN |
| **GCP Cloud Storage + Cloud CDN** | Static hosting + CDN |
| **Cloudflare R2 + Cloudflare CDN** | Static hosting + CDN |
| **Vercel, Netlify, Cloudflare Pages, AWS Amplify** | Jamstack platforms |
| **Hugo, Jekyll, Gatsby, Astro, Eleventy, Next.js** | Static-site / SSG frameworks |
| **Imgix, Cloudinary, Cloudflare Images** | Image optimization services |
| **GitHub Pages** | Free static hosting |
| **CloudFront Functions, Lambda@Edge, Cloudflare Workers, Vercel Edge, Fastly Compute@Edge** | Edge functions |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **Most modern SaaS** | SPA on CDN + backend API. | ✅ Industry universal |
| **Netflix** | Massive use of CDN-fronted static + image optimization. | ✅ Verified — Netflix Tech Blog |
| **YouTube, Facebook, Instagram, TikTok** | Heavy claim-check pattern for media pipelines. | ⚠ Implied by architecture; specifics in talks |
| **Stripe** | API + JS SDK hosted on CDN. | ✅ Verified — Stripe docs |
| **Shopify** | Storefront assets on CDN; theme assets in object storage. | ✅ Verified — Shopify docs |
| **GitHub** | github.io is static hosting; releases assets on object storage. | ✅ Verified |
| **Documentation sites everywhere** | Read the Docs, Hugo sites, MkDocs sites — all SSG+CDN. | ✅ Industry standard |
| **Most marketing sites of tech companies** | Jamstack / SSG. | ✅ Universally adopted |
| **AWS S3+CloudFront customers** | Millions; the canonical static-hosting stack. | ✅ Industry standard |
| **Vercel/Netlify customers** | Millions of frontend deployments. | ✅ Industry standard |

---

## Further reading

- *Enterprise Integration Patterns* (Hohpe & Woolf, 2003) — Store-in-Library / Claim Check.
- Microsoft Azure Architecture Center — Claim Check, Static Content Hosting.
- AWS Architecture Blog — large messages and SQS.
- *Designing Web APIs* — chapter on CDN-fronted APIs.
- Jamstack.org — community resources and conferences.
- Cloudflare blog, Vercel blog — edge architecture deep dives.
- *Building Microservices* (Newman, 2nd ed.) — messaging patterns.
- AWS / Azure / GCP CDN documentation.

---

*Diagram sources: [`../diagrams/src/claim-check.d2`](../diagrams/src/claim-check.d2), [`../diagrams/src/static-content-hosting.d2`](../diagrams/src/static-content-hosting.d2).*

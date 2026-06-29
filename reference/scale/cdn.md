# Content Delivery Network (CDN)

**Aliases:** Static Content Hosting (Azure), Edge Caching, Edge Network
**Category:** Scalability
**Sources:**
[Microsoft Azure](https://learn.microsoft.com/en-us/azure/architecture/patterns/static-content-hosting) ·
[Neo Kim](https://systemdesign.one/system-design-interview-cheatsheet/) ·
[ByteByteGo](https://github.com/ByteByteGoHq/system-design-101)

---

## Problem

> [!TIP]
> **ELI5.** Your shop is in one city. Customers in other countries have to wait days for shipping. Also, when many people order at once, your one shop runs out of staff. Better: open small stockrooms in every major city, pre-stocked with the popular items. Most customers pick up locally and never bother the main shop.

A single origin server (or even a small fleet) has two problems for global users. **Latency**: physics dictates that a request from Tokyo to Virginia takes at least ~150 ms of round-trip time just for light to traverse the cable, regardless of how fast your server is. For an asset-heavy page with dozens of resources, this compounds into seconds of perceived load time. **Bandwidth and load**: every byte every user downloads comes out of the origin's pipe; popular content can hammer the origin's network and CPU even when serving simple cached files.

Both problems share a property: most of what users request is *the same* content (images, JS bundles, video segments, CSS, fonts) or content that *could be* the same with appropriate cache headers (API responses, HTML pages). Serving identical bytes from a single origin every time, for millions of users, is enormously wasteful.

## How it works

> [!TIP]
> **ELI5.** Put **edge servers** in every major city. When a user requests something, send them to the *nearest* edge. The edge keeps copies of popular files; it only bothers the main server the first time someone asks for a file. Everyone after that gets it from the local edge.

A CDN is a network of **edge servers** (also called *Points of Presence* or *POPs*) distributed across many geographic locations. Each edge holds a cache of content fetched from your **origin server**. User requests are routed to the nearest edge — typically by **GeoDNS** (DNS resolves to a different IP based on the requester's location) or **anycast** (the same IP is announced from many locations, and BGP routes to the closest one).

![CDN topology](../diagrams/svg/cdn-topology.svg)

In the topology above, the **Origin Server** is your application's source of truth — your AWS/GCP/Azure region or data center. **Edge POPs** in Virginia, Frankfurt, and Singapore each cache content close to their respective user populations. A **US user** is routed to the Virginia edge; an **EU user** to Frankfurt; an **Asia user** to Singapore. The edges serve cached content directly (solid colored arrows) — fast, no origin involvement. Only on **cache miss** (dashed gray arrows) does the edge fetch from the origin, store the result, and serve it.

Two consequences: latency for cached requests drops from "speed-of-light to origin" to "speed-of-light to nearest edge" — usually 5–30 ms instead of 100–300 ms. And origin load drops dramatically: for a popular asset, the origin sees one request per edge per TTL period, instead of one request per user. A site that would otherwise need to scale the origin for global peak traffic instead scales for "cache misses" — typically a tiny fraction.

The cache lifecycle is the operational core:

![CDN cache hit vs miss flow](../diagrams/svg/cdn-hit-miss.svg)

Every request begins with **DNS resolution** (steps 1–2): the client asks for `cdn.example.com`; the CDN's DNS infrastructure replies with the IP address of the nearest edge POP (GeoDNS uses the client IP's geolocation; anycast uses BGP routing).

On a **cache hit** (green, steps 3a–4a), the edge already has the asset stored locally — having served it to someone in the same region recently. It responds in a few milliseconds with the cached bytes, never touching the origin. This is the case for the vast majority of requests in a healthy CDN deployment.

On a **cache miss** (orange, steps 3b–7b) — the first request for an asset in this region, or a request after the cached copy expired — the edge fetches from the origin (this is sometimes called *cache fill* or *back-fetch*). The origin returns the bytes along with cache-control headers (`Cache-Control: max-age=86400`, `ETag`, `Last-Modified`) that tell the edge how long to cache it. The edge stores the response, then forwards the bytes to the user. Subsequent requests for the same asset from any user in this region now hit the cache.

The interesting design questions are about **what to cache and for how long**. Static assets (images, JS bundles with content-hashed filenames, fonts) can be cached for a year — they're immutable by name, so the URL changes when content changes. HTML pages and API responses are trickier: too long a TTL and users see stale content; too short and the cache is pointless. The standard tools are **TTL** (time-based expiry), **Cache-Control / ETag / If-None-Match** (HTTP-native revalidation), **purge APIs** (the CDN lets you proactively invalidate cached content by URL or tag), and **stale-while-revalidate** (serve stale, asynchronously refresh — best of both).

Modern CDNs do much more than cache static files. **Edge compute** (Cloudflare Workers, AWS Lambda@Edge, Vercel Edge Functions) lets you run code at the edge, enabling personalized HTML rendering, A/B testing, request transformation, and entire applications served from the edge with millisecond latency to users worldwide. **Edge databases** (Cloudflare D1, Workers KV, Durable Objects; Vercel Postgres) push state to the edge. The CDN gradually becomes the runtime, not just the cache.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **Pull CDN** | Edges fetch from origin on first request and cache. The default; lowest origin work. |
| **Push CDN** | Content is proactively pushed to edges (e.g., for video releases). Used when you need every edge warmed before launch. |
| **Edge compute / edge functions** | Run code at the edge — see Cloudflare Workers, Lambda@Edge, Vercel/Netlify Edge, Fastly Compute@Edge. |
| **Anycast routing** | Same IP from many locations; BGP routes to nearest. The basis of Cloudflare and Google's edge networks. |
| **GeoDNS** | DNS resolves differently by client location. The basis of Akamai-style edge routing. |
| **Static Content Hosting** (Microsoft Azure) | The narrower pattern of serving static assets from object storage + CDN; very common for SPAs. |

## When NOT to use

- **Hyperlocal services** where all users are in one geographic area — a CDN is overkill if your users are all in one metro. (Though the cache-offload value can still be worthwhile.)
- **Highly personalized content with no shared structure** — if every response is unique to the user, cache hit rate is near zero and you've added a hop.
- **Strong consistency / no-stale-data requirements without careful design** — CDN edges are eventually consistent with the origin. You can't trust them for "have I purchased this yet?" lookups without purge-driven invalidation.
- **Truly tiny sites** — modern hosting (Vercel, Netlify, Cloudflare Pages) provides CDN for free; but if you're running a personal blog on a Raspberry Pi for fun, a CDN is unnecessary ceremony.

---

## Real-world implementations

| CDN | Notes |
|---|---|
| **Cloudflare** | Massive anycast network; aggressive free tier; strong edge-compute story (Workers). |
| **AWS CloudFront** | Tightly integrated with S3, ALB, Lambda@Edge; the AWS default. |
| **Akamai** | The original CDN (founded 1998); largest physical footprint; enterprise focus. |
| **Fastly** | Programmable edge via VCL (Varnish-based) and now Compute@Edge (WASM). Loved by ops engineers for instant purge and configurability. |
| **Google Cloud CDN** | Anycast on Google's global network; uses Premium Tier network for end-to-end Google routing. |
| **Azure Front Door / Azure CDN** | Microsoft's offerings; tied to Azure ecosystem. |
| **BunnyCDN, KeyCDN, jsDelivr** | Cheaper / OSS-friendly options. |
| **Vercel / Netlify Edge Network** | Application-platform CDNs optimized for Next.js / static-site / serverless deploys. |

## Companies using it (notable examples)

Virtually every consumer-facing internet company uses a CDN. Selected canonical examples:

| Company | Use | Status |
|---|---|---|
| **Netflix → Open Connect** | Operates its own dedicated CDN (Open Connect) — ISPs install Netflix appliances inside their networks. Streams the majority of internet video bandwidth in the US/EU at peak. | ✅ Verified — [Open Connect](https://openconnect.netflix.com/) |
| **Cloudflare itself** | Operates one of the largest networks (300+ cities); fronts millions of sites. | ✅ Verified — [Cloudflare network map](https://www.cloudflare.com/network/) |
| **YouTube / Google** | Operates Google Global Cache (GGC) deployed inside ISPs — analogous to Netflix Open Connect for YouTube traffic. | ✅ Verified — [Google Peering & Content Delivery](https://peering.google.com/) |
| **Facebook / Meta** | Operates its own CDN-like edge network (FBCDN); also uses commercial CDNs as fallback. | ⚠ Public engineering posts exist; specific link not re-verified |
| **Spotify** | Uses multiple commercial CDNs (Akamai, Cloudfront, Fastly historically) for audio delivery. | ⚠ Discussed in talks; not re-verified |
| **GitHub** | Uses Fastly for serving raw content and Pages; Cloudflare for some other surfaces. | ✅ Verified — fastly.net domains in GitHub Pages responses |
| **Shopify** | Uses Fastly for storefront delivery. | ✅ Verified — [Fastly customer story](https://www.fastly.com/customers/shopify) |
| **Discord** | Uses Cloudflare. | ⚠ Visible in DNS / response headers; not re-verified by formal source |

**⚠ marks claims widely known industry-wide but not re-verified by primary-source fetch.**

---

## Further reading

- High Performance Browser Networking, Ilya Grigorik — [hpbn.co](https://hpbn.co/) — free book; the CDN/edge chapters are foundational.
- Cloudflare Learning Center, *What is a CDN?* — good high-level intro.
- *Web Performance in Action*, Jeremy Wagner — covers CDN as part of the broader latency story.
- Fastly's *Altitude* talks and Cloudflare's TV / blog series — operationally rich CDN content.
- *The Tao of HashiCorp*-style blogs from Cloudflare on edge compute (Workers, Durable Objects) for the modern "edge as runtime" story.

---

*Diagram sources: [`../diagrams/src/cdn-topology.d2`](../diagrams/src/cdn-topology.d2), [`../diagrams/src/cdn-hit-miss.d2`](../diagrams/src/cdn-hit-miss.d2).*

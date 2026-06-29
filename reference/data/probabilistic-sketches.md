# Probabilistic Data Structures: Bloom Filter, HyperLogLog, Count-Min Sketch

**Aliases:** Sketches, Approximate Data Structures, Streaming Algorithms
**Category:** Data structures / Scalability
**Sources:**
[Bloom — *Space/Time Trade-offs in Hash Coding with Allowable Errors* (CACM 1970)](https://dl.acm.org/doi/10.1145/362686.362692) ·
[Flajolet, Fusy, Gandouet, Meunier — *HyperLogLog* (AofA 2007)](https://algo.inria.fr/flajolet/Publications/FlFuGaMe07.pdf) ·
[Cormode, Muthukrishnan — *An Improved Data Stream Summary: The Count-Min Sketch* (2005)](https://www.cs.princeton.edu/courses/archive/spring04/cos598B/bib/CormodeM-CM.pdf) ·
[Heule, Nunkesser, Hall — *HyperLogLog in Practice (HLL++)* (EDBT 2013)](https://research.google/pubs/pub40671/) ·
[Redis — Probabilistic data types](https://redis.io/docs/latest/develop/data-types/probabilistic/) ·
*Designing Data-Intensive Applications* (Kleppmann), Ch 11

---

## Problem

> [!TIP]
> **ELI5.** Sometimes you have *billions* of items and need to answer questions about them, but you cannot fit them all in memory. **Probabilistic data structures** give you approximate answers in *tiny* memory by accepting a small, bounded error. Three flavors cover most needs: **Bloom filter** ("is X in the set?" — never wrong about NO), **HyperLogLog** ("how many distinct items?" — accurate to ~1%), **Count-Min Sketch** ("how often did X appear?" — never underestimates). Each fits in kilobytes regardless of input size.

Exact data structures cost memory proportional to the data:

- A `Set` of 1 billion 30-character strings: **30 GB+**.
- A `Map` counting frequency of 1 billion distinct URLs: **30+ GB**.
- A `Counter` of unique visitors per hour, exact: depends on cardinality, but easily gigabytes.

At web scale, this is often impossible:
- Routers see 100M+ flows/second; can't store them all.
- Search engines process billions of queries/day; can't keep exact distinct counts cheaply.
- LSM-tree compactions need fast "does this key exist?" checks across many SSTables.
- DDoS detection systems need "how many requests has this IP made?" across millions of IPs.

The insight from theoretical computer science (starting with Bloom in 1970): **you can answer many of these questions approximately in sublinear memory** if you accept a controlled probability of error. The trade-off is a knob: more memory → smaller error.

The three structures covered here are the most widely-used probabilistic sketches in industry. Each answers one specific question and is wildly cheaper than the exact alternative.

## How they work

> [!TIP]
> **ELI5.** All three structures share the same trick: **hash items into a small fixed structure, accept collisions, exploit the math of randomness to bound the resulting error**. The differences are: what they're computing, how they handle collisions, and which direction the error goes.

The comparison:

![Three sketches side by side](../diagrams/svg/sketches-comparison.svg)

### Bloom Filter — "Is X in the set?"

The simplest sketch. Conceived by **Burton Bloom in 1970** — pre-dates much of modern computing.

![Bloom filter mechanics](../diagrams/svg/bloom-filter.svg)

**Structure**: a bit array of `m` bits + `k` independent hash functions.

**Insert(x)**: compute `k` hash values; set the corresponding `k` bits to 1.

**Query(x)**: compute the same `k` hash values; if all `k` bits are 1 → "possibly in set"; if any is 0 → "definitely not in set."

**Key property**:
- **No false negatives**: if Bloom says "no," the item is definitely absent.
- **Possible false positives**: if Bloom says "yes," the item *might* be present (or might be a collision of several others' bits).

The false positive rate (FPR) is:
$$\text{FPR} \approx \left(1 - e^{-kn/m}\right)^k$$

where `n` is items inserted. With `n=1M, m=10M, k=7`: FPR ≈ 0.8%, using only 1.25 MB. Compared to storing the actual strings (~30 MB at 30 chars each), that's a **24× memory reduction** for the cost of accepting 0.8% false positives.

**When to use Bloom filters**:

- **LSM-tree negative lookups** (LevelDB, RocksDB, Cassandra): before searching an SSTable for a key, check its Bloom filter. If it says "no," skip the SSTable. Massive speedup for non-existent keys.
- **Web cache filtering**: "Is this URL on the blocklist?" Chrome's Safe Browsing uses Bloom filters to avoid round-tripping every URL to Google's servers.
- **Cache penetration prevention**: see [Cache-Aside pitfalls](../cache/cache-aside.md). Filter out queries for keys that don't exist in the DB.
- **Distributed deduplication**: "Have I already processed this event ID?" — quick negative check.
- **Bitcoin / blockchain SPV clients**: filter the chain for transactions relevant to your wallet.
- **NoSQL query planning**: rule out obvious non-matches without scanning data files.

**Variants**:
- **Counting Bloom filter**: each cell is a small counter, not a bit; supports deletions.
- **Scalable Bloom filter**: grows as needed; multiple stacked filters.
- **Cuckoo filter**: alternative with better space-efficiency and supports deletions.
- **XOR filter**: even more space-efficient; static (no inserts after build).

### HyperLogLog — "How many distinct items?"

Answers: **cardinality** (count of distinct values) in a stream, with constant memory.

![HyperLogLog mechanics](../diagrams/svg/hyperloglog.svg)

**Intuition**: a uniformly-distributed hash has a `1/2^k` probability of starting with `k` leading zeros. Seeing a long run of leading zeros implies you've sampled many items. The maximum number of leading zeros across all items hashed estimates `log2(distinct count)`.

**The clever bit**: doing this with one counter has too much variance. HyperLogLog uses `m = 2^p` "buckets":

1. Hash each item; use the first `p` bits to pick a bucket.
2. Count leading zeros in the remaining bits.
3. Update `max_zeros[bucket] = max(max_zeros[bucket], leading_zeros)`.

After the stream, combine the bucket estimates with a harmonic mean and a bias-correction constant:
$$\text{distinct} \approx \frac{\alpha_m \cdot m^2}{\sum_i 2^{-M_i}}$$

Error scales as `1.04 / sqrt(m)`. With `m=16384` buckets using ~12 KB total:
- ~0.8% standard error.
- Accurate from cardinalities of 0 to billions.
- Constant memory regardless of input size.

**Where it's used**:

- **Google BigQuery `APPROX_COUNT_DISTINCT()`** — HLL++ implementation.
- **Snowflake, Redshift, Druid, Presto, Pinot, ClickHouse** — all have HLL-based approximate distinct functions.
- **Redis `PFADD` / `PFCOUNT` / `PFMERGE`** — HLL primitives as native commands.
- **Apache Spark, Flink** — built-in HLL aggregates.
- **Twitter** — tweet impression cardinality.
- **Reddit** — per-post unique view counters.
- **Cloudflare, DataDog, NewRelic** — unique-IP / unique-user metrics in observability.

The killer feature: HLLs are **mergeable**. Per-shard HLLs can be combined (max per bucket) into a global HLL. This is why distributed analytics engines love them — each worker computes a local HLL, the coordinator merges.

**HLL++ variants** add bias correction at low cardinalities and sparse representations to save memory when distinct count is small.

### Count-Min Sketch — "How many times did X appear?"

Answers: **frequency estimate** for items in a stream.

![Count-Min Sketch](../diagrams/svg/count-min-sketch.svg)

**Structure**: a 2D array of `d` rows × `w` columns of counters, with `d` independent hash functions (one per row).

**Update(x, c)**: for each row `i`, compute `hash_i(x) mod w` → column `j`; add `c` to `counters[i][j]`.

**Query(x)**: for each row `i`, look up `counters[i][hash_i(x) mod w]`; return the **minimum** across rows.

**Why MIN?** Collisions only *add* to counters (no negative updates in the basic version), so the smallest of the `d` candidate counters has the fewest collisions and is closest to the true count.

**Bounds**: with `w = O(1/ε)` and `d = O(log 1/δ)`, the overestimate is at most `ε × total_stream_count` with probability `1-δ`. In practice, a few KB suffices for very useful accuracy.

**Properties**:
- **Never underestimates** (in the basic variant).
- **Overestimates by at most a controlled amount.**
- **Mergeable**: add sketches cell-wise to combine streams.
- **Heavy hitters**: pair with a small top-K heap updated alongside the sketch; you get the top-K most frequent items in constant memory.

**Where it's used**:

- **Streaming top-K**: most-frequent IPs (DDoS detection), URLs, search terms, words.
- **Network monitoring**: heavy-flow detection in routers (NetFlow analysis).
- **Twitter, Facebook**: real-time trending detection uses Count-Min Sketch (with HLL for unique-user-per-trend).
- **Cassandra**: internal use in compaction decisions.
- **NLP preprocessing**: frequency tables for word2vec, BPE tokenization on large corpora.
- **Cloudflare, Imperva**: rate-limit and abuse detection at the edge.

**Variants**:
- **Count-Min-Log Sketch**: logarithmic counters; saves memory at cost of precision for low counts.
- **Conservative update**: only increment the smallest matching counters; reduces overestimate.
- **Count Sketch**: similar but signs the updates; can both over- and under-estimate, but unbiased in expectation.

### Common themes

All three structures:

- **Sublinear memory** in the size of the input (often constant, regardless of stream size).
- **Controlled approximation error** with mathematically proven bounds.
- **Fast O(k) updates and queries**, where k is the number of hash functions (typically 4-10).
- **Mergeable across shards**: compute partials independently, combine at the end. Essential for distributed systems.
- **One-sided error**: Bloom has no false negatives; CMS never underestimates; HLL is roughly unbiased.
- **Trade memory for error**: more cells → tighter bounds. The knob is yours.

### When to choose which

| Question | Sketch |
|---|---|
| "Is X in the set?" | Bloom filter |
| "Is X likely in the DB / cache?" | Bloom filter (negative lookup) |
| "How many unique users today?" | HyperLogLog |
| "How many distinct values in this column?" | HyperLogLog |
| "How often did X appear?" | Count-Min Sketch |
| "What are the top 10 most frequent items?" | Count-Min Sketch + top-K heap |
| "How many users visited this URL?" | HLL per URL |
| "How many requests has IP X made?" | Count-Min Sketch keyed by IP |
| "Has user X ever clicked ad Y?" | Bloom filter per user, or per ad |

You often use them in **combination**. Twitter's trending-topics system uses Count-Min Sketch to find topics with high tweet counts, then HLLs per topic to find topics with many *unique users* tweeting them (filtering out single-user spam).

### Why they're "magical"

The right reaction the first time you see HLL or CMS is amazement: counting distinct items in a *billion*-event stream using **12 KB** with ~1% error feels impossible. The trick is that the math of random hashing is doing the work for you — these structures don't need to remember items, just the *signatures of their hashes*.

This is also why probabilistic structures are **so cheap to share**: a Bloom filter for "URLs Google has flagged as malicious" is downloaded to your browser in a small fixed size, not the full URL list. HyperLogLog per A/B test bucket is bytes. The bandwidth and storage win for distributed systems is enormous.

### Limitations and caveats

- **Bounded error is not zero error**: never use a Bloom filter where false positives would cause wrong correctness decisions (banking, security verification — but okay as a *first* filter that's followed by exact check).
- **You commit to memory upfront**: hard to dynamically resize sketches without restart.
- **Hash quality matters**: bad hashes (poor avalanche) wreck the error bounds. Use MurmurHash, xxHash, or similar.
- **Cardinality assumptions**: HLL's error model assumes hashed items are uniform; pathological inputs can violate this.
- **No deletion in basic Bloom or CMS**: use Counting Bloom or specialized variants if needed.
- **CMS overestimate can be unbounded** for very skewed streams; the bound is *probabilistic*.

### A note on more exotic sketches

The probabilistic-DS family is rich. Some others worth knowing:

- **T-Digest** — approximate quantiles (median, p99); used by DataDog, Cassandra, Druid.
- **MinHash** — approximate Jaccard similarity; used in near-duplicate detection.
- **q-digest, GK-sketch** — alternative quantile sketches.
- **Apache DataSketches** library (Yahoo!) — a library of sketches for distributed analytics.
- **HLL++** — Google's improved HLL with bias correction.
- **Cuckoo filter** — better Bloom alternative.
- **XOR filter, Ribbon filter** — even more space-efficient than Cuckoo for static sets.

But Bloom, HLL, and Count-Min cover the vast majority of practical use cases.

---

## Variants & related patterns

| Variant | Difference |
|---|---|
| **Counting Bloom filter** | Supports deletion. |
| **Scalable Bloom filter** | Grows on demand. |
| **Cuckoo filter** | Bloom alternative; better space, supports deletion. |
| **XOR / Ribbon filter** | Best space for static sets. |
| **HLL / HLL++** | Distinct counting; HLL++ adds bias correction. |
| **Count-Min Sketch** | Frequency estimation. |
| **Count Sketch** | Signed variant; unbiased. |
| **Top-K (frequent items)** | CMS + small heap. |
| **T-Digest** | Approximate quantiles. |
| **MinHash** | Similarity estimation. |

## When NOT to use

- **When exact answers are legally or correctness-required** (financial sums, regulatory counts).
- **When the data fits in memory anyway** — exact structures are simpler.
- **As the only filter** in a security-critical path — always pair with exact verification.
- **When stream is small** — overhead may exceed exact storage cost.

---

## Real-world implementations

| Tool | Sketches |
|---|---|
| **Redis** | Bloom (RedisBloom), HLL (PFADD/PFCOUNT), Count-Min, T-Digest, Top-K. |
| **Google BigQuery** | HLL++ via APPROX_COUNT_DISTINCT. |
| **Snowflake, Redshift, Presto, Pinot, Druid, ClickHouse** | HLL distinct counting; many support Count-Min/T-Digest. |
| **Apache DataSketches** (Yahoo!) | Open-source library of sketches: HLL, Theta, KLL, Quantiles, CPC. |
| **stream-lib** | Java library with Bloom, HLL, CMS, T-Digest. |
| **algebird** (Twitter) | Scala sketches library. |
| **probables-cpp, pdsa-python** | Per-language libraries. |
| **RocksDB, LevelDB, Cassandra** | Bloom filters for LSM negative lookups. |
| **Chrome Safe Browsing** | Bloom filter for malicious URL filtering. |
| **Bitcoin Core, BIP-37 SPV clients** | Bloom filters for transaction filtering. |

## Companies / canonical uses

| Where | Use | Status |
|---|---|---|
| **Google** | HLL++ powers BigQuery's APPROX_COUNT_DISTINCT and many internal analytics. | ✅ Verified — [HLL++ paper](https://research.google/pubs/pub40671/) |
| **Facebook / Meta** | Sketches in observability and internal analytics. | ✅ Verified — Meta engineering blog |
| **Twitter / X** | Trending topics (CMS + HLL); algebird library is theirs. | ✅ Verified — Twitter Engineering blog + algebird OSS |
| **Reddit** | HLL for unique pageviews per post. | ✅ Verified — Reddit Engineering blog |
| **Cloudflare** | Sketches for DDoS detection and edge rate-limiting. | ✅ Verified — Cloudflare blog |
| **DataDog** | T-Digest for quantile metrics. | ✅ Verified — DataDog engineering blog |
| **Yahoo!** | DataSketches library created and open-sourced by Yahoo. | ✅ Verified — [Apache DataSketches](https://datasketches.apache.org/) |
| **LinkedIn** | Pinot uses HLL extensively. | ✅ Verified — LinkedIn / Apache Pinot |
| **Cassandra, RocksDB, LevelDB** | Bloom filters internal to LSM-tree implementations. | ✅ Verified — source code, docs |
| **Chrome (Google)** | Safe Browsing uses Bloom filters in clients. | ✅ Verified — Safe Browsing technical docs |

---

## Further reading

- Bloom, *Space/Time Trade-offs in Hash Coding with Allowable Errors* (CACM 1970) — the original.
- Flajolet et al., *HyperLogLog* (AofA 2007) — the math.
- Heule, Nunkesser, Hall, *HyperLogLog in Practice (HLL++)* (EDBT 2013) — the practical version.
- Cormode & Muthukrishnan, *An Improved Data Stream Summary: The Count-Min Sketch* (2005).
- *Probabilistic Data Structures and Algorithms for Big Data Applications* (Gakhov, 2018) — accessible book.
- *Designing Data-Intensive Applications* (Kleppmann), Ch 11 — context within streaming systems.
- Redis Probabilistic Data Types documentation.
- Apache DataSketches documentation and papers.
- Andy Pavlo's CMU lectures on database internals — Bloom filters in storage engines.

---

*Diagram sources: [`../diagrams/src/bloom-filter.d2`](../diagrams/src/bloom-filter.d2), [`../diagrams/src/hyperloglog.d2`](../diagrams/src/hyperloglog.d2), [`../diagrams/src/count-min-sketch.d2`](../diagrams/src/count-min-sketch.d2), [`../diagrams/src/sketches-comparison.d2`](../diagrams/src/sketches-comparison.d2).*

# Patterns

A curated, source-verified, diagram-rich reference for **system design patterns** (and, soon, **LLM application patterns**).

Each pattern gets its own page with:

- **ELI5 callouts** explaining the problem and the solution in plain language
- **Inline D2 diagrams** rendered to SVG and committed alongside the source
- **Variants & related patterns** cross-linked across the catalog
- **When NOT to use** — because patterns are tools, not gospel
- **Real-world implementations** (libraries, products, services)
- **Companies using it** — every claim marked ✅ verified-by-fetch or ⚠ widely-known-but-unverified

The goal is a working reference an engineer can read end-to-end or skim by category — grounded in primary sources (Microsoft Azure Cloud Design Patterns, microservices.io, Joshi's *Patterns of Distributed Systems*, Kleppmann's DDIA, AWS Builders' Library, Fowler / Young / Vernon on event sourcing, Hohpe & Woolf on EIP, Hennessy & Patterson on memory hierarchy, and others), not regurgitated from a model's training data.

---

## What's in this repo

```
patterns/
├── reference/                     # 86 system-design pattern pages (complete)
│   ├── README.md                  # full TOC + themes + sources
│   ├── arch/                      # architectural styles (10)
│   ├── data/                      # storage, replication, partitioning (22)
│   ├── cache/                     # caching strategies & hierarchy (3)
│   ├── comm/                      # service communication (16)
│   ├── res/                       # resilience (6)
│   ├── scale/                     # scalability (1; more coming)
│   ├── coord/                     # coordination protocols (6)
│   ├── block/                     # distributed-systems building blocks (10)
│   ├── ops/                       # deployment, observability, ops (8)
│   ├── sec/                       # security & identity (3)
│   └── diagrams/
│       ├── src/                   # 170 D2 source files
│       └── svg/                   # 170 rendered SVGs (committed)
│
└── .knowledge/                    # structured pattern catalog (JSON)
    ├── index.json                 # entry point
    ├── graph.json                 # cross-pattern relationships
    └── semantic/
        ├── system-design-patterns-unified-registry.json    # PRIMARY: 142 patterns merged from 6 sources
        ├── system-design-patterns-azure.json               # Microsoft Azure (43)
        ├── system-design-patterns-microservices-io.json    # Chris Richardson (~50)
        ├── system-design-patterns-distributed-joshi.json   # Unmesh Joshi (30)
        ├── ddia-kleppmann-reference.json                   # DDIA conceptual mapping
        │
        ├── llm-patterns-eugene-yan.json                    # LLM patterns (Yan)
        ├── llm-patterns-anthropic.json                     # LLM patterns (Anthropic)
        ├── llm-patterns-12-factor-agents.json              # 12-factor agents
        ├── llm-patterns-2024-2026-additions.json           # recent LLM additions
        ├── llm-agent-components-weng.json                  # agent components (Weng)
        └── llm-agent-orchestration-microsoft.json          # agent orchestration (MS)
```

---

## System design patterns — what's complete

**86 pages · 170 D2 diagrams · ~165,000 words · 0 broken internal links · 0 orphan diagrams.**

See **[`reference/README.md`](reference/README.md)** for the full table of contents and the **themes** section that maps families of patterns together (the dual-write trilogy, the resilience trinity, the three pillars of observability, building blocks compose, etc.).

The work was structured in seven phases:

| Phase | Scope | Pages |
|---|---|---|
| A | Most-cited patterns (≥3 sources): CQRS, Saga, Sharding, Circuit Breaker, etc. | 15 |
| B | Joshi distributed-systems core (Raft, Paxos, Gossip, WAL, Lamport, etc.) | 20 |
| C | Microservices / communication / data (Sidecar, Mesh, BFF, Outbox, CDC, etc.) | 21 |
| D | Caching, scaling, data structures, resilience, storage taxonomy | 13 |
| E | Ops & observability (Blue-Green, Canary, Tracing, Chaos, Audit, etc.) | 8 |
| F | Security + remaining architectural styles (Gatekeeper, OAuth, Event Sourcing) | 4 |
| G | Final additions (Space-Based, Edge, SCIM/Passkeys, Static Stability, Hierarchies) | 5 |

Each page is self-contained but cross-linked. The ELI5 admonitions make even gnarly patterns (Paxos, CRDTs, MVCC, HLC) approachable without sacrificing depth.

---

## LLM application patterns — captured but not yet built out

The `.knowledge/semantic/llm-*.json` files capture LLM-pattern catalogs from:

- **Eugene Yan** — *Patterns for Building LLM-based Systems & Products*
- **Anthropic** — agent design recommendations
- **Microsoft** — agent-orchestration architectures
- **Lilian Weng** — LLM-powered autonomous agents (component breakdown)
- **Dex Horthy** — 12-Factor Agents
- A curated set of **2024–2026 additions** (post-training-cutoff patterns; flagged with lower confidence pending re-verification)

These will get their own reference-page treatment in a follow-up pass, kept clearly **separate** from the system-design corpus.

---

## Sources

| Source | URL | Scope |
|---|---|---|
| Microsoft Azure Cloud Design Patterns | https://learn.microsoft.com/en-us/azure/architecture/patterns/ | Cloud architectural patterns (43) |
| Chris Richardson — microservices.io | https://microservices.io/patterns/index.html | Microservice-specific (~50) |
| Unmesh Joshi — Patterns of Distributed Systems | https://martinfowler.com/articles/patterns-of-distributed-systems/ | Distributed-system implementation patterns (30) |
| Martin Kleppmann — *Designing Data-Intensive Applications* | https://dataintensive.net/ | Data-systems concepts |
| Hohpe & Woolf — *Enterprise Integration Patterns* | https://www.enterpriseintegrationpatterns.com/ | Messaging patterns |
| Fowler / Young / Vernon — DDD + Event Sourcing | https://martinfowler.com/eaaDev/EventSourcing.html | Event sourcing, CQRS, DDD |
| Amazon Builders' Library | https://aws.amazon.com/builders-library/ | Static Stability, large-system practices |
| Hennessy & Patterson — *Computer Architecture* | — | Memory hierarchy fundamentals |
| Mark Richards — *Software Architecture Patterns* | https://www.oreilly.com/library/view/software-architecture-patterns/9781491971437/ | Named source for Space-Based Architecture |
| Eugene Yan, Anthropic, Lilian Weng, Dex Horthy, Microsoft | various | LLM patterns (separate corpus) |

All page citations link back to these primary sources. Per-pattern facts and company claims are flagged ✅ (verified) or ⚠ (widely-known-but-not-re-verified).

---

## Diagram tooling — D2

All diagrams use [D2](https://d2lang.com/) (declarative, text-based, GitHub-friendly):

```bash
# Render a single diagram
d2 reference/diagrams/src/circuit-breaker-flow.d2 reference/diagrams/svg/circuit-breaker-flow.svg

# Render all
for f in reference/diagrams/src/*.d2; do
  d2 "$f" "reference/diagrams/svg/$(basename "$f" .d2).svg"
done
```

Both `.d2` source and `.svg` output are version-controlled so renders are reproducible and diffs are reviewable.

---

## Reading the pages

GitHub renders the markdown directly. The `> [!TIP]` admonitions appear as GitHub callout boxes; the `![…](…)` image tags render the SVG diagrams inline.

Start with:
- **[`reference/README.md`](reference/README.md)** — full TOC and themes overview
- Any pattern page that interests you — each is self-contained

If you're learning, the **Themes** section in the reference README is the best starting point — it maps how patterns compose into families (the dual-write trilogy, resilience trinity, building blocks composition, etc.).

---

## License

The text and diagrams are released under **CC BY 4.0** — use them anywhere, with attribution. See [`LICENSE`](LICENSE).

The pattern *names* and *underlying ideas* are public knowledge and belong to the wider community; this repo's contribution is the curation, the verification, the diagrams, and the prose explaining them.

---

## Status

- ✅ System design patterns: complete (86 pages)
- ⬜ LLM patterns: captured as JSON; reference pages forthcoming
- 📋 Maintained by: [@sanzgiri](https://github.com/sanzgiri)

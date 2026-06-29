# Patterns

> A curated reference of **system-design patterns** and **LLM/agentic patterns**, with diagrams, current sources, and ELI5 explanations.

Each pattern page follows the same structure: a plain-English **ELI5 callout** for both the problem and how-it-works, **D2 diagrams inline with the prose**, variants, when **not** to use it, implementations, and companies that use it (verified ✅ or widely-attributed ⚠).

## Browse

<div class="grid cards" markdown>

-   :material-server-network:{ .lg .middle } **System Design Patterns**

    ---

    **86 pages** across distributed systems, microservices, data, caching, resilience, scale, operations, security, and architecture. Sourced from Microsoft Azure Architecture Center, microservices.io, Kleppmann (DDIA), Joshi, Hennessy & Patterson, Hohpe & Woolf, AWS Builders' Library, Fowler/Young, Richards, and others.

    [:octicons-arrow-right-24: Browse system design](system-design/README.md)

-   :material-robot-outline:{ .lg .middle } **LLM & Agentic Patterns** *(2026-current)*

    ---

    Modern patterns for LLM applications and agents — **context engineering, agentic loops, sub-agents, skills, evals, security/containment** — verified against authoritative 2025-2026 sources (Anthropic engineering, OpenAI, Simon Willison, 12-Factor Agents, MS orchestration).

    [:octicons-arrow-right-24: Browse LLM patterns](llm/README.md)

</div>

## What's in here

| Section | Pages | Diagrams | Coverage |
|---|---|---|---|
| [System Design](system-design/README.md) | 86 | 170 | **Complete** — phases A–G |
| [LLM / Agentic](llm/README.md) | 33 | 44 | Phases 2026-A–E (Context, Agentic, Evals, Security, Foundations) shipped; workflows/retrieval/memory/skills/eng/model/protocols next |

## Format conventions

Every reference page has:

- **ELI5 callouts** — `> [!TIP]` GitHub-flavored admonitions on the **Problem** and **How it works** sections
- **D2 diagrams inline** — diagrams sit *with* the prose that explains them; the prose is the walkthrough
- **Variants & related** — links to sibling patterns
- **When NOT to use** — failure modes and anti-cases
- **Implementations** — concrete tools/libraries/services
- **Companies** — ✅ verified with URL, ⚠ widely-attributed but not re-verified
- **Further reading** — primary sources only

Diagrams are authored in [D2](https://d2lang.com/) (sources in `diagrams/src/`, rendered SVGs in `diagrams/svg/`) — both committed.

## Sources

The corpus draws only from authoritative, named sources:

- Microsoft Azure Architecture Center · [microservices.io (Richardson)](https://microservices.io)
- Kleppmann — *Designing Data-Intensive Applications*
- Joshi — *System Design Patterns for Distributed Systems*
- Hennessy & Patterson — *Computer Architecture*
- Hohpe & Woolf — *Enterprise Integration Patterns*
- Richards — *Software Architecture Patterns*
- AWS Builders' Library · ByteByteGo · Neo Kim · Fowler & Young
- **For LLM patterns (2026-current):** Anthropic Engineering blog, OpenAI documentation, Simon Willison, Eugene Yan, Lilian Weng, HumanLayer (12-Factor Agents), Microsoft AI Orchestration

## License

Content is licensed [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — share and adapt with attribution. Code (D2 sources, build config) is MIT.

[:simple-github: View on GitHub](https://github.com/sanzgiri/patterns){ .md-button .md-button--primary }
[:material-tag-multiple: Topics](https://github.com/sanzgiri/patterns){ .md-button }

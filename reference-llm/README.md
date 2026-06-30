# LLM & Agentic Patterns Reference

A working reference covering the most-cited LLM and agentic patterns dominant in 2025-2026, drawn from primary sources fetched between Jun 2025 and Jun 2026.

This is a **sibling** to `../reference/` (system-design patterns). Patterns are grouped by what they do, not by source.

Each page follows the same structure used in `../reference/`:

- **ELI5 callouts** on Problem and How-it-works
- **Inline D2-rendered diagrams** explaining components and flows
- **Variants & related patterns** with cross-links
- **When NOT to use**
- **Real-world implementations** (libraries, products)
- **Companies using it** — every claim marked ✅ verified by fetch or ⚠ widely-known-but-unverified
- **Further reading** with cited URLs

> Catalog source-of-truth: `../.knowledge/semantic/llm-patterns-2026-current.json` (47 patterns, confidence 0.9), plus the older companions (`llm-patterns-anthropic.json`, `llm-patterns-eugene-yan.json`, `llm-patterns-12-factor-agents.json`, `llm-agent-components-weng.json`, `llm-agent-orchestration-microsoft.json`).

---

## Why 2026 is different

Through 2024 the dominant frame was **prompt engineering**: craft the right system prompt and the model will behave. Then in September 2025, Anthropic published *[Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)* and made it explicit: the discipline has shifted.

> "Building with language models is becoming less about finding the right words and phrases for your prompts, and more about answering the broader question of *what configuration of context is most likely to generate our model's desired behavior?*"

That post — and the family of patterns that grew around it (compaction, just-in-time context, progressive disclosure, structured note-taking, sub-agent architectures) — is the reason this reference exists as a separate corpus. The 2023 baseline (RAG, Evals, Guardrails, Defensive UX, Feedback) all still applies; it's just no longer the whole story.

Four 2026 themes get their own large sections:

1. **Context Engineering** (`ctx/`) — the new primary discipline
2. **Agentic Patterns** (`agt/`) — what changed once "agent = LLM in a tool loop" stabilized
3. **Quality & Evals** (`qua/`) — the field professionalized through 2025-2026
4. **Security & Containment** (`sec/`) — driven by the reality that powerful agents have blast radius

---

## Coverage status

| Phase | Scope | Pages | Status |
|---|---|---|---|
| **2026-A** | Context Engineering (`ctx/`) | 7 | ✅ complete |
| **2026-B** | Agentic Patterns (`agt/`) | 10 | ✅ complete |
| **2026-C** | Quality & Evals (`qua/`) | 6 | ✅ complete |
| **2026-D** | Security & Containment (`sec/`) | 5 | ✅ complete |
| **2026-E** | Foundations (`fnd/`) | 5 | ✅ complete |
| **2026-F.1** | Workflows (`wf/`) | 5 | ✅ complete |
| **2026-F.2** | Retrieval (`ret/`) | 5 | ✅ complete |
| **2026-F.3** | Memory (`mem/`) | 3 | ✅ complete |
| **2026-F.4** | Skills (`skills/`) | 3 | ✅ complete |
| **2026-F.5** | Engineering (`eng/`) | 5 | ✅ complete |
| **2026-F.6** | Model patterns (`model/`) | 4 | ✅ complete |
| **2026-F.7** | Protocols (`proto/`) | 2 | ✅ complete |

**Target reached:** 60 LLM pages, 71 diagrams.

---

## Reference index

### Context Engineering (`ctx/`) — the 2026 centerpiece
- [Context Engineering — overview](ctx/context-engineering-overview.md)
- [Context Rot](ctx/context-rot.md)
- [Compaction](ctx/compaction.md)
- [Structured Note-Taking / Agentic Memory](ctx/structured-note-taking.md)
- [Just-in-Time Context](ctx/just-in-time-context.md)
- [Progressive Disclosure](ctx/progressive-disclosure.md)
- [Context Plumbing & Reinforcement](ctx/context-plumbing-reinforcement.md)

### Agentic Patterns (`agt/`) — the heart
- [The agent loop](agt/agent-loop.md) (modern definition)
- [Workflows vs Agents](agt/workflows-vs-agents.md) (the spectrum)
- [Single agent with tools](agt/single-agent-with-tools.md) (ACI design)
- [Multi-agent orchestration](agt/multi-agent-orchestration.md) (production)
- [Sub-agent architectures](agt/sub-agent-architectures.md) for context isolation
- [Self-directed agent swarms](agt/self-directed-swarms.md)
- [Coding agents](agt/coding-agents.md)
- [Computer use](agt/computer-use.md) / browser use agents
- [Generative agents](agt/generative-agents.md) (memory-stream)
- [Maker-checker](agt/maker-checker.md) / generator-verifier (agentic)

### Quality & Evals (`qua/`)
- [Eval-driven development](qua/eval-driven-development.md) (formalized)
- [Grader taxonomy](qua/grader-taxonomy.md) (code / model / human)
- [pass@k vs pass^k](qua/pass-at-k-vs-pass-power-k.md) metrics
- [Capability vs regression evals](qua/capability-vs-regression-evals.md)
- [End-state evaluation](qua/end-state-evaluation.md)
- [Eval saturation & eval awareness](qua/eval-saturation.md)

### Security & Containment (`sec/`)
- [Containment & blast radius](sec/containment-blast-radius.md)
- [The lethal trifecta](sec/lethal-trifecta.md) (Simon Willison)
- [Egress allowlisting](sec/egress-allowlisting.md)
- [Browser-as-sandbox](sec/browser-as-sandbox.md)
- [Auto-mode permission boundaries](sec/auto-mode-permission-boundaries.md)

### Workflows (`wf/`)
- [Prompt chaining](wf/prompt-chaining.md)
- [Routing](wf/routing.md)
- [Parallelization (sectioning & voting)](wf/parallelization.md)
- [Orchestrator-workers](wf/orchestrator-workers.md)
- [Evaluator-optimizer](wf/evaluator-optimizer.md)

### Retrieval (`ret/`)
- [Basic RAG](ret/basic-rag.md) (embed + retrieve + generate)
- [Advanced RAG](ret/advanced-rag.md) (HyDE, hybrid, multi-query, MMR)
- [Contextual Retrieval](ret/contextual-retrieval.md) (Anthropic Sept 2024; –49% retrieval failures)
- [GraphRAG](ret/graph-rag.md) (Microsoft Apr 2024; aggregation-friendly)
- [Reranking](ret/reranking.md) (cross-encoder, LLM, ColBERT)

### Memory (`mem/`)
- [Memory architectures](mem/memory-architectures.md) (working / episodic / semantic / procedural / archival)
- [Prompt caching](mem/prompt-caching.md) (Anthropic Aug 2024; ~90% cost reduction)
- [Long-context vs RAG](mem/long-context-vs-rag.md) (and CAG; the 2024–2026 trade-off)

### Skills & Packaging (`skills/`)
- [SKILL.md format](skills/skill-md-format.md) (Anthropic Agent Skills, Oct 2025)
- [Code Mode](skills/code-mode.md) (code execution with MCP; Anthropic Nov 2025)
- [AGENTS.md](skills/agents-md.md) (project-level standing instructions; 60k+ projects)

### Production Engineering (`eng/`)
- [12-Factor Agents](eng/12-factor-agents.md) (Dex Horthy methodology)
- [Brain vs Hands](eng/brain-vs-hands.md) (reasoning model + executor model split)
- [Harness Design](eng/harness-design.md) (the 'agent OS' — 8 standard layers)
- [Rainbow Deployments](eng/rainbow-deployments.md) (parallel-variant rollouts)
- [LLM Observability](eng/llm-observability.md) (LangSmith, Braintrust, Helicone, Langfuse, Phoenix, OTel GenAI)

### Model-level (`model/`)
- [Reasoning Models](model/reasoning-models.md) (o1/o3, Claude extended thinking, DeepSeek R1, Gemini Thinking)
- [Speculative Decoding](model/speculative-decoding.md) (draft + verifier; 2–4× speedup)
- [Fine-Tuning vs Prompting vs RAG](model/fine-tuning-vs-prompting.md) (the specialization trinity)
- [Preference Tuning](model/preference-tuning.md) (DPO, KTO, ORPO, GRPO)

### Protocols (`proto/`)
- [MCP (Model Context Protocol)](proto/mcp.md) (Anthropic Nov 2024; universally adopted by 2025)
- [A2A (Agent-to-Agent)](proto/a2a.md) (Google Apr 2025; Linux Foundation governance)

### Foundations (`fnd/`)
- [Augmented LLM](fnd/augmented-llm.md) (Anthropic's atom)
- [Reasoning prompts](fnd/reasoning-prompts.md) (CoT, ToT, Self-Consistency, Reflection)
- [Structured outputs / function calling](fnd/structured-outputs.md)
- [ReAct (Thought / Action / Observation)](fnd/react.md)
- [Prompting fundamentals](fnd/prompting-fundamentals.md)

*LLM corpus COMPLETE — 60 pages, 71 diagrams, ~115k words.*

---

## Sources

| Author / Org | Title | Date | Fetched | Verified |
|---|---|---|---|---|
| Eugene Yan | [Patterns for Building LLM-based Systems & Products](https://eugeneyan.com/writing/llm-patterns/) | Jul 2023 | Jun 2026 | ✅ |
| Lilian Weng | [LLM-powered Autonomous Agents](https://lilianweng.github.io/posts/2023-06-23-agent/) | Jun 2023 | Jun 2026 | ✅ |
| Anthropic (Schluntz, Zhang) | [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) | Dec 2024 | Jun 2026 | ✅ |
| Anthropic (Applied AI team) | [Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) | Sept 2025 | Jun 2026 | ✅ |
| Anthropic (Jones, Kelly) | [Code Execution with MCP](https://www.anthropic.com/engineering/code-execution-with-mcp) | Nov 2025 | Jun 2026 | ✅ |
| Anthropic (Zhang, Lazuka, Murag) | [Equipping Agents for the Real World with Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills) | Oct 2025 | Jun 2026 | ✅ |
| Anthropic (Hadfield et al.) | [How We Built Our Multi-Agent Research System](https://www.anthropic.com/engineering/multi-agent-research-system) | Jun 2025 | Jun 2026 | ✅ |
| Anthropic (Grace et al.) | [Demystifying Evals for AI Agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) | Jan 2026 | Jun 2026 | ✅ |
| Dexter Horthy | [12-Factor Agents](https://github.com/humanlayer/12-factor-agents) | 2025 | Jun 2026 | ✅ |
| Microsoft (Kittel, Siemens) | [AI Agent Orchestration Patterns](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns) | Feb 2026 | Jun 2026 | ✅ |
| Simon Willison | [ai-agents tag archive](https://simonwillison.net/tags/ai-agents/) | ongoing | Jun 2026 | ✅ |
| Matt Webb | [Context plumbing](https://interconnected.org/home/2025/11/29) | Nov 2025 | Jun 2026 | ✅ |
| Armin Ronacher | [Agent design is still hard](https://lucumr.pocoo.org/2025/11/22/agent-design/) | Nov 2025 | Jun 2026 | ✅ |
| Paul Kinlan (Google) | [The browser is the sandbox](https://paul.kinlan.me/the-browser-is-the-sandbox/) | Jan 2026 | Jun 2026 | ✅ |

---

## Diagrams

All diagrams are source-controlled D2 files in `diagrams/src/`, rendered to SVG in `diagrams/svg/`. To re-render:

```bash
cd reference-llm/diagrams
d2 src/foo.d2 svg/foo.svg
```

Color convention (consistent with `../reference/`):
- **green** = good / correct flow
- **blue** = neutral / current state
- **yellow/orange** = warning / transition / cost
- **red** = failure / forbidden
- **purple** = note / annotation

---

## License

CC BY 4.0 — © Ashutosh Sanzgiri 2026. Reuse with attribution.

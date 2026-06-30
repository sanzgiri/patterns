# LLM Observability

**Aliases:** LLM tracing, GenAI observability, agent telemetry, LLM monitoring, LLM ops, prompt analytics
**Category:** Production Engineering
**Sources:**
[LangSmith documentation](https://docs.smith.langchain.com/) ·
[Braintrust documentation](https://www.braintrust.dev/docs) ·
[Helicone documentation](https://docs.helicone.ai/) ·
[Langfuse documentation](https://langfuse.com/docs) ·
[Phoenix (Arize) documentation](https://docs.arize.com/phoenix) ·
[OpenTelemetry GenAI semantic conventions (2024-2025)](https://opentelemetry.io/docs/specs/semconv/gen-ai/) ·
production playbooks: Eugene Yan, Hamel Husain, Chip Huyen

---

## Problem

> [!TIP]
> **ELI5.** LLM apps fail *silently*. A traditional service throws errors when it breaks — your monitoring catches HTTP 500s, slow responses, exceptions. An LLM app's failures look like *plausible-sounding wrong outputs*: hallucinated facts, missed tool calls, subtle quality degradation, increased refusals. Without specific LLM observability, you find out from user complaints days later. **LLM observability** instruments the things that matter for LLM apps: full prompts/responses (with PII redaction), token counts, costs, tool calls, eval scores, cache hits, user feedback, multi-step trace trees. The tooling ecosystem (LangSmith, Braintrust, Helicone, Langfuse, Phoenix, W&B Weave) has consolidated around a few clear patterns by 2026, plus an emerging standard (OpenTelemetry GenAI semantic conventions) that lets you mix-and-match. Without it, you're flying blind; with it, you can debug, optimize cost, detect regressions, and improve quality.

The need emerged sharply in 2023 as teams shipped LLM apps and discovered traditional APM tools (Datadog, New Relic, Honeycomb) weren't built for the LLM use case. Specific issues:

- HTTP-level monitoring tells you the API succeeded but not whether the response was good.
- Token counts and cost aren't first-class metrics.
- Multi-step traces (LLM → tool → LLM → tool → ...) aren't well-represented in traditional spans.
- Per-call inputs and outputs are too large/sensitive for default APM log destinations.
- Eval scores aren't a natural APM concept.

A new tooling category emerged. LangSmith (LangChain, 2023), Braintrust (2023), Helicone (2023), Langfuse (open-source, 2023), Phoenix (Arize, open-source), and Weights & Biases Weave (2024). Datadog and others added LLM-specific products in 2024-2025.

By 2026, OpenTelemetry has standardized the schema (GenAI semantic conventions), enabling vendor-agnostic instrumentation.

## How it works

> [!TIP]
> **ELI5.** Wrap every LLM call (and tool call, and sub-call) in a tracing span. Capture the full prompt, response, model, tokens, cost, latency, parameters, tool calls made, cache hit, eval score (if any), and user feedback signal (if/when it arrives). Spans link into a trace tree so a multi-step session is one navigable timeline. Send that to a tracing service. Now you can debug ("why did the agent do X?"), analyze cost ("which prompt is the cost driver?"), detect regressions ("did v4 of the prompt degrade?"), and learn from real traffic.

![LLM observability](../diagrams/svg/llm-observability.svg)

### What to instrument — per LLM call

For every individual LLM call, capture:

- **Full prompt** (system + user + tool definitions, with PII redaction)
- **Full response** (assistant message, tool calls)
- **Model** (vendor, name, version: e.g. `anthropic/claude-sonnet-4-5`)
- **Parameters** (temperature, max_tokens, top_p, etc.)
- **Token counts** (input, output, cached input, cached creation)
- **Cost** (computed from token counts × current pricing)
- **Latency** (time-to-first-token + total)
- **Cache metadata** (hit/miss, cached prefix length)
- **Tool calls emitted** (name, arguments, result, latency)
- **Eval score** (if a grader ran)
- **Error / refusal / blocked** flag
- **Streaming events** (chunk timestamps for diagnosing slowness)

### What to instrument — per session / trace

A session is many LLM calls plus tool calls plus business logic. Capture:

- **Trace tree** (parent-child span relationships)
- **Total session cost** (sum across all LLM calls)
- **Total session latency** (wall-clock for the user)
- **Total tokens** by tier (brain vs hands)
- **User identifier** (hashed for privacy)
- **Session identifier** (correlation across calls)
- **Variant assignment** (which A/B variant — see [rainbow deployments](rainbow-deployments.md))
- **User feedback signal** (thumbs up/down, escalation, abandon)
- **Outcome** (task completed, error, user gave up)
- **Custom business metadata** (tenant, plan tier, route, etc.)

### Why trace trees matter

A typical agent session looks like:

```
Session [42s, $0.18]
├── Plan [4s, $0.05]
│   └── LLM call: brain model, 8K tok, 3.5s
├── Tool: search_docs [2s]
├── Synthesize [3s, $0.02]
│   └── LLM call: hands model, 12K tok, 2.8s
├── Tool: write_file [400ms]
├── Verify [5s, $0.04]
│   └── LLM call: brain model, 6K tok, 4.8s
├── Tool: run_tests [25s]  ← bottleneck visible here
└── Reflect [3s, $0.07]
    └── LLM call: brain model, 9K tok, 2.9s
```

This visualization is invaluable for debugging. Without it, you have a flat list of LLM API calls and can't see causality.

### Why "full prompt + full response" matters

You'll want to:
- Reproduce a bad output by re-running the exact prompt.
- Show users (or auditors) the exact reasoning behind a decision.
- A/B test prompt variations (need to know what was actually sent).
- Detect prompt-injection attacks (search prompts for suspicious patterns).
- Compute new metrics retrospectively (run a new grader on past outputs).

Without full capture, all of this is impossible.

### PII redaction is non-negotiable

You cannot log raw prompts containing PII to a third-party SaaS. You need a redaction layer:

- Replace email addresses with `[EMAIL_1]`, `[EMAIL_2]`, ...
- Replace phone numbers with `[PHONE_*]`
- Replace credit card numbers with `[CC]`
- Replace names (if detectable) with `[PERSON_*]`
- Redact based on regex + an NER classifier
- Keep a separate, more-restricted store for non-redacted versions if needed for debugging

Tools like Helicone offer built-in redaction. Custom builds need it.

### The OpenTelemetry GenAI semantic convention (2024-2025)

OpenTelemetry standardized LLM observability:

- Span attributes: `gen_ai.system`, `gen_ai.request.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, etc.
- Span events: `gen_ai.content.prompt`, `gen_ai.content.completion`
- Conventions for tool calls, embeddings, agents

Implication: instrument once using OpenTelemetry SDK; ship to *any* compatible backend (Datadog, Honeycomb, Phoenix, Jaeger, custom). Vendor lock-in dramatically reduced.

By 2026 most tools accept OpenTelemetry GenAI spans natively.

### Vendor landscape (2026)

| Tool | Strengths | When to pick |
|---|---|---|
| **LangSmith** | LangChain-native; rich eval features | LangChain/LangGraph apps |
| **Braintrust** | Best-in-class eval / experiment workflows | Eval-heavy teams |
| **Helicone** | Drop-in proxy; great cost analysis | Cost optimization focus |
| **Langfuse** | Open-source; full-featured | Self-host requirement |
| **Phoenix** | Open-source; great trace UI | Self-host requirement |
| **W&B Weave** | Integrates with broader W&B MLOps | Existing W&B users |
| **Datadog LLM** | Integrates with existing Datadog | Already on Datadog |
| **Arize AI** | Production monitoring + drift detection | Enterprise focus |
| **OpenAI Evals + tracing** | Vendor-native | OpenAI-only stacks |
| **Custom OpenTelemetry** | Maximum control | Special requirements |

Most production teams use one primary tracing tool plus a separate eval framework (sometimes the same product, often different).

### What the dashboards typically show

- **Cost dashboard**: total cost per day, per route, per user, per variant. Cost per outcome.
- **Latency dashboard**: p50, p95, p99 per route. Time-to-first-token vs total.
- **Quality dashboard**: eval scores trending; pass rate per route.
- **Error dashboard**: refusal rate, error rate, retry rate.
- **Cache dashboard**: hit rate per endpoint; estimated savings.
- **Variant comparison**: side-by-side variant metrics.
- **Trace browser**: drill into individual sessions; filter by user, error, eval failure.

### Detecting regressions

A key use case: prove that v4.1 of your prompt didn't break things v4.0 was doing well.

Approaches:
- **Eval score trend.** Run the same eval set after each release; alert on score drops.
- **Production traffic eval.** Sample 1% of production traffic; run a grader; compare period-over-period.
- **User feedback rate.** Thumbs-up rate is your floor metric. Watch it like an SRE watches error rate.
- **Escalation / abandon rate.** Did users give up more after the change?
- **Variant comparison** (see [rainbow deployments](rainbow-deployments.md)).

### Drift detection

Production traffic changes over time. Detect:
- New query types appearing (NLP cluster drift)
- Cost-per-query trending up (new heavy users, longer conversations)
- Tool call distribution shifting (users discovering new flows)
- Cache hit rate dropping (more diverse queries)

Drift detection lets you proactively adjust before quality regresses.

### Engineering details

- **Sampling rate**: full capture is expensive and possibly privacy-sensitive. Sample (e.g., 100% for errors, 10% for normal).
- **Async write**: instrument shouldn't slow user-facing requests. Buffer + async ship to backend.
- **Retention policy**: align with privacy/compliance. 90 days typical for traces; longer for aggregated metrics.
- **Cost of observability itself**: Helicone-style proxies add 50-100ms; eval calls cost real money.
- **Multi-tenant isolation**: traces from tenant A must not appear in tenant B's dashboards.
- **Cardinality control**: don't tag spans with high-cardinality fields (full user IDs) blindly.
- **Long-running traces**: agent sessions can run minutes-to-hours; backend must handle.
- **Embedding observability**: capture per-call embedding cost and dimensionality too.

### Anti-patterns

- **No full-prompt capture.** You can't debug what you can't see.
- **No PII redaction.** Compliance violation, breach risk.
- **Token counts not tracked.** No cost optimization possible.
- **Synchronous trace shipping.** Slows user-facing requests.
- **Trace cardinality explosion.** High-cardinality tags overwhelm the backend.
- **No alerting.** You spot regressions only when you happen to look.
- **No correlation between traces and user feedback.** Can't tell which sessions users liked.
- **Sampling that drops errors.** Always log errors at 100%.
- **Logs in plain text dumped to one big file.** Useless for analysis.
- **Different schemas per service.** Cross-service traces don't link.

### How observability changes the team

A team with mature LLM observability operates fundamentally differently:

- **Cost meetings** look at dashboards, not gut feel.
- **Quality debates** reference eval scores + production samples, not anecdotes.
- **Prompt changes** ship with experiment plans, not "I think this is better."
- **Incidents** get resolved by replaying the trace, not guessing.
- **Capacity planning** uses real per-route token telemetry.
- **Customer support** can answer "why did the AI say that?" by pulling the trace.

This is the "DevOps for LLM" maturity curve — observability is the floor.

## Variants & related patterns

- [**Harness design**](harness-design.md) — the harness emits the spans.
- [**Rainbow deployments**](rainbow-deployments.md) — variant ID is a span attribute.
- [**12-Factor Agents**](12-factor-agents.md) — Factor 12 (stateless reducer) makes tracing easier.
- [**Eval-driven development**](../qua/eval-driven-development.md) — eval scores are a span attribute.
- [**Grader taxonomy**](../qua/grader-taxonomy.md) — graders write to spans.
- [**Brain vs hands**](brain-vs-hands.md) — segment metrics by tier.
- [**Prompt caching**](../mem/prompt-caching.md) — cache hits are span attributes.
- **OpenTelemetry** — the standard tracing protocol.
- **Distributed tracing** (system design) — the pattern's ancestor.

## When NOT to use

- **Prototypes** where you're throwing away code daily.
- **Tiny apps** with one trusted user.
- **Offline batch jobs** where there's no production traffic to observe.
- **Strict-air-gapped deployments** where SaaS observability isn't allowed (use OSS tools instead).

## Implementations

| Tool | Type | Hosting |
|---|---|---|
| **LangSmith** | Commercial SaaS (free tier) | Hosted (self-host available) |
| **Braintrust** | Commercial SaaS | Hosted (self-host available) |
| **Helicone** | Commercial SaaS + OSS | Hosted or self-host |
| **Langfuse** | OSS + commercial | Hosted or self-host |
| **Phoenix (Arize)** | OSS | Self-host (Arize hosted is separate) |
| **W&B Weave** | Commercial | Hosted (W&B platform) |
| **Datadog LLM Observability** | Commercial | Datadog |
| **Honeycomb** | Commercial APM | Honeycomb |
| **OpenTelemetry SDK + any backend** | OSS | Any |
| **Sentry AI** | Commercial APM | Sentry |
| **OpenInference (Arize)** | OSS instrumentation library | Library only |

## Companies / products that publicize their observability practice

- **LangChain (LangSmith)** ✅ — productize observability.
- **Anthropic** ⚠ — internal extensive tracing.
- **OpenAI** ⚠ — internal observability + customer Evals API.
- **Helicone, Braintrust customers** ✅ — paying users.
- **Notion AI, Linear AI** ⚠ — public posts mention rigorous tracing.
- **Cursor, Continue, Cody** ⚠ — extensive internal observability.
- **GitHub Copilot** ⚠ — extensive Microsoft-grade observability.
- **Most data-driven LLM teams** ⚠ — adopt within months of production.

## Further reading

- [LangSmith documentation](https://docs.smith.langchain.com/)
- [Braintrust documentation](https://www.braintrust.dev/docs)
- [Helicone docs](https://docs.helicone.ai/)
- [Langfuse docs](https://langfuse.com/docs)
- [Phoenix docs](https://docs.arize.com/phoenix)
- [OpenTelemetry GenAI semantic conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/)
- [Patterns for Building LLM-based Systems & Products](https://eugeneyan.com/writing/llm-patterns/) — Eugene Yan
- [What we learned from a year of building with LLMs (Part II)](https://www.oreilly.com/radar/what-we-learned-from-a-year-of-building-with-llms-part-ii/) — production
- [OpenInference](https://github.com/Arize-ai/openinference) — OSS LLM instrumentation library

---

*Diagram source: [`../diagrams/src/llm-observability.d2`](../diagrams/src/llm-observability.d2)*

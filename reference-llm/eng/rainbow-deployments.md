# Rainbow Deployments (for LLM Apps)

**Aliases:** parallel-variant deployment, multi-version rollout, prompt/model A/B testing, prompt experimentation, agent canary
**Category:** Production Engineering
**Sources:**
[Braintrust documentation on prompt versioning](https://www.braintrust.dev/docs) ·
[LangSmith experiments documentation](https://docs.smith.langchain.com/old/evaluation) ·
[Weights & Biases Weave prompt experimentation](https://wandb.ai/site/weave) ·
inspired by blue-green and canary deployments (system design `ops/blue-green-canary`) ·
related: feature flags, multi-armed bandits

---

## Problem

> [!TIP]
> **ELI5.** Changing an LLM app — a new prompt, a new model version, a new tool, a different chunk size — is *risky*. The quality impact is hard to measure pre-deployment because LLM behavior is probabilistic and context-dependent. Eval sets help, but real users find edge cases evals miss. **Rainbow deployment** is the LLM-adapted version of canary deploys: run multiple variants of your agent IN PRODUCTION SIMULTANEOUSLY, each on a slice of real traffic. Collect per-variant metrics (cost, latency, eval score, user feedback). Promote the winner; roll back the loser. Standard practice for any team that's been burned by a "looked fine in eval, broke in prod" release.

The pattern is named "rainbow" because you typically run more than two variants at once (blue/green is only two; rainbow could be 5+ in parallel), each "colored" with metadata so you can split-test results. The name comes from CI/CD culture but the LLM-specific instance has unique characteristics: the axes you vary (prompts, models, tool sets, retrieval configs, temperatures) are different from traditional deploys (binaries, container images, schemas).

Why LLM apps need this more than traditional services:

- **Quality is hard to pre-measure.** Eval sets catch known failure modes; real users find unknown ones.
- **Behavior depends on inputs you don't control.** Same prompt + same model + same user input → different outputs across runs.
- **Subtle regressions are common.** A prompt tweak that improved 10 evals might silently degrade 100 production conversations.
- **Cost and quality trade off non-linearly.** Cheaper model X might match Y on 95% of traffic but degrade catastrophically on the 5% that mattered.

A rainbow deployment lets you ship changes safely AND learn about your traffic distribution at the same time.

## How it works

> [!TIP]
> **ELI5.** Define K variants: each is a (prompt, model, tools, RAG config) tuple. Route, say, 70% of traffic to the current production version (control), 10% to each of three experimental variants. Tag every LLM call with its variant ID so metrics segment by variant. Daily (or in real time), look at per-variant cost / latency / eval score / user feedback. A clear winner gets promoted (its traffic share grows); clear losers get rolled back (their share goes to zero). Repeat indefinitely.

![Rainbow deployments](../diagrams/svg/rainbow-deployments.svg)

### What you can vary across variants

Every dimension of your LLM stack is a candidate for variant testing:

- **System prompt** (the most common variant axis)
- **Model** (Sonnet 4 vs Opus 4 vs GPT-5 vs Gemini Ultra)
- **Model version** (Sonnet 4.0 vs Sonnet 4.5)
- **Temperature / sampling parameters**
- **Tool definitions** (signature changes, new tools)
- **RAG retrieval config** (chunk size, top-K, reranker on/off)
- **Memory architecture** (compaction strategy, episodic vs not)
- **Number of few-shot examples**
- **Output schema** (different structured-output formats)
- **Reasoning mode on/off** (Claude extended thinking, etc.)
- **Embedding model** (Cohere v3 vs Voyage v2)
- **Reranker** (BGE vs Cohere vs Voyage)

Typically you vary ONE axis per deployment to keep results interpretable. Multi-axis variants are valid but harder to attribute wins to.

### Traffic routing strategies

**1. Random allocation.** Each call independently assigned to a variant by random sampling. Best for high-volume traffic where you need quick statistical power.

**2. User-stable allocation.** Hash the user ID to a variant; that user always sees the same variant. Better for measuring user-level effects (engagement, retention).

**3. Conversation-stable allocation.** A user's session sticks to one variant throughout. Avoids "the agent changed personality mid-conversation" experience.

**4. Multi-armed bandit.** Algorithmically shift traffic toward winners as evidence accumulates. Self-tuning but harder to A/B with traditional stats.

**5. Targeted allocation.** Send specific cohorts (paying customers vs free; enterprise vs SMB; specific use cases) to specific variants. Useful for premium-vs-cheap experiments.

### What to measure per variant

| Metric | Why it matters |
|---|---|
| **Cost per query** | Direct $$ comparison |
| **Latency p50 / p95** | User-perceived speed |
| **Eval score** | Quality on a static set |
| **User feedback** (thumbs up/down, retention) | Quality on real traffic |
| **Tool call success rate** | Functional reliability |
| **Schema validation rate** | Structured-output reliability |
| **Error / refusal rate** | Failure floor |
| **Cache hit rate** | Cost amplifier |
| **Escalation rate** | When agent gives up |
| **Downstream conversion** | Business outcome |

For each variant, you want statistical confidence on these metrics before declaring a winner. Some metrics (cost, latency) reach confidence quickly; some (user retention) need weeks of data.

### Variant rollout pattern

A typical rollout sequence for a new prompt:

1. **Internal-only**: 100% of internal-team traffic to variant; 0% production.
2. **Smoke test**: 1-5% of production traffic for 24 hours. Watch error rates.
3. **Early signal**: 10-20% for several days. Compare core metrics.
4. **Significance**: 30-50% until statistical significance on key metrics.
5. **Full promotion**: 100% if winning; 0% if losing.
6. **Cleanup**: remove the old variant after promotion is stable.

This is essentially canary deployment (see system-design `ops/blue-green-canary`) adapted to LLM specifics (the "build" being changed is the prompt + model, not a container image).

### Per-variant cost accounting

A critical piece of telemetry: how much does each variant cost?

- Cost per call, attributed by variant ID
- Total cost per variant per day
- Cost per outcome (per successful conversation, per business goal)

You can't pick a winner just on quality — a 2% quality improvement isn't worth a 3× cost increase in most contexts. Rainbow deployments make this trade-off visible.

### Engineering details

- **Variant ID in every log entry.** Without it, you can't segment metrics by variant.
- **Variant ID in user-facing telemetry** (when you ask for feedback). Match the thumbs-up to the variant.
- **Reproducibility**: a variant config should be a versioned, named entity (e.g., `v2025-03-15-promptA-sonnet45-temp0.2`).
- **Reverse rollback**: shutting off a variant should be one click / one flag flip.
- **Per-variant cost cap**: a leak in one variant shouldn't bankrupt you.
- **Per-variant guardrails**: an experimental variant might need stricter safety filtering.
- **Significance testing**: don't promote on day 1 noise. Wait for confidence.
- **Variant exposure window**: typically 1-4 weeks per variant before decision.
- **Long-tail evaluation**: some variants regress on rare query types; need long enough exposure to detect.

### Common variant tests

**1. New model version.** "Does Claude Sonnet 4.5 actually improve our use case?" Run 50/50 with Sonnet 4.0 for a week.

**2. Prompt rewrite.** "Does this clearer prompt produce better outputs?" 80/20 with current prompt.

**3. Tool addition.** "Does adding a calendar tool reduce escalations?" 50/50 with current tool set.

**4. RAG config change.** "Does increasing top-K from 5 to 10 help?" 50/50 with current config.

**5. Cheaper-model substitution.** "Can Haiku replace Sonnet here?" 10/90 to start; expand if quality holds.

**6. Memory architecture change.** "Does adding episodic memory improve session quality?" 30/70.

**7. Different reranker.** "Cohere Rerank vs Voyage Rerank — which fits our content?" 50/50.

### When NOT to use rainbow deployments

- **Tiny traffic.** Statistical significance is impossible at 10 queries/day.
- **One-off batch jobs.** Run all variants on the eval set offline.
- **Strict consistency required.** Some compliance regimes require deterministic per-user behavior.
- **Cost-extreme paths** where running K variants is itself the cost issue.
- **Without per-variant telemetry.** Don't run experiments you can't measure.

### Anti-patterns

- **Too many variants at once.** Each gets too little traffic; nothing reaches significance.
- **Multi-axis variants with no isolation.** Can't tell which change drove the result.
- **No variant exposure window discipline.** Promoting on day 1 noise.
- **Cherry-picking metrics.** Variant A wins on cost; variant B on quality. Pick the winner you like, ignore the other metric.
- **No rollback capability.** Variant B is bad; you can't shut it off cleanly.
- **Variant ID missing from logs.** Can't slice metrics; experiment is unusable.
- **Same variant ID across deploys.** Confounds old and new attempts.
- **Hidden variant impact.** Variant A affects downstream caching, distorting other variants' metrics.
- **Skipping internal-only and smoke test phases.** Catastrophic variants reach users.

### How rainbow deployment changes how you build

Once a team has rainbow deployment infrastructure, the workflow shifts:

- **Prompt edits become experiments.** Every prompt change is a variant; PR includes the experiment plan.
- **Model upgrades are routine.** No more big-bang model swaps; gradual ramp.
- **Reverse migrations are easy.** When you regret a change, roll it back via variant share.
- **Eval sets are validated by reality.** Variants that look good in evals but regress in production teach you to improve evals.
- **Cost optimization is data-driven.** "Can we save 30% with Haiku here?" → measurable experiment, not opinion.

This is one of the highest-leverage investments for a production LLM team.

## Variants & related patterns

- [**Harness design**](harness-design.md) — the harness implements variant routing.
- [**LLM observability**](llm-observability.md) — provides the per-variant metrics.
- [**Eval-driven development**](../qua/eval-driven-development.md) — evals run on variants pre- and post-deploy.
- [**Brain vs hands**](brain-vs-hands.md) — tier choices are often variant-tested.
- [**12-Factor Agents**](12-factor-agents.md) — variants align with Factor 11 (trigger anywhere).
- **Blue-green / canary deployment** (system-design `ops/blue-green-canary`) — system-design ancestor.
- **Feature flags** — common implementation primitive.
- **Multi-armed bandits** — algorithmic relative.

## When NOT to use

- **Tiny traffic** (no statistical power).
- **Strict deterministic requirements** (compliance).
- **Without per-variant telemetry** (can't measure).
- **One-off jobs** (use offline evals).

## Implementations

| Tool | Variant support |
|---|---|
| **LangSmith** | Experiments + prompt versioning |
| **Braintrust** | Strong native variant testing |
| **Weights & Biases Weave** | Prompt experimentation + tracing |
| **Langfuse** | Prompt versioning + experiments |
| **Helicone** | Cost-segmented variant analysis |
| **Phoenix (Arize)** | Trace-based variant comparison |
| **LaunchDarkly + custom** | Feature flag → variant assignment |
| **Statsig** | Experimentation + LLM-specific support |
| **OpenRouter, Portkey** | Routing infrastructure for variants |
| **Custom** | Most teams build a thin variant assignment layer |

## Companies / products with rainbow deployment practice

- **Anthropic** ⚠ — internal experimentation on prompts and models.
- **OpenAI** ⚠ — ChatGPT model routing is essentially a giant rainbow.
- **GitHub Copilot** ⚠ — multi-tier model rollouts.
- **Cursor** ⚠ — model A/B tests visible in changelog.
- **Stripe Sigma + AI products** ⚠ — financial-product rigor.
- **Notion AI, Linear AI** ⚠ — prompt experimentation programs.
- **Glean, Vectara** ✅ — productized variant testing for customers.
- **Braintrust customers** ✅ — the tool is built for this.
- **OpenAI ChatGPT auto-model selection** ✅ — public-facing example.
- **Most data-driven LLM teams** ⚠ — converge on this practice within months of going to production.

## Further reading

- [Braintrust docs on experiments](https://www.braintrust.dev/docs)
- [LangSmith experiments](https://docs.smith.langchain.com/old/evaluation)
- [Weights & Biases Weave](https://wandb.ai/site/weave)
- [Patterns for Building LLM-based Systems & Products](https://eugeneyan.com/writing/llm-patterns/) — Eugene Yan
- [What we learned from a year of building with LLMs](https://www.oreilly.com/radar/what-we-learned-from-a-year-of-building-with-llms-part-i/) — evaluation patterns
- [Blue-green canary deployment] (system-design `ops/blue-green-canary`) — system-design precedent
- [Feature flag patterns] (system-design `ops/feature-flag`) — implementation primitive

---

*Diagram source: [`../diagrams/src/rainbow-deployments.d2`](../diagrams/src/rainbow-deployments.d2)*

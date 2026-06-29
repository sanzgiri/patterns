# Pass@k vs Pass^k

**Aliases:** pass-at-k vs pass-to-the-power-k, sample-best vs all-runs-must-pass, capability-pass vs reliability-pass
**Category:** Quality & Evals
**Sources:**
[Anthropic — Demystifying evals for AI agents (Jan 2026)](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) ·
[Chen et al. — Evaluating Large Language Models Trained on Code (HumanEval paper, 2021)](https://arxiv.org/abs/2107.03374) (original pass@k) ·
[SWE-bench paper](https://arxiv.org/abs/2310.06770) ·
production discussion: [LMSys, METR, Cognition engineering blogs](https://www.cognition.ai/blog)

---

## Problem

> [!TIP]
> **ELI5.** Your agent solves the same task 4 times in a row — and gets it right twice, wrong twice. Is that a *good* result or a *bad* one? It depends on who's using the agent. A **research benchmark** treats it as good — "yes, the model is *capable* of solving this." A **production deployment** treats it as bad — "this would fail half the time for real users." Those two answers correspond to two different metrics: **pass@k** (any of K runs succeeded) and **pass^k** (all K runs succeeded). Picking the wrong one for your context will lie to you about how well your system actually works.

The early LLM-as-coder benchmarks — HumanEval, MBPP, APPS — measured "pass@k": with K independent samples from the model, does *any one* of them pass the test cases? This was reasonable for the research question of the time: *can the model produce correct code at all?* If it can do it 1-in-100 tries, the capability exists; with cherry-picking and retry, you can extract that capability.

By 2025-2026, with agents shipping to production, the question changed. Real users don't have an oracle to pick the best of 100 attempts. They run the agent once. If it works, great; if it doesn't, they're frustrated, lose trust, and churn. The relevant metric became **pass^k** — out of K independent runs on the same task, do *all of them* succeed? That measures *reliability*, not capability.

Anthropic's [January 2026 evals post](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) made this distinction explicit and pushed for pass^k as the default production metric. The shift matters because a system with pass@10 = 95% and pass^10 = 30% is a system that *appears* almost perfect on benchmarks and is *actually* unreliable in production. The gap between the two numbers is the gap between research demos and production users.

## How it works

> [!TIP]
> **ELI5.** Run the agent K times on the same task. **pass@k** = "did at least one succeed?" (lenient — counts as 1 if any pass). **pass^k** = "did ALL K succeed?" (strict — counts as 1 only if every pass). The same agent on the same task can have pass@k near 100% and pass^k near 0% — the gap tells you how reliable it is, not just how capable.

![pass@k vs pass^k](../diagrams/svg/pass-at-k-vs-pass-power-k.svg)

### The definitions, carefully

For a single task, run the agent **K** independent times. Each run produces success (1) or failure (0). Let `c` be the number of successes out of K runs.

**pass@k** = 1 if `c ≥ 1`, else 0 — "at least one of K runs succeeded."

**pass^k** (sometimes written `pass^K`) = 1 if `c = K`, else 0 — "all K runs succeeded."

A single task → a single 0/1 per metric. Average over many tasks to get the eval score.

A more general definition of pass@k (the original Chen et al. formulation) is an *expectation*: for n samples and a target of k retries, what's the probability that at least one of k randomly chosen attempts passes? That formulation lets you estimate pass@k from a larger sample of n runs. In agentic-eval practice, the simpler "K runs, at least one passed" definition is what's used.

### What the gap between them means

For the same task, with the same agent, `pass@k ≥ pass^k` always. The interesting quantity is the *gap*:

- If `pass@k ≈ pass^k`, the agent is either always-passing or always-failing on this task. Reliable (in either direction).
- If `pass@k >> pass^k`, the agent is sometimes-passing-sometimes-failing — high variance, low reliability.

This gap is exactly what production deployment cares about. A task where the agent might succeed or fail unpredictably is a task users won't trust the agent on. Even if pass@k is 90%, if pass^k is 20%, users will perceive a 70% failure rate.

### Why the field shifted from pass@k to pass^k

Through 2021-2023, pass@k was the default. Why?

- **Most early benchmarks were single-LLM-call** (HumanEval, MBPP). Variance was about temperature sampling, not agent variance.
- **The research question was capability** — does this model *know how* to write correct code?
- **Compute was tight** — running the same task 100 times to measure variance was expensive.
- **Production wasn't the focus** — most papers measured pure model behavior, not deployed systems.

By 2024-2026:
- **Agentic systems multiplied variance.** Tool calls, sub-agents, search results, MCP state — all introduce non-determinism beyond temperature sampling.
- **Production deployment matters.** SWE-bench-verified leaderboard numbers were being cited as "this agent solves X% of bugs" — but those numbers were typically pass@k with K > 1, hiding the unreliability.
- **Cognition (Devin) published transparently** about pass^k for their long-horizon tasks, demonstrating how much pass@1 vs pass@8 differed.
- **Anthropic's 2026 evals post** formalized the recommendation.

### The "5 nines" framing

A useful intuition: pass^k of a system at K runs is related to single-run reliability. If pass@1 (= single-run success rate) is `p`, then naively pass^k = `p^k`.

| Single-run reliability (pass@1) | pass^10 | pass^100 |
|---|---|---|
| 50% | 0.1% | ~0% |
| 80% | 11% | ~0% |
| 90% | 35% | 0.003% |
| 95% | 60% | 0.6% |
| 99% | 90% | 37% |
| 99.9% | 99% | 90% |

For an agent that gets used 100 times by 100 users in a day, even 95% single-run reliability means most of those 100 users see a failure that day. To achieve "5 nines" reliability (99.999%) per attempt — telecom-grade — across many tasks requires extraordinarily high single-run pass rates.

This is why production AI engineering increasingly cares about getting pass@1 from 80% → 95% → 99% rather than chasing benchmark capability gains. The economic value of reliability compounds.

### Variants and related metrics

- **pass@1.** Single-run success rate. The most user-facing metric.
- **pass@k for k > 1.** Useful for "agentic ensembles" or retry-tolerant scenarios.
- **pass^k.** Strict reliability.
- **maj@k** ("majority at k"). Take the most common answer across k runs. Used for self-consistency.
- **best@k.** Like pass@k but with an explicit oracle picking the best output.
- **Avg score over k.** For continuous metrics (BLEU, ROUGE, model-graded scores), not just binary.

### When you genuinely want pass@k

Not always wrong — there are real situations:

- **Capability ceiling research.** Can this model do X *at all*? pass@k with large K answers that question.
- **Agent ensembles.** If your production system runs K agents in parallel and an oracle picks the best, pass@k *is* the right metric.
- **High-budget agent search.** Devin-style long-horizon agents may legitimately retry on internal failures; pass@k measures the *system*, not just the single attempt.
- **Generation followed by verification.** If a verifier filters K outputs and picks the passing one, pass@k is what matters.
- **Cherry-picked benchmark demos.** Not legitimate, but common; explains why public benchmark numbers often look better than production behavior.

### When pass^k is what you need

Almost everything user-facing:

- **Interactive coding assistants.** Users run once; failure is failure.
- **Customer-service agents.** Users won't tolerate "retry 8 times to get a good answer."
- **Compliance / regulated decisions.** Reliability is the entire game.
- **Long-horizon autonomous agents.** Internal-retry helps, but cross-task pass^k catches systemic failure modes.

A useful rule of thumb: if your product has a "regenerate" button or oracle selection, pass@k is relevant. If your users see a single output and judge it on that, pass^k is what matters.

### Reporting both is best

The most honest benchmark reports include both. SWE-bench-verified leaderboards in late 2025 started including both pass@1 and pass^k results; Anthropic's research papers often report multiple K values. Production teams typically track pass@1 (closest to user experience), pass^k for some K (reliability stress test), and capability pass@k for capability-ceiling tracking.

The danger is reporting one without the other. Pass@k alone overstates production reliability; pass^k alone undersells capability. The pair tells the full story.

## Variants & related patterns

- [**Eval-driven development**](eval-driven-development.md) — the methodology these metrics serve.
- [**Grader taxonomy**](grader-taxonomy.md) — what produces the 0/1 per run.
- [**Capability vs Regression evals**](capability-vs-regression-evals.md) — which metric to use depends on eval type.
- [**End-state evaluation**](end-state-evaluation.md) — how to grade agentic traces consistently across K runs.
- [**Eval saturation**](eval-saturation.md) — pass@k saturates faster than pass^k as models improve.
- **Self-consistency / maj@k** — voting variant.
- **best-of-N sampling** — closely related at inference time.

## When NOT to use

**Avoid pass@k for production-reliability claims** — it inflates the perceived quality of unreliable systems.

**Avoid pass^k** for pure capability research — it undersells what a model *can* do.

**Avoid either alone** in published benchmarks. Always report both with the K value clear.

**Avoid pass^k with very large K** when intermediate randomness is genuinely irreducible (e.g., external API calls that have inherent flakiness independent of the agent). In those cases, separate the agent's reliability from the system's reliability.

## Implementations

| Tool / framework | Native support |
|---|---|
| **Anthropic Workbench** | Both metrics, reported alongside |
| **OpenAI Evals** | pass@k supported; pass^k via custom aggregator |
| **Promptfoo** | Both via assertion configurations |
| **Inspect AI** | Both via `sample_count` + custom scorers |
| **SWE-bench harness** | pass@1 by default; pass@k via `--num_evals N` |
| **HumanEval / MBPP** | Original pass@k formulation |
| **METR Task Suite** | Multi-run evaluation with both metrics |
| **LiveCodeBench** | pass@1 primary; multi-run available |
| **Custom: any framework + numpy.mean over K runs** | Easy to roll your own |

## Companies / projects using these metrics

- **Anthropic** ✅ — formalized the recommendation ([source](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)).
- **OpenAI** ✅ — HumanEval paper introduced pass@k formulation ([source](https://arxiv.org/abs/2107.03374)).
- **Cognition (Devin)** ✅ — pass^k reported transparently in their SWE-bench results.
- **METR (Model Evaluation and Threat Research)** ✅ — multi-K evaluation for capability and safety.
- **UK AISI (Inspect AI)** ✅ — multi-sample evals for safety / capability assessments.
- **GitHub (Copilot)** ⚠ — internal reliability tracking referenced in engineering posts.
- **Cursor, Replit, Aider** ⚠ — track pass^k-like reliability metrics internally.
- **OpenAI / Google / Anthropic benchmark leaderboards** ✅ — increasingly report both.

## Further reading

- [Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) — Anthropic Jan 2026 (the 2026 framing)
- [Evaluating Large Language Models Trained on Code](https://arxiv.org/abs/2107.03374) — Chen et al. 2021 (original pass@k formulation, HumanEval paper)
- [SWE-bench: Can Language Models Resolve Real-World GitHub Issues?](https://arxiv.org/abs/2310.06770) — Jimenez et al. 2023
- [Cognition engineering blog](https://www.cognition.ai/blog) — Devin's transparent pass^k discussions
- [METR's task evaluation methodology](https://metr.org/) — multi-run safety/capability assessments
- [Self-consistency improves chain-of-thought reasoning](https://arxiv.org/abs/2203.11171) — Wang et al. 2022 (maj@k variant)

---

*Diagram source: [`../diagrams/src/pass-at-k-vs-pass-power-k.d2`](../diagrams/src/pass-at-k-vs-pass-power-k.d2)*

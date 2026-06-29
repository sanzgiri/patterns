# Capability vs Regression Evals

**Aliases:** ceiling evals vs floor evals, exploratory vs gate evals, frontier evals vs CI evals
**Category:** Quality & Evals
**Sources:**
[Anthropic — Demystifying evals for AI agents (Jan 2026)](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) ·
[Hamel Husain — Your AI Product Needs Evals](https://hamel.dev/blog/posts/evals/) ·
[OpenAI Evals documentation](https://github.com/openai/evals) ·
production practice: GitHub Copilot, Cursor, Anthropic engineering posts

---

## Problem

> [!TIP]
> **ELI5.** Two different questions, two different types of eval. **Capability eval**: "Can my system handle *new and harder* cases — should I invest more here?" **Regression eval**: "Did the change I just made *break* anything that was working before?" Teams that conflate them end up using a small capability suite as their CI gate (and shipping regressions on tasks not in the suite) or using a giant regression suite as their capability tracker (and never noticing when the *ceiling* of what the system can do is rising or falling). Mature teams run both, on different cadences, with different content.

By 2025-2026 the practitioner consensus split evals into two categories that serve fundamentally different purposes. Anthropic's [January 2026 evals post](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) makes the distinction explicit. Teams that treat all evals as one big bucket either:

- **Under-test the floor**: ship regressions because their CI eval only covered the "interesting" cases.
- **Over-test the ceiling**: spend compute running hard tasks every PR when most PRs don't move them.
- **Miss capability gains**: when a model upgrade unlocks new capabilities, a regression-only eval suite doesn't show it.
- **Miss safety drift**: when behavior subtly degrades on previously-working flows, a capability-only suite doesn't catch it.

The fix is to run *both* types, but with different content, cadences, and signals.

## How it works

> [!TIP]
> **ELI5.** Build two eval suites. **Capability evals** = a small set of hard, novel, or research-relevant tasks. Run weekly or on model upgrades. The metric you want to see *increase*. **Regression evals** = a larger set of currently-working production-typical tasks. Run on every PR. The metric you want to see *not decrease*. Different content, different cadence, different alarms.

![Capability vs Regression evals — two different jobs](../diagrams/svg/capability-vs-regression-evals.svg)

### Capability evals — the ceiling

A **capability eval** asks: *can the system handle this task at all?* The content is deliberately difficult — at or beyond the edge of what the current system reliably does. The goal is to *raise* the score over time.

Characteristics:
- **Small** (often 20-100 examples). You're not seeking statistical power on many cases; you're probing the frontier.
- **Hard or novel.** New domains, edge cases, tasks at the limit of current capability, recently-failing real-user requests.
- **Slow.** Often takes hours to run. Uses expensive judges, multi-sample evaluation, long-horizon agent trajectories.
- **Reported with pass@k** more than pass^k (you care about whether the system *can* do it, even if not always).
- **Used to decide investment.** Should we adopt the new model? Add this tool? Build this skill? Capability evals make those decisions concrete.

Run *cadence*: when a new model is released, when a major feature is added, monthly capability-review, or driven by specific investment decisions. Not every PR.

Concrete examples:
- **A coding agent team** runs SWE-bench-verified weekly as a capability eval. They want to see "we went from 65% to 71% on the verified set" as a *signal* about overall capability — not as a per-PR gate.
- **A customer-service agent team** runs a curated set of "hard tickets the human team complained about" as a capability eval. They want to track whether the agent is gradually closing the gap to human-level.
- **A research-agent team** runs known-hard research tasks where today's agent does 40-60%; they want to see that number rise as they invest.

### Regression evals — the floor

A **regression eval** asks: *did my change break anything that was working?* The content is deliberately *currently working* — tasks the system already handles correctly. The goal is to *not decrease* the score across changes.

Characteristics:
- **Larger** (hundreds to thousands of examples). You need statistical power to detect small regressions.
- **Production-typical.** Drawn from real production traffic when possible, or representative of it. Includes the "boring" cases that make up 80% of real use.
- **Fast.** Sub-minute to minutes per run. Uses cheap code graders where possible, model graders where required.
- **Reported with pass^k** (production reliability) — you don't want regression on something that worked before, even occasionally.
- **Used as a gate.** Score must not decrease (or must not decrease by more than X%) for a PR to merge.

Run *cadence*: every PR, every deploy. Some teams gate merges; others alert on regressions and require explicit ack.

Concrete examples:
- **A coding agent team** maintains 500 "always-passing" SWE-bench tasks as their regression suite. Any PR is gated on not losing more than 5 of them.
- **A customer-service agent team** keeps 1,000 "this conversation went well" production traces; the regression eval re-runs them after every prompt change.
- **A search agent team** keeps a "golden set" of known queries with expected answers; daily run alerts on changes.

### What goes in each, concretely

| Aspect | Capability eval | Regression eval |
|---|---|---|
| **Size** | 20-100 examples | 100-10,000+ examples |
| **Difficulty** | At or beyond current capability | Currently passing |
| **Source** | Novel/hard cases, edge cases, research benchmarks | Production traffic, known-good traces, golden sets |
| **Cadence** | Weekly / on model upgrade / monthly | Every PR / every deploy |
| **Cost** | Higher per run (slower, fancier judges) | Lower per run (fast, cheap judges) |
| **Metric** | pass@k, absolute score, "can we?" | pass^k, score-delta, "did we break?" |
| **Wanted signal** | Score *increasing* | Score *not decreasing* |
| **Failure response** | "Investigate, consider investing" | "Block merge or revert" |
| **Owner** | Often research / ML team | Often production engineering team |

### Why both, and why separate

The two evals catch different problems:

- A regression suite full of easy cases will be at 100% forever. It won't catch capability stagnation or notice when a model upgrade *should* have moved the needle.
- A capability suite of only hard cases won't catch the case where a prompt edit makes the agent perform worse on routine traffic.
- Treating them as one bucket means either over-running capability evals on every PR (slow, expensive) or under-running regression evals at capability cadence (slow feedback, regressions slip in).

**Anthropic's 2026 guidance**: maintain both suites separately. Different content, different cadences, different alarms.

### How they interact when changes happen

A typical model upgrade or major feature release:

1. **Capability eval** is run on both old and new system. The score should go *up* — that's the value the change is creating.
2. **Regression eval** is run on both old and new system. The score should *not go down* (or, if it does, the regression should be small, understood, and accepted by the team).
3. Both numbers go in the change announcement / release notes. "Capability +6 points, regression -1 point" tells reviewers exactly what trade is being made.

If capability rises but regression also falls significantly: you have a *capability/reliability trade-off*. Sometimes worth it (new model is just qualitatively better), sometimes not.

### Beyond two: more eval categories

In practice mature teams run more than two categories:

- **Capability evals** (ceiling)
- **Regression evals** (floor)
- **Safety evals** (refusal, jailbreak resistance, harmful output)
- **Behavioral evals** (tone, format, style consistency)
- **Latency/cost evals** (operational characteristics)
- **A/B production evals** (real-user behavior, separate from offline)

Each serves a distinct purpose and has distinct content, cadence, and gating policy. The capability/regression distinction is the most fundamental — get that one right first, then add others as the program matures.

### Common mistakes

- **Using SWE-bench as the only eval.** It's a capability benchmark. It won't catch regressions in your specific deployment.
- **Using production traffic as the only eval.** It's a regression-style signal. It won't tell you when you've capped your capability.
- **Same cadence for both.** Either capability is too slow (no signal) or regression is too slow (regressions slip in).
- **Same content for both.** The example set doing both jobs serves neither well.
- **Letting capability evals leak into training.** If you ever optimize a prompt against a capability eval, it's no longer measuring capability — it's measuring how well you tuned to it.
- **No baseline tracking for capability.** Capability evals are only useful when you compare against past runs. Track scores over time, not just current score.

### Eval rotation and freshness

A capability eval that hasn't changed in a year may have effectively been trained against. Both newer models and your own prompt iteration drift toward whatever you measured against. Mitigations:

- **Rotate capability eval content** quarterly or biannually. Retire saturated tasks; add new ones from the frontier.
- **Hold out a sub-set** that's *only* used for final go/no-go decisions, never iterated against.
- **Watch for capability/regression confusion** — a task that used to be capability-level is now regression-level if your system reliably passes it. Promote it to the regression suite.

## Variants & related patterns

- [**Eval-driven development**](eval-driven-development.md) — the methodology both fit inside.
- [**Grader taxonomy**](grader-taxonomy.md) — what runs each eval.
- [**Pass@k vs pass^k**](pass-at-k-vs-pass-power-k.md) — capability uses pass@k, regression uses pass^k.
- [**End-state evaluation**](end-state-evaluation.md) — how to grade agentic traces in both.
- [**Eval saturation**](eval-saturation.md) — capability evals saturate; need rotation.
- **A/B production testing** — the real-user analogue of regression.
- **HELM, BIG-bench, GAIA, AgentBench** — public capability benchmarks.

## When NOT to use

- **Tiny teams / prototypes.** Maintaining two eval suites is overhead. Start with one general eval, split when you outgrow it.
- **Single-purpose narrow systems.** A simple classifier doesn't need a capability suite; regression alone is enough.
- **When you can't even build one good eval yet** — start with one before splitting.

## Implementations

| Tool / framework | Two-suite support |
|---|---|
| **Anthropic Workbench** | Multiple eval suites can be configured with different schedules |
| **OpenAI Evals** | Multiple evals possible; cadence is up to your CI config |
| **Promptfoo** | Multiple configs; CI integration |
| **LangSmith** | Test sets + datasets, with scheduled vs PR-triggered runs |
| **Inspect AI** | Tasks can be tagged and run independently |
| **Braintrust, Weave, Phoenix** | Dashboards naturally accommodate multiple suites |
| **Custom CI** | Two GitHub Actions: one nightly capability, one per-PR regression |

## Companies running both

- **Anthropic** ✅ — explicit recommendation in 2026 post.
- **OpenAI** ⚠ — internal practice referenced in launch posts.
- **GitHub Copilot** ⚠ — capability tracking + production regression.
- **Cursor, Replit, Aider** ⚠ — visible from public discussion of internal eval programs.
- **Cognition (Devin)** ✅ — public SWE-bench scores (capability) + internal regression suites.
- **Notion, Linear, Stripe** ⚠ — modern AI engineering practice.

## Further reading

- [Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) — Anthropic Jan 2026
- [Your AI Product Needs Evals](https://hamel.dev/blog/posts/evals/) — Hamel Husain
- [What we learned from a year of building with LLMs (Part I)](https://www.oreilly.com/radar/what-we-learned-from-a-year-of-building-with-llms-part-i/) — Yan, Bischoff, Shankar et al. (covers eval discipline)
- [OpenAI Evals](https://github.com/openai/evals) — OSS framework
- [Inspect AI](https://inspect.ai-safety-institute.org.uk/) — UK AISI framework with multi-suite support

---

*Diagram source: [`../diagrams/src/capability-vs-regression-evals.d2`](../diagrams/src/capability-vs-regression-evals.d2)*

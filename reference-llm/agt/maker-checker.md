# Maker-Checker

**Aliases:** generator-verifier, evaluator-optimizer (Anthropic's name), critic-actor, draft-and-review, two-agent verification, generate-critique-revise
**Category:** Agentic Patterns
**Sources:**
[Anthropic — Building effective agents (Dec 2024)](https://www.anthropic.com/research/building-effective-agents) ·
[Anthropic — Demystifying evals for AI agents (Jan 2026)](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) ·
[Madaan et al. — Self-Refine (2023)](https://arxiv.org/abs/2303.17651) ·
[Shinn et al. — Reflexion (2023)](https://arxiv.org/abs/2303.11366) ·
classic CS: maker-checker is a banking/security pattern dating to the 1970s

---

## Problem

> [!TIP]
> **ELI5.** LLMs are confidently wrong all the time. A single model checking its own work tends to *agree with itself* — confirmation bias is a real cognitive failure mode for LLMs, just like for humans. The fix: split the work. One agent *makes* the thing (writes the code, drafts the email, summarizes the document). A *different* agent *checks* it — looks for errors, bias, missing pieces, policy violations. The checker doesn't know how the maker reasoned, so it brings fresh eyes. If the checker says "no, this is wrong because X," the maker tries again with that critique in hand. Loop until the checker is satisfied.

A single agent producing high-stakes output has an asymmetry problem: **generating is cheap, verifying is hard**. Even with reasoning models, an agent that produces an answer and then "checks" its own work usually finds nothing wrong — because the same model that produced the answer is the one assessing it, and it's already convinced itself.

The pattern *every* mature high-stakes agentic system uses: separate the *generator* from the *verifier*. Different prompts. Often different models. Sometimes different vendors. The verifier's job is to find problems; it gets credit for finding them and no credit for rubber-stamping.

Anthropic's [Building effective agents](https://www.anthropic.com/research/building-effective-agents) post calls this **evaluator-optimizer** and classifies it as a workflow (the loop structure is fixed, even though it iterates). The CS literature calls it **maker-checker** — a pattern that predates LLMs by decades; banks have used it forever ("the person making the trade is not the person approving it"). In agentic terms, it's also closely related to **Self-Refine** (Madaan et al. 2023) and **Reflexion** (Shinn et al. 2023).

This pattern is unfashionable compared to "single super-agent" architectures, but it's the one that ships in production for compliance-sensitive, customer-facing, or safety-critical output.

## How it works

> [!TIP]
> **ELI5.** Three pieces. (1) A **maker** prompt that generates the output. (2) A **checker** prompt that critiques it — and crucially, the critique has to be *specific* ("the email mentions a Q4 deadline that's actually Q3" — not "looks okay"). (3) A **loop** that feeds the critique back to the maker for revision. Stop when the checker passes or when you hit max iterations. The split prompts + specific critiques are what make this work; without them, the checker just agrees with the maker.

![Maker-checker — adversarial verification](../diagrams/svg/maker-checker.svg)

Three design discoveries make this pattern actually work in 2025-2026 production:

### 1. The maker and checker must be *separate prompts* (often separate models)

The bare-minimum split: the maker has a generative prompt ("Write a code refactor that..."), the checker has a critical prompt ("Find errors, bugs, or missed requirements in this code"). Same model, different roles. This alone catches many errors a single agent would miss.

The stronger split: **different models**. Use Sonnet for the maker, GPT-5 for the checker (or vice versa). Different model families have different blind spots; cross-model checking catches systematic errors better than same-model checking. In high-stakes systems, this asymmetry is worth the operational complexity.

The strongest split (used in some financial and security contexts): **different vendors entirely**. Reduces the risk of correlated failures from shared training data, shared safety filters, or shared bugs.

### 2. The critique must be *actionable* and *specific*

A pass/fail verdict from the checker doesn't help the maker improve. A useful critique looks like:

```
Issue 1: The function `compute_total` doesn't handle the case where `items` is empty;
         it will raise IndexError on line 12. Add a guard.

Issue 2: The PR description claims this is a "non-breaking change" but the signature
         of `process_order` now requires `customer_id` where it was optional. This IS
         breaking. Either restore the default or update the description.

Issue 3: No test added for the new `discount_code` parameter. Add at least one
         positive and one negative test case.
```

The maker can take this critique and address each item. A vague "this looks rough; rewrite it" is useless.

Generating actionable critiques is harder than generating the original output. It usually requires:
- An explicit rubric or checklist in the checker's prompt ("Check for: correctness, completeness, style, tests, security, performance").
- Few-shot examples of "good critiques" in the checker's prompt.
- The checker having access to ground truth or specifications when available (test files, design docs, requirements).

Anthropic's [evals post](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) (Jan 2026) explicitly recommends LLM-as-judge prompts that ask for *justifications*, not just scores — for the same reason.

### 3. The loop must have a halt condition

Naively, you'd loop until the checker says "perfect." In practice:
- Models can be too critical, finding nits forever.
- Some critiques are wrong; the maker can't fix what isn't broken.
- Cost grows with each iteration.

Typical halt conditions, layered:
- **Checker passes** — no issues found.
- **Max iterations** — usually 3-5 in production; rarely beyond.
- **Quality plateau** — if the checker's critique is the same as last round, stop.
- **Escalate to human** — after N rounds without convergence, hand to a person.
- **Cost cap** — total tokens / dollars per maker-checker session.

The right balance depends on the task. For a legal document review, you might iterate 10 times and escalate; for an internal Slack message, 2 iterations max.

### When this pattern matters most

Maker-checker is the right shape when:

- **High-stakes output** — code that ships, emails that send, decisions that bind.
- **Spec-checkable outputs** — there's a clear rubric the checker can apply.
- **Confirmation bias is a real concern** — when a self-checking agent would rubber-stamp.
- **Multi-perspective value** — when the checker can bring expertise the maker can't.
- **Compliance / audit** — separation of generation and approval is often a regulatory requirement.

It's the wrong shape when:

- **The task is creative / open-ended** — there's no rubric to apply.
- **The maker is already a reasoning model with extensive internal critique** — much of the benefit is already captured.
- **Latency matters** — the loop doubles or triples wall-time.
- **The output is low-stakes / disposable** — single agent + accept-or-redo is faster.

### Concrete production examples

**Code review agents.** Many coding-agent systems run a separate "reviewer agent" pass on the proposed diff before showing it to the user. Anthropic's Claude Code does this implicitly when running tests; Cursor and Devin have explicit review steps. The checker prompt focuses on correctness, test coverage, and matching the user's stated intent.

**Customer-service draft-and-review.** Many SaaS customer-service tools (Intercom, Zendesk, Sierra, Decagon) generate draft responses via one model and have a separate model check for tone, accuracy, and policy compliance before either sending automatically or surfacing to a human agent.

**Compliance / financial / legal agents.** In banking, insurance, and legal-tech, the pattern is non-optional: the model generating an output is *not allowed* to be the model approving it. Many vendors implement this with explicitly different model families.

**Content moderation pipelines.** A maker generates content (e.g., a marketing email). A checker scans for policy violations, hallucinated facts, brand-voice violations. Used by most large content platforms.

**Self-Refine and Reflexion (research).** These 2023 papers formalized the single-agent version of maker-checker (the same model alternates roles). Useful for benchmarks; in production, two-agent separation almost always outperforms.

### Variations

- **N-of-M consensus.** Run M makers in parallel, take the most-agreed answer. Variant of [parallelization](#).
- **N-of-M voting on the checker.** Run M checkers in parallel; flag the output if any checker objects.
- **Checker-with-tools.** The checker has access to its own tools (web search, fact-check API, test runners) that the maker didn't use.
- **Hierarchical maker-checker.** A checker that finds an issue can spawn a more specialized sub-checker for that specific concern.
- **Maker-checker with human escalation.** After N rounds, hand to a person.

## Variants & related patterns

- [**Workflows vs Agents**](workflows-vs-agents.md) — maker-checker is classified as a workflow (evaluator-optimizer).
- [**Multi-agent orchestration**](multi-agent-orchestration.md) — maker-checker is the simplest 2-agent topology.
- [**Single agent with tools**](single-agent-with-tools.md) — what maker-checker improves over.
- [**LLM-as-judge with calibration**](#) (see `../qua/`) — the offline evaluation analogue.
- [**Sub-agent architectures**](sub-agent-architectures.md) — the checker is often spawned as a sub-agent.
- **Self-Refine** — single-model self-iteration (weaker).
- **Reflexion** — adds memory of past critiques.
- **Tree of Thoughts / Graph of Thoughts** — explore multiple maker outputs, checker selects.
- **Constitutional AI** (Anthropic) — checker prompts derived from explicit principles.
- **RLHF / RLAIF** — training-time analogue: human (or AI) raters as checkers.

## When NOT to use

- **Low-stakes, disposable output** — overhead doubles cost for marginal gain.
- **Tasks without clear quality criteria** — the checker has nothing to check against.
- **Real-time interactive use** where latency matters more than perfection.
- **When the maker is already a strong reasoning model** with built-in self-critique (some of the benefit is already there).
- **When the checker would just rubber-stamp** — without a different model, different prompt, and explicit critique rubric, this pattern silently fails. You'll think you have verification when you have just-extra-cost.

## Implementations

| Framework / pattern | Maker-checker support |
|---|---|
| **Anthropic Agent SDK** | DIY — two agents, one critiques the other |
| **LangGraph** | Native — explicit critic node + loop edge |
| **DSPy** | `ChainOfThought` + `Assert` modules for built-in checking |
| **CrewAI** | Two-role crew: writer + reviewer |
| **AutoGen** | Critic agent in a group-chat |
| **OpenAI Assistants API** | DIY: two assistants, second reviews first |
| **OpenAI structured outputs + guardrails** | Schema-based checking before LLM-as-judge |
| **GuardrailsAI, NeMo Guardrails, Lakera Guard** | Production-grade checker layers |
| **Constitutional AI training** | Training-time maker-checker |

## Companies using maker-checker in production

- **Anthropic** ✅ — Constitutional AI in training; runtime maker-checker patterns in Claude products ([source](https://www.anthropic.com/research/building-effective-agents)).
- **OpenAI** ⚠ — moderation pipeline; safety classifier as checker; specifics not all public.
- **Sierra, Decagon, Intercom** ⚠ — customer-service draft-and-review pattern is widespread but architecture details vary.
- **Sourcegraph (Cody Review), Codacy** ⚠ — code-review agents.
- **JPMorgan, Goldman Sachs, banks generally** ⚠ — regulated maker-checker is required; LLM-augmented versions emerging.
- **NVIDIA NeMo Guardrails** ✅ — open-source framework that productizes checker patterns.
- **Lakera, GuardrailsAI** ✅ — vendors selling the checker layer.

## Further reading

- [Building effective agents — Evaluator-Optimizer section](https://www.anthropic.com/research/building-effective-agents) — Anthropic Dec 2024
- [Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) — Anthropic Jan 2026 (LLM-as-judge principles)
- [Self-Refine: Iterative Refinement with Self-Feedback](https://arxiv.org/abs/2303.17651) — Madaan et al. 2023
- [Reflexion: Language Agents with Verbal Reinforcement Learning](https://arxiv.org/abs/2303.11366) — Shinn et al. 2023
- [Constitutional AI: Harmlessness from AI Feedback](https://arxiv.org/abs/2212.08073) — Bai et al. 2022 (Anthropic training-time pattern)
- [NeMo Guardrails](https://github.com/NVIDIA/NeMo-Guardrails) — production checker framework

---

*Diagram source: [`../diagrams/src/maker-checker.d2`](../diagrams/src/maker-checker.d2)*

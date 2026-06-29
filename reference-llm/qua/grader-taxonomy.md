# Grader Taxonomy (Code / Model / Human)

**Aliases:** judge taxonomy, eval-grader families, automatic vs LLM-as-judge vs human evaluation
**Category:** Quality & Evals
**Sources:**
[Anthropic — Demystifying evals for AI agents (Jan 2026)](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) ·
[Zheng et al. — Judging LLM-as-a-Judge (NeurIPS 2023)](https://arxiv.org/abs/2306.05685) ·
[Shankar et al. — Who Validates the Validators? (2024)](https://arxiv.org/abs/2404.12272) ·
[Hamel Husain — Putting Evals into Production](https://hamel.dev/blog/posts/llm-evals/)

---

## Problem

> [!TIP]
> **ELI5.** You want to score an LLM's output. Three options. **Code grader**: a script checks it (`output == expected`). Fast and cheap, but only works for tasks with one right answer. **Model grader**: another LLM scores it. Scales, handles fuzzy quality, but costs money and can be biased. **Human grader**: a person rates it. Gold standard, but slow and expensive. The taxonomy isn't "which is best" — it's "which combination, for this task." Most production setups use all three: code where they can, model where they must, human to keep the model honest.

When you build an eval ([eval-driven development](eval-driven-development.md)), the load-bearing piece is the **grader** — the thing that takes a model's output and produces a verdict. Get the grader wrong and the eval is worse than no eval at all: you'll confidently optimize for the wrong thing.

The 2025-2026 consensus organizes graders into three families. Anthropic's [January 2026 evals post](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) names them explicitly. Each family has clear strengths and weaknesses; mature eval systems use them in combination, with each grading what it's best at.

## How it works

> [!TIP]
> **ELI5.** Pick the cheapest grader that still works for your task. **Code grader** for anything verifiable (exact match, schema valid, test passes). **Model grader** (LLM-as-judge) for open-ended outputs where a rubric exists. **Human grader** for calibration — they're the source of truth that tells you whether your model grader can be trusted. The rule of thumb: code where you can, model where you must, human to calibrate the models.

![Grader taxonomy — three families](../diagrams/svg/grader-taxonomy.svg)

### 1. Code graders (deterministic)

The output gets compared to a ground truth by code. Concrete forms:

- **Exact match.** `output == "Paris"`. Used for trivia, structured-data extraction, classification.
- **Regex / pattern match.** `re.match(r"^\d{4}-\d{2}-\d{2}$", output)`. Used for format checks.
- **Schema validation.** Validate the output against a JSON Schema, Pydantic model, or Protobuf definition. Used heavily with structured outputs.
- **Test suite pass/fail.** Run the generated code through `pytest`, `cargo test`, `go test`. Used for coding agents and SWE-bench-style evals.
- **Diff equality / set equality.** For tasks where order doesn't matter.
- **Function-return equality.** Did the generated SQL produce the expected rows? Did the generated Python function return the expected value on test inputs?

**Pros**: Cheap (microseconds per grade). Reproducible — same input always gives the same verdict. Zero variance. Auditable. Trustworthy.

**Cons**: Only works for tasks with verifiable outputs. Brittle to format changes (a single extra whitespace breaks exact-match). Often too strict for natural-language outputs ("the capital of France is Paris" fails exact-match against "Paris").

**Use whenever you can.** A code grader you trust is dramatically better than a model grader you don't. Many open-ended tasks have *partial* code-checkable structure (a date in the output, a specific field present, a constraint satisfied) — extract that signal first; let model graders handle what's left.

### 2. Model graders (LLM-as-judge)

An LLM scores the output against a rubric or compares two outputs. The field invented this in 2023 to handle open-ended tasks code couldn't grade; by 2026 it's the dominant grader family for production evals.

Two main shapes:

- **Rubric scoring.** The judge gets the input, the output, and a rubric: "Rate the response 1-5 on whether it (a) answers the user's question, (b) cites correct sources, (c) avoids inventing facts." Returns a score + justification.
- **Pairwise comparison.** The judge sees two outputs (one from system A, one from system B, for the same input) and decides which is better — or declares a tie. Then aggregate over many examples to get a win-rate. Pairwise is generally more reliable than absolute scoring because humans (and LLMs) are better at *comparing* than *rating in isolation*.

**Pros**: Scales — you can grade thousands of examples per hour. Handles open-ended outputs. Captures nuance code can't. Reasonably consistent within a single grader instance.

**Cons**: Costs LLM tokens (often comparable to running the system you're evaluating). Has biases — position bias (favors first or second option), length bias (favors longer responses), self-preference bias (favors outputs that look like the judge model's style). Variance — same input can get different scores on different runs. Must be calibrated against humans before you trust it.

**The calibration step is non-negotiable.** Before relying on a model grader, you measure its agreement with human raters on a small but representative set of examples. If the model grader agrees with humans 70% of the time, that's your ceiling — and you should be skeptical of small score differences (1-2 points within the noise band). If it agrees 90%+, you can trust larger differences. The [Shankar et al. "Who Validates the Validators?"](https://arxiv.org/abs/2404.12272) paper is the canonical methodology.

**Production patterns for model graders**:
- Use a strong model as the judge (often stronger than the system being evaluated). GPT-5, Claude Opus 4.5, Gemini 2.5 Pro are common choices.
- Use **multiple judges** and aggregate (mean or majority vote) to reduce variance.
- Pin judge model versions — judges drift just like other models.
- Generate **justifications** alongside scores; they're useful for debugging and meta-eval.
- Provide explicit **examples** of good and bad outputs in the judge's prompt (few-shot).
- Use **pairwise** instead of absolute when possible.

### 3. Human graders (expert raters)

Trained humans rate or compare outputs. The gold standard — and the source of truth that calibrates everything else.

Forms:
- **Internal SMEs.** Engineers, product managers, domain experts rate outputs in a shared spreadsheet or labeling tool.
- **External annotation platforms.** Scale AI, Surge, Mechanical Turk, Toloka — trained raters at scale.
- **Production raters.** Real users grading via thumbs-up/down or detailed feedback in the product (Cursor's "good response" / "bad response" buttons, ChatGPT's regenerate-and-prefer flow).

**Pros**: Gold standard. Captures preferences code and models can't (cultural nuance, judgment calls, edge cases). The reference all other graders are calibrated against. Required for some safety / compliance contexts.

**Cons**: Slow (hours to days for a batch). Expensive. Doesn't scale to per-PR CI. Inter-rater variance is significant — even trained raters disagree 10-20% of the time on subjective rubrics. Requires careful rater-training, guideline-writing, and quality control.

**Use human graders for**:
- **Calibration** — the small held-out set you measure your model graders against.
- **Final acceptance** of high-stakes deployments.
- **Periodic audits** — even if production runs on model graders, sample a percentage for human review.
- **Bootstrapping new eval tasks** — when you don't yet know what "good" looks like, humans tell you.

### How the three combine in production

The rule of thumb that emerged from 2024-2025 practice and is codified in Anthropic's 2026 guidance:

> **Use code graders where you can, model graders where you must, human graders to calibrate the model graders.**

A mature production eval stack looks like:

1. **Code graders run on every PR.** Schema, format, regression checks. Sub-second per case. Always green.
2. **Model graders run nightly or on significant changes.** Cover open-ended quality. Take minutes to hours.
3. **Human graders run periodically** — weekly or monthly batches — to recalibrate the model graders and catch drift.
4. **Production telemetry** (user thumbs-up/down) feeds back into the rater pool.

This combination keeps cost manageable, latency tolerable, and trust in the eval scores high.

### Specific failure modes per family

- **Code graders**: overly strict on format → false negatives ("Paris." marked wrong because trailing period). Mitigation: normalize before comparing, or use partial-credit graders.
- **Model graders**: silently agree with the system being evaluated (especially if both are the same model family). Mitigation: cross-vendor judge.
- **Model graders**: position / length / style bias. Mitigation: randomize order, swap models, control for length.
- **Human graders**: inter-rater disagreement → noisy scores. Mitigation: 2-3 raters per item with adjudication; clear rubrics; rater training.
- **All three**: distribution shift. Your eval set was relevant in 2024; it may not represent 2026 use. Refresh periodically.

### When the grader IS the bottleneck

Sometimes the grader is harder than the system being evaluated. Especially true for:
- **Multi-step agentic tasks** where the "right" trajectory is one of many.
- **Reasoning-model outputs** where the chain of thought matters but is hard to grade.
- **Creative outputs** where preference varies widely.

In these cases, the eval-eval (meta-eval) becomes its own engineering project. This is the leading edge of evals research in 2026.

## Variants & related patterns

- [**Eval-driven development**](eval-driven-development.md) — the methodology graders live inside.
- [**Pass@k vs pass^k**](pass-at-k-vs-pass-power-k.md) — graders applied to multi-sample outputs.
- [**Capability vs regression evals**](capability-vs-regression-evals.md) — different eval types use different grader mixes.
- [**End-state evaluation**](end-state-evaluation.md) — graders applied to agent trajectories.
- [**Eval saturation**](eval-saturation.md) — what happens when graders can't distinguish models any more.
- [**Maker-checker**](../agt/maker-checker.md) — the model-grader pattern applied at runtime, not just at eval time.
- **G-Eval / GEMBA / Prometheus** — published LLM-as-judge methodologies.
- **MT-Bench, AlpacaEval, Arena (lmsys)** — well-known model-graded benchmarks.
- **HumanEval, MBPP, SWE-bench** — code-graded benchmarks.
- **RLHF / DPO / KTO** — human-preference-driven *training*, downstream of human-grader datasets.

## When NOT to use specific grader types

**Avoid code graders** for: open-ended generation, creative tasks, multi-valid-answer questions, any task where partial credit matters.

**Avoid model graders** for: high-stakes safety decisions without human review, very subjective preference (use human), and any case where you haven't calibrated against humans first.

**Avoid human graders** for: per-PR regression CI (too slow), high-throughput production telemetry (too expensive), any pipeline where speed matters more than gold-standard quality.

## Implementations

| Tool / framework | Code | Model | Human |
|---|---|---|---|
| **Anthropic Workbench evals** | ✅ | ✅ | manual |
| **OpenAI Evals** | ✅ | ✅ | manual |
| **Promptfoo** | ✅ | ✅ | via UI |
| **Inspect AI (UK AISI)** | ✅ | ✅ | via integrations |
| **LangSmith** | ✅ | ✅ | annotator UI |
| **Weave (W&B)** | ✅ | ✅ | annotator UI |
| **Braintrust** | ✅ | ✅ | annotator UI |
| **Phoenix (Arize)** | ✅ | ✅ | manual |
| **DeepEval** | ✅ | ✅ | manual |
| **Ragas** | ✅ | ✅ | manual |
| **Scale AI, Surge AI, Toloka** | — | — | ✅ (managed rater services) |
| **HumanLayer** | — | — | ✅ (human-in-loop) |
| **Argilla, Label Studio** | — | — | ✅ (annotation platforms) |

## Companies using mixed grader strategies

- **Anthropic** ✅ — uses all three; explicit recommendation ([source](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)).
- **OpenAI** ✅ — public evals repo; internal use mixed.
- **Cognition (Devin)** ⚠ — SWE-bench (code-graded) + internal model graders.
- **GitHub** ✅ — Copilot uses code graders (tests) + human telemetry + model graders.
- **Cursor, Replit** ⚠ — similar stack to GitHub Copilot.
- **Scale AI, Surge AI, Toloka** ✅ — productize the human-grader layer.
- **Stripe, Notion, Linear** ⚠ — internal eval programs combining all three.
- **AcadeMix LMSys (Chatbot Arena)** ✅ — pioneered crowdsourced human pairwise evaluation at scale.

## Further reading

- [Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) — Anthropic Jan 2026
- [Judging LLM-as-a-Judge with MT-Bench and Chatbot Arena](https://arxiv.org/abs/2306.05685) — Zheng et al. 2023 (the foundational LLM-as-judge paper)
- [Who Validates the Validators? Aligning LLM-Assisted Evaluation with Human Preferences](https://arxiv.org/abs/2404.12272) — Shankar et al. 2024
- [G-Eval: NLG Evaluation using GPT-4 with Better Human Alignment](https://arxiv.org/abs/2303.16634) — Liu et al. 2023
- [Prometheus 2: An Open Source Language Model Specialized in Evaluating Other Language Models](https://arxiv.org/abs/2405.01535) — Kim et al. 2024
- [Hamel Husain — Putting Evals into Production](https://hamel.dev/blog/posts/llm-evals/)
- [Chatbot Arena](https://lmarena.ai/) — the canonical human-pairwise benchmark

---

*Diagram source: [`../diagrams/src/grader-taxonomy.d2`](../diagrams/src/grader-taxonomy.d2)*

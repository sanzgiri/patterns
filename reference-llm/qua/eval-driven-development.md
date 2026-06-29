# Eval-Driven Development

**Aliases:** eval-first development, EDD, evals-as-tests, test-driven LLM development
**Category:** Quality & Evals
**Sources:**
[Anthropic — Demystifying evals for AI agents (Jan 2026)](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) ·
[Hamel Husain — Your AI Product Needs Evals (2024)](https://hamel.dev/blog/posts/evals/) ·
[Eugene Yan — How to be successful with LLMs (2024)](https://eugeneyan.com/writing/llm-patterns/) ·
[Shreya Shankar — Who Validates the Validators?](https://arxiv.org/abs/2404.12272)

---

## Problem

> [!TIP]
> **ELI5.** You change a prompt. Did it get better, worse, or the same? You can't tell by squinting at a few outputs — LLMs are too variable. Without a *measurement* you trust, you'll spend hours convinced you're improving things while actually moving sideways (or backwards). Evals are the LLM equivalent of unit tests: a thing you run that says "this version scores 73, the previous one scored 71" and gives you ground under your feet. **Eval-driven development** means you write the eval *first* — define what success looks like — *before* you write the prompt or build the agent.

By mid-2024 it was clear that "vibes-based prompt engineering" didn't scale. Teams that lacked evals would spend weeks adjusting prompts, get conflicting reports from team members about whether things were improving, and eventually ship something that turned out to be worse than what they had three months earlier. Teams that had evals shipped consistent improvements measured in % points.

By 2026 the discipline has a name: **eval-driven development** (EDD), explicit in Anthropic's January 2026 ["Demystifying evals for AI agents"](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) and a baseline assumption in most serious AI engineering teams. Anthropic's framing is blunt:

> *"Without evals you cannot tell whether a change helped, hurt, or did nothing. Evals come first, code follows."*

The parallel to test-driven development (TDD) is deliberate. In TDD, you write the failing test before the implementation. In EDD, you write the eval — the definition of success, the held-out examples, the grading rubric — *before* you build the prompt / agent / workflow. Once the eval exists, every change has a clear scoreboard.

## How it works

> [!TIP]
> **ELI5.** Five steps, run in order, every time. (1) Define what "good output" looks like — write it down as examples or a rubric. (2) Build an eval that *measures* how close any output is to "good." (3) Run the current system against the eval; that's your baseline score. (4) Make the change. (5) Re-run the eval. If the score went up, ship; if not, iterate or revert. Add every shipping eval to CI so regressions get caught later.

![Eval-driven development loop](../diagrams/svg/eval-driven-development.svg)

The methodology has five steps that recur on every change:

### 1. Define what success looks like — *before* coding

Before you write a prompt or build an agent, define the answer to the question: *"How will I know if this works?"* The answer is concrete. It might be:

- A set of input examples with the expected outputs (for classification, extraction, structured generation).
- A rubric the output must satisfy (for open-ended tasks: "the email should be polite, address all the user's points, and not invent any facts about pricing").
- A success criterion checkable by code or tools (for agentic tasks: "the test suite passes after the change").
- A pairwise-preference judgment (for subjective tasks: "this output is better than the baseline output").

Whatever the shape, **the definition lives in code or documents, not in your head**. If only you know what "good" means, only you can evaluate the system — and that doesn't scale to a team or to time (you'll forget your own criteria in three months).

### 2. Build the eval

The eval is a runnable thing — a script, a notebook, a CI job — that takes the system's outputs and produces a number (or a structured verdict). Three families of *graders* exist (see [grader taxonomy](grader-taxonomy.md)):

- **Code graders** — deterministic checks (exact match, schema validation, test pass-fail).
- **Model graders** — LLM-as-judge with a rubric.
- **Human graders** — expert raters for calibration / gold standard.

The choice of grader depends on the task. Always prefer code where possible (cheap, reproducible). Fall back to model graders for open-ended outputs. Use human graders to *calibrate* the model graders (covered in [grader taxonomy](grader-taxonomy.md)).

The eval also needs **examples**: a held-out set of inputs you'll run the system against. Anthropic recommends starting with **20-50 examples** drawn from realistic use cases (not synthetic — real production traffic if you have it). Scale up later. Quality over quantity in the early phase.

### 3. Run the baseline

Run the eval against the *current* system before making any change. Record the score. This is your reference point — without it you can't tell whether subsequent changes are improvements.

### 4. Make the change

Now build / edit / tune. The change might be:
- A prompt revision.
- A new tool added.
- A new model swapped in.
- A new sub-agent introduced.
- A retrieval pipeline change.
- A fine-tuning round.
- An agent skill addition.

### 5. Re-run the eval, compare scores

The change either:
- **Improved the score** — ship it, add the eval to CI as a regression check.
- **Hurt the score** — revert or iterate. Often the failure mode is interesting and points to a specific weakness you can address.
- **Did nothing** — also useful information. Probably the change wasn't worth the complexity it added.

The discipline is in *always* doing step 5, *always* comparing to baseline, and *never* shipping changes you haven't measured.

### What changes between EDD and traditional TDD

EDD inherits much from TDD but has some notable differences:

- **Variance is fundamental.** LLM outputs aren't deterministic. Evals need to run multiple samples and report distributions, not single scores. A score of `73 ± 5` means something different from `73 ± 0.5`.
- **The eval can be wrong.** TDD tests are deterministic and (mostly) trustworthy. LLM-as-judge evals can have bias, can mis-grade, can drift over time. You have to evaluate your evals — meta-evaluation.
- **Held-out data is precious.** Once an eval set has been used many times, you've effectively trained on it (selecting prompts that score high). Periodically rotate held-out sets.
- **Calibration matters more than raw scores.** A `0.73` from one model grader and a `0.73` from another don't mean the same thing. Always anchor to human-rated examples.
- **Cost is non-trivial.** Running a full eval suite can cost dollars per run; LLM-as-judge multiplies that. Budget accordingly.

### What goes in CI

Once an eval is built and trusted, add it to CI so:
- Every PR that touches the prompt / agent / model triggers the eval.
- Reviewers see "score went from 73 to 71" alongside the diff.
- Regressions get caught before deploy.

Many teams gate deploys on eval scores: "this PR cannot merge until the eval score is at least 70." This is the production equivalent of "tests must pass."

### Where teams fail at EDD

The pattern is simple, but doing it consistently is rare. Common failure modes:

- **No baseline.** Teams write evals but never establish a reference score; every result is uninterpretable.
- **Vague rubrics.** "Is the output good?" is not a rubric. "Does the output address all 5 of the user's questions, in under 200 words, without inventing pricing?" is.
- **Synthetic-only examples.** Evals built from imagined cases miss the failure modes that actual users trigger.
- **Eval drift.** The same eval gives different scores week-over-week because the LLM-as-judge changed. Pin model versions.
- **Score-chasing.** Optimizing for the eval score, not for actual user outcomes. Periodically validate with humans.
- **Eval-as-afterthought.** Building the eval *after* the prompt is reverse EDD. The prompt biases you toward writing an eval it'll pass.

### What "eval-driven" looks like in different team sizes

- **Solo developer.** A notebook with 30 examples, a rubric, and a regenerate-and-compare script. Run before every prompt iteration.
- **Small team (5-15).** Shared eval set in version control, a script that runs nightly, dashboard showing scores over time. PRs require eval-improvement justification.
- **Production org.** Multiple eval suites for different deployments (regression, capability, safety), nightly + per-PR runs, alerting on score drops, eval-driven model upgrades. Often a dedicated evals team.

## Variants & related patterns

- [**Grader taxonomy**](grader-taxonomy.md) — what kinds of evals you can build (code / model / human).
- [**End-state evaluation**](end-state-evaluation.md) — what to evaluate in agent traces (the final state, not every step).
- [**Pass@k vs pass^k**](pass-at-k-vs-pass-power-k.md) — how to think about multi-sample evals.
- [**Capability vs regression evals**](capability-vs-regression-evals.md) — different eval purposes serve different needs.
- [**Eval saturation**](eval-saturation.md) — what happens when models exceed your eval's ceiling.
- [**Maker-checker**](../agt/maker-checker.md) — the runtime analogue of EDD (LLM grades LLM in production).
- **Test-driven development (TDD)** — the conceptual parent.
- **Continuous integration (CI/CD)** — where production evals live.
- **A/B testing in production** — the real-user analogue of held-out eval sets.

## When NOT to use

- **For one-off scripts and disposable code.** Evals are infrastructure; one-off prompts don't need them.
- **In very early prototyping** where you don't know what you're building yet. Build the v0 first, then build the eval.
- **For purely creative / aesthetic tasks** where there's no rubric, only preference. Use preference-comparison instead.
- **When the work product itself is the eval.** If you're building an eval *as* your deliverable, EDD doesn't recursively apply.
- **For safety-critical decisions where evals alone aren't sufficient** — adversarial red-teaming, formal verification, and human review remain necessary alongside.

## Implementations

| Tool / framework | EDD support |
|---|---|
| **Anthropic Workbench** | Built-in eval framework (Anthropic Console). Reference implementation. |
| **OpenAI Evals** | Open-source eval framework, official OpenAI repo. |
| **Promptfoo** | Open-source CLI for prompt evals; CI-friendly. |
| **LangSmith** | Tracing + evals, tight integration with LangChain. |
| **Weights & Biases Weave** | Eval tracking + UI for results over time. |
| **Inspect AI (UK AISI)** | Open-source agentic eval framework, gaining production traction. |
| **DeepEval** | Pytest-style assertions for LLM outputs. |
| **Ragas** | RAG-specific evals; faithfulness, relevance, etc. |
| **Phoenix (Arize)** | Production observability + evals. |
| **Braintrust** | Commercial eval platform with replay + diffing. |
| **HumanLayer** | Adds human-in-loop eval to agent outputs. |
| **TruLens** | Eval framework focused on RAG / agentic. |

## Companies practicing eval-driven development

- **Anthropic** ✅ — explicit EDD methodology in Jan 2026 post ([source](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)).
- **OpenAI** ✅ — OpenAI Evals repo + internal practice referenced in launch posts.
- **Cognition (Devin)** ⚠ — internal eval discipline referenced in engineering posts.
- **GitHub (Copilot)** ✅ — public engineering posts describe eval-first methodology.
- **Cursor, Sourcegraph, Replit** ⚠ — eval-driven culture referenced in hiring posts and tech talks.
- **Stripe, Notion, Linear, Vercel** ⚠ — internal AI features developed with EDD; not all publicly documented.
- **Hamel Husain consultancies** ✅ — "Your AI Product Needs Evals" is the most-cited practitioner essay on EDD.

## Further reading

- [Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) — Anthropic Jan 2026 (the canonical 2026 source)
- [Your AI Product Needs Evals](https://hamel.dev/blog/posts/evals/) — Hamel Husain (the most-cited practitioner essay)
- [How to be successful with LLMs — the evals section](https://eugeneyan.com/writing/llm-patterns/) — Eugene Yan
- [Who Validates the Validators? Aligning LLM-Assisted Evaluation with Human Preferences](https://arxiv.org/abs/2404.12272) — Shankar et al. 2024 (meta-evals)
- [OpenAI Evals](https://github.com/openai/evals) — the OSS framework
- [Inspect AI](https://inspect.ai-safety-institute.org.uk/) — UK AISI's agentic eval framework

---

*Diagram source: [`../diagrams/src/eval-driven-development.d2`](../diagrams/src/eval-driven-development.d2)*

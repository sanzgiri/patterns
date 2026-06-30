# Evaluator-Optimizer (Generator-Critic Loop)

**Aliases:** generator-critic loop, iterative refinement, self-critique loop, reflection-driven generation, optimize-by-evaluation
**Category:** Workflows
**Sources:**
[Anthropic — Building effective agents (Dec 2024)](https://www.anthropic.com/research/building-effective-agents) ·
[Shinn et al. — Reflexion (2023)](https://arxiv.org/abs/2303.11366) ·
[Madaan et al. — Self-Refine (2023)](https://arxiv.org/abs/2303.17651) ·
[Bai et al. — Constitutional AI (2022)](https://arxiv.org/abs/2212.08073) (revision-via-critique foundations)

---

## Problem

> [!TIP]
> **ELI5.** Two LLM calls play tag-team. The first one (the **generator**) writes a draft. The second one (the **evaluator**) reads it, scores it against a rubric, and writes specific feedback. If the score isn't good enough, hand the feedback back to the generator with "fix these things." Loop until the evaluator says "this is good" or you hit a cap. Result: outputs that are often dramatically better than the generator could produce in one shot — because *catching specific mistakes is easier than avoiding them in the first place*. This is Anthropic's fifth canonical workflow pattern.

The evaluator-optimizer pattern (Anthropic's [Building effective agents](https://www.anthropic.com/research/building-effective-agents) fifth pattern) is built on a useful asymmetry: **LLMs are often better at *evaluating* outputs than *producing* them**. Asking a model to write a perfect essay in one shot is hard; asking the same model to critique an essay against a rubric is easier; asking it to fix a critiqued essay is easier still. Chaining the three (generate → critique → fix) extracts more quality than asking for a perfect first draft.

This is also the *closed-loop* version of generation: the loop continues until quality crosses a threshold, rather than running a fixed number of times. That makes it different from [voting-style parallelization](parallelization.md) (which runs K times in parallel, picks the best) and from [maker-checker](../agt/maker-checker.md) (which runs the verifier once, not iteratively).

By 2026, the pattern is everywhere in production — most coding agents use a version of it (generate diff → run tests → fix), most content products use it (write → critique → polish), and most reasoning workflows include it implicitly via reasoning-model self-revision.

## How it works

> [!TIP]
> **ELI5.** Generator writes a draft. Evaluator reads it, scores it, and writes feedback. If the score is high enough, you're done. Otherwise, generator gets "here's what's wrong, fix it" and tries again. After 2-3 iterations, quality usually plateaus or improves dramatically. Cap the loop to avoid infinite spin.

![Evaluator-optimizer loop](../diagrams/svg/evaluator-optimizer.svg)

### The mechanics

1. **Generator produces a candidate.** A normal LLM call with the task and any context.
2. **Evaluator critiques.** A separate LLM call (usually with a stronger or differently-prompted model) reads the candidate and the rubric. Output: a score + specific feedback + a binary pass/revise verdict.
3. **Decision.** If the verdict is "pass" (or the score crosses a threshold), return the candidate. If "revise," continue.
4. **Generator revises.** Receives the original task + previous candidate + evaluator feedback. Produces a new candidate.
5. **Loop** back to step 2. Cap at N iterations (typically 2-4) to prevent infinite refinement.

### Why this works (the asymmetry)

Models are typically better at evaluation than generation, and better at *fixing specific issues* than *avoiding issues from scratch*. Mechanistic reasons:

- **The candidate already exists.** Critique is grounded in a concrete artifact, not an open task.
- **Specific feedback is high-signal.** "Sentence 3 is unclear — restate the argument" is much more actionable than the original task description.
- **The rubric narrows scope.** The evaluator looks for specific problems; the generator only has to fix those.
- **Different prompts activate different patterns.** Generator-prompted models go into "produce" mode; evaluator-prompted models go into "critique" mode — distinct activations from training.

The compounding effect: a 70%-quality first draft + 80%-effective critique + 90%-effective fix → final quality often well above 90%, on tasks where the model can't produce 90% directly.

### When the loop converges (and when it doesn't)

**Convergent tasks** (the loop reliably improves quality):
- Style / clarity editing.
- Code that has explicit specs / tests.
- Translation where fluency can be assessed by another reader.
- Structured outputs where format validation is checkable.
- Math / reasoning where the final answer can be verified.

**Divergent tasks** (the loop can plateau or degrade):
- Open-ended creative writing where there's no agreed-upon "better."
- Subjective tone / personality where iteration averages toward the model's bias.
- Tasks where the evaluator's rubric doesn't match what users actually want.

### Diminishing returns and iteration caps

Empirically: most quality gain happens in iterations 1-2. By iteration 3-4, returns drop sharply. By iteration 5+, the loop may *degrade* — over-revision damages the work.

Production patterns:
- **Hard cap at 3-4 iterations.**
- **Soft threshold: stop early** if evaluator says "pass" or score crosses a threshold.
- **Bail out on plateau:** if iteration N has nearly identical score to N-1, exit.
- **Best-of-iterations:** keep all candidates; return the best-scored at the end.

### Variants

**Same-model loop.** Generator and evaluator are the same model with different prompts. Cheapest. Works when the model is "good enough."

**Two-model loop.** Generator is a cheap/fast model; evaluator is a stronger model. Common production setup — strong-model critique is expensive but only one call per iteration.

**Self-Refine.** Madaan et al.'s [2023 paper](https://arxiv.org/abs/2303.17651) — the same model alternates roles via prompt changes.

**Reflexion.** Shinn et al.'s [2023 paper](https://arxiv.org/abs/2303.11366) — adds *episodic memory* of past failures; agent learns across iterations.

**Constitutional AI revision** ([Bai et al. 2022](https://arxiv.org/abs/2212.08073)). The original "revise-via-critique" pattern, used to train safer model outputs.

**Pair-critic.** Two evaluators with different rubrics; generator must satisfy both.

**External-validator loop.** Evaluator isn't an LLM — it's a code grader (test suite, type checker, lint). Generator iterates until tests pass. The dominant pattern in [coding agents](../agt/coding-agents.md).

### Difference from related patterns

| Pattern | Iteration | Evaluator | Purpose |
|---|---|---|---|
| **Evaluator-optimizer** | Yes, loops until threshold | LLM with rubric | Quality improvement |
| **[Maker-checker](../agt/maker-checker.md)** | Usually one-shot (no retry) | LLM or human | Verification before action |
| **[Self-Consistency](../fnd/reasoning-prompts.md)** | No iteration; K parallel samples | Voting aggregator | Variance reduction |
| **Reflection (in CoT)** | Optional, single-step | Same LLM | Reasoning quality |
| **External-validator loop (coding)** | Yes, loops until tests pass | Code grader | Functional correctness |

All four use "evaluation" in some form, but the topology differs.

### Designing the evaluator

The evaluator is the load-bearing piece. Get it wrong and the loop produces nothing useful.

**Concrete rubric.** Vague "is this good?" prompts give vague feedback. Specific rubrics ("does it satisfy these 5 criteria?") give specific feedback.

**Few-shot examples.** Show the evaluator examples of good-and-bad outputs and how they should be scored. Reduces evaluator variance.

**Structured output.** Evaluator should emit JSON: `{"score": 1-10, "verdict": "pass"|"revise", "issues": [...]}`. Don't parse free text.

**Mutual independence.** Evaluator should be configured *separately* from the generator. Avoid putting them in the same prompt — the evaluator gets biased toward the generator's style.

**Different model.** Using a *different vendor*'s model as evaluator reduces same-model-bias. (You can also self-evaluate with same model; just be aware of the bias.)

**Cost-aware.** Evaluator runs every iteration. Don't make it so expensive that the loop bankrupts you.

### The coding-agent special case

For coding tasks, the evaluator is often *not* an LLM — it's the test suite or compiler. This makes the loop dramatically more robust:

```
generator: produce code change
evaluator: run tests
  pass → done
  fail → feed test output to generator as "fix this error"
```

Code grading is deterministic, fast, and trustworthy. This is the secret behind why [coding agents](../agt/coding-agents.md) work so well — the evaluator is unbiased and unfoolable. For non-code tasks, model-graded evaluation has known biases ([grader taxonomy](../qua/grader-taxonomy.md)) that production setups have to compensate for.

### Production engineering

- **Iteration cap.** Hard limit (3-4 typical). Prevents runaway cost.
- **Token budget.** Total budget for the whole loop, not per call. Avoids surprises.
- **Logging per iteration.** Generator candidate, evaluator feedback, score progression. Essential for debugging.
- **Convergence telemetry.** Track how many iterations the average task needs. Drift means something changed.
- **Fallback when uncertain.** If iteration cap reached and threshold not met, escalate to human or fail gracefully.
- **Per-iteration cost telemetry.** "We spend 70% of tokens on iterations 3+" is a useful signal to cap earlier.

### Anti-patterns

- **No iteration cap.** Pathological inputs spin forever.
- **Same prompt as generator and evaluator.** Loses the asymmetry; effectively no improvement.
- **Vague rubric.** Critique is vague; generator can't act on it.
- **Loop without progress tracking.** Iteration 4 may be worse than iteration 2 — keep the best.
- **Evaluating tasks that have no clear "better."** Creative writing where preference is purely subjective.
- **Coupling evaluator's failure modes to generator's.** Same model + same context bias = useless loop.

### Composing with other patterns

- **Inside a chain step.** A chain step can be an evaluator-optimizer loop for the high-quality step.
- **As a worker in orchestrator-workers.** Orchestrator dispatches "produce high-quality X" → that worker is itself an evaluator-optimizer loop.
- **Following maker-checker.** Maker-checker gates whether to execute; evaluator-optimizer improves quality before gating.
- **Combined with voting.** Generate K candidates; have the evaluator pick the best; iterate from there.

## Variants & related patterns

- [**Prompt chaining**](prompt-chaining.md) — sequential composition without iteration.
- [**Routing**](routing.md) — dispatch, no loop.
- [**Parallelization**](parallelization.md) — voting is parallel-then-aggregate, not iterate.
- [**Orchestrator-workers**](orchestrator-workers.md) — can embed an evaluator-optimizer per worker.
- [**Maker-checker**](../agt/maker-checker.md) — single-shot verification vs iterating.
- [**Reasoning prompts**](../fnd/reasoning-prompts.md) — Reflection is a single-step relative.
- [**Grader taxonomy**](../qua/grader-taxonomy.md) — the evaluator is a grader.
- [**Eval-driven development**](../qua/eval-driven-development.md) — formalizes the rubric.
- **Self-Refine** (Madaan et al. 2023).
- **Reflexion** (Shinn et al. 2023).
- **Constitutional AI** (Bai et al. 2022) — the deeper antecedent.

## When NOT to use

- **Latency-critical paths** — adds round-trips per iteration.
- **Cost-sensitive paths** — 3-4× generator cost minimum.
- **Tasks without clear quality criteria** — evaluator can't help if "better" is undefined.
- **Tasks where one good prompt suffices** — measure with evals; don't add complexity for nothing.
- **Highly creative tasks** where over-iteration averages out the interesting parts.

## Implementations

| Framework | Evaluator-optimizer support |
|---|---|
| **LangGraph** | Loop-with-conditional-edge pattern; native |
| **LangChain** | Custom chains with feedback loops |
| **OpenAI Agents SDK (2025)** | Sub-agent patterns support this |
| **DSPy** | Optimizer with metrics (programmatic version) |
| **Vercel AI SDK** | Plain loop with `generateText` |
| **Mastra** | Workflow primitives for iteration |
| **Anthropic SDK** | Build with structured outputs + loop in code |
| **Coding-agent specific: Aider, Cursor, Cline** | Test-suite-driven evaluator-optimizer |
| **Custom** | Often the right answer — straightforward to implement |

## Companies / products using evaluator-optimizer

- **Anthropic** ✅ — recommends it explicitly ([source](https://www.anthropic.com/research/building-effective-agents)); used internally for content and code.
- **OpenAI** ✅ — Constitutional-AI-style revision in training; deep-research uses it.
- **GitHub Copilot Workspace** ⚠ — plan-edit-verify-revise loops.
- **Cursor, Aider, Cline** ✅ — test-driven evaluator-optimizer.
- **Cognition (Devin)** ⚠ — extensive use in long-horizon coding.
- **Replit Agent** ⚠ — verify-revise loops.
- **Notion AI, Linear AI** ⚠ — content polish loops.
- **Anthropic Claude (extended thinking + tools)** ⚠ — implicit evaluator-optimizer via tool feedback.

## Further reading

- [Building effective agents](https://www.anthropic.com/research/building-effective-agents) — Anthropic Dec 2024 (canonical workflow patterns)
- [Self-Refine: Iterative Refinement with Self-Feedback](https://arxiv.org/abs/2303.17651) — Madaan et al. 2023
- [Reflexion: Language Agents with Verbal Reinforcement Learning](https://arxiv.org/abs/2303.11366) — Shinn et al. 2023
- [Constitutional AI: Harmlessness from AI Feedback](https://arxiv.org/abs/2212.08073) — Bai et al. 2022
- [DSPy: Compiling Declarative Language Model Calls](https://arxiv.org/abs/2310.03714) — programmatic optimization
- [Eugene Yan — LLM evaluation patterns](https://eugeneyan.com/writing/abstractive/)
- [Patterns for Building LLM-based Systems & Products](https://eugeneyan.com/writing/llm-patterns/)

---

*Diagram source: [`../diagrams/src/evaluator-optimizer.d2`](../diagrams/src/evaluator-optimizer.d2)*

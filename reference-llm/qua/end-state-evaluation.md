# End-State Evaluation

**Aliases:** outcome-based evaluation, terminal-state grading, result-only evaluation, trajectory-agnostic eval
**Category:** Quality & Evals
**Sources:**
[Anthropic — Demystifying evals for AI agents (Jan 2026)](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) ·
[SWE-bench paper](https://arxiv.org/abs/2310.06770) (canonical end-state benchmark) ·
[GAIA paper](https://arxiv.org/abs/2311.12983) (end-state for research agents) ·
production practice: Anthropic, Cognition, OpenAI engineering posts

---

## Problem

> [!TIP]
> **ELI5.** Your agent did 35 things to solve a task. Do you grade *all 35 steps* or *just the final result*? Grading every step seems thorough — but agents are creative, and there are usually many valid paths to a correct outcome. If you grade each step against an "expected path," you'll penalize agents for being smart (taking a different but correct route). The better default: **only grade the end state**. Did the tests pass? Was the right file created? Was the ticket closed? You don't care *how* the agent got there — just whether it got there. Cheaper, more reproducible, and respects the very thing that makes agents useful: they figure out their own paths.

When evaluating agents — as opposed to single-shot LLM calls — there are two fundamentally different things you could grade:

1. **The trajectory.** Every think-act-observe step the agent took along the way. Did it pick the right tool at each turn? Did its reasoning at step 12 look correct? Did it follow the "expected" sequence?
2. **The end state.** What the world looks like after the agent finished. Do the tests pass? Does the file exist with the right contents? Is the ticket closed? Is the answer correct?

The 2023-2024 instinct was to grade trajectories. It felt more thorough — you could catch the agent making bad decisions even if it happened to recover. But grading trajectories turned out to be expensive, brittle, and *penalizes the very flexibility that makes agents valuable*. There's almost never one right way to solve a task; agents that find a *different* correct path get marked down.

By 2026, the consensus — explicit in Anthropic's [January 2026 evals post](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) — is to **default to end-state evaluation** for agentic tasks. Inspect the trajectory only when the end state isn't sufficient (which is rarer than it seems).

## How it works

> [!TIP]
> **ELI5.** Pick a clear, machine-checkable definition of "done correctly" — tests pass, expected output, file diff matches, ticket closed. Then run the agent and check *only that*. Don't care if the agent took 5 turns or 50. Don't care if it used Tool A or Tool B. The whole point of an agent is that it figures out the path. End-state evaluation is what lets you measure that.

![End-state vs trajectory evaluation](../diagrams/svg/end-state-evaluation.svg)

The mechanics are simpler than trajectory eval:

1. **Define the end state.** What does "success" look like as a *checkable property* of the world after the agent runs?
   - For coding agents: tests pass, no lint errors, expected files exist.
   - For research agents: the answer is correct (against a known reference).
   - For customer-service agents: the ticket is in the right resolved state.
   - For data-pipeline agents: the output table matches the expected schema and contains the right rows.
2. **Run the agent.** Let it think, call tools, sub-spawn, retry, do whatever it does. Don't track or grade intermediate steps.
3. **Check the end state.** Code grader (preferred), model grader (if the end state isn't code-checkable), or human (calibration).
4. **Score: 1 or 0.** Done.

That's it. The agent gets a binary verdict per task; you average over many tasks for the eval score.

### Why this is dramatically better than trajectory grading

**Respects agent flexibility.** The whole reason to use an agent (rather than a workflow) is that the agent figures out its own path. Trajectory grading directly penalizes that. If the rubric expects `[read_file, think, write_file]` and the agent does `[grep, read_file, read_test, think, write_file, run_tests]`, the trajectory grader flags it as wrong-but-correct — useless signal.

**Cheaper.** A trajectory eval has to grade every step (maybe 30+ for a multi-hour task). An end-state eval has one grading call. For LLM-as-judge graders, this is 30× cheaper.

**More reproducible.** Trajectory grading is highly sensitive to non-determinism — the same task can produce different trajectories on different runs of the same agent. End-state evaluation either succeeds or fails; the path variance doesn't matter.

**More auditable.** "The tests pass" is auditable. "Step 12's reasoning was acceptable" is subjective.

**Faster to build.** Defining a checkable end state is usually easier than defining a "correct trajectory." Most tasks have an obvious end-state criterion; few have an obvious step-by-step rubric.

### The canonical example: SWE-bench

SWE-bench is the most-cited agentic eval and uses end-state evaluation exclusively. The task is: given a real GitHub issue, produce a patch that resolves it. The grader is: apply the patch, run the project's test suite. Pass or fail.

It doesn't matter:
- How many tool calls the agent made.
- Whether the agent investigated the right file first.
- Whether the agent's reasoning at any step was correct.
- Whether the agent went down dead ends before recovering.

Only: does the patch make the previously-failing test pass without breaking other tests. That single end-state check is the entire grader.

This is part of why SWE-bench became the dominant agentic benchmark: the eval is *simple*, *trustable*, and *meaningful*. Every agentic-coding system can be compared on the same end-state metric.

### When end-state evaluation isn't enough

There are cases — fewer than you'd think — where end-state alone misleads:

- **The agent reaches the right end state for the wrong reason.** It guessed correctly, or it took an unsafe shortcut, or it produced the right output by hallucinating intermediate steps. End-state grading can't tell.
- **The end state is right but the trajectory was dangerously expensive.** The agent spent $50 in tokens for a task that should cost $2. End-state grading misses this; you need a cost-trajectory metric alongside.
- **Safety-critical decisions.** "The agent decided correctly *and* for the right reasons" sometimes matters. For high-stakes deployments, periodic trajectory audits supplement end-state evals.
- **Compliance / audit.** Regulated environments may require evidence of *how* a decision was reached, not just that the right decision was reached.

The solution isn't to grade trajectories all the time — it's to **add trajectory inspection as a supplementary signal when end-state alone is insufficient**. Anthropic's guidance: default to end-state; reach for trajectory inspection only when end-state passes/fails are happening for the wrong reasons.

### Choosing a checkable end state

The hard part of end-state eval is *defining* the end state. Heuristics:

**Prefer code-checkable end states**:
- Tests pass / fail.
- File exists with specific content / hash.
- Database row matches expected.
- HTTP response matches schema.
- Process exited with code 0.

**When code-checkable isn't possible, prefer structured end states**:
- "The agent's final answer contains the string 'Paris'."
- "The final response includes all 5 required JSON fields."
- "The summary covers all 3 main topics."

These can be checked with a small LLM-as-judge prompt that just looks at the final output.

**Avoid free-form end states** when possible:
- "The output is helpful." — Too subjective; needs human grading or extensive rubric.
- "The agent did a good job." — Vague; not a checkable state.

If your task genuinely doesn't have a checkable end state, the task may not be well-suited for agentic automation in the first place — or you need to invest in a better task definition.

### End-state for non-binary success

Many real tasks aren't pure pass/fail. Partial credit comes in:

- **Multiple checkable conditions.** "Tests pass" + "no lint errors" + "documentation updated" — three sub-checks, agent gets 0-3 points.
- **Rubric scoring.** A model grader scores the final output 1-5 against multiple criteria.
- **Cost-adjusted score.** Pass/fail × cost-efficiency factor.
- **Tiered success.** Did the agent complete the primary task? Did it also complete optional sub-tasks?

These are still end-state evaluations — they grade *what the world looks like after the agent runs*, not how the agent got there.

### Hybrid: end-state + post-hoc trajectory inspection

In production, the pragmatic pattern is:

1. **Run end-state eval on every task.** Cheap, fast, the main signal.
2. **For tasks that failed**, do a trajectory inspection. *Why* did it fail? What step led to the wrong path? This is debugging information, not the primary eval signal.
3. **For a small sample of passing tasks**, do trajectory inspection periodically. Spot-check that the agent is succeeding for the right reasons.
4. **Track aggregate trajectory metrics** (avg cost, avg turns, avg sub-agent spawns) separately from the success metric.

This gives you a fast end-state signal as the primary metric and trajectory-level visibility as a *diagnostic*, not a grading dimension.

## Variants & related patterns

- [**Eval-driven development**](eval-driven-development.md) — the methodology this fits inside.
- [**Grader taxonomy**](grader-taxonomy.md) — what grades the end state.
- [**Pass@k vs pass^k**](pass-at-k-vs-pass-power-k.md) — both metrics use end-state success as the per-run signal.
- [**Capability vs Regression evals**](capability-vs-regression-evals.md) — both use end-state grading.
- [**Eval saturation**](eval-saturation.md) — end-state evals can saturate too; need updating.
- **SWE-bench / GAIA / OSWorld / WebArena** — major agentic benchmarks, all end-state.
- **Maker-checker** ([../agt/maker-checker.md](../agt/maker-checker.md)) — runtime end-state checker.

## When NOT to use

- **Safety-critical decisions** where reasoning quality matters as much as outcome. Supplement with trajectory inspection.
- **Compliance contexts** requiring auditable decision trails.
- **When the end state is genuinely ill-defined** (creative tasks, open-ended chat). Use rubric-based end-state grading instead.
- **When you specifically want to debug agent behavior** — trajectory inspection is the right tool, but use it for debugging, not as the primary metric.
- **For training-data collection** where you want to imitate good trajectories (e.g., behavior cloning, RLAIF). Then you do want trajectory data, but not as your primary eval metric.

## Implementations

| Tool / framework | End-state support |
|---|---|
| **Anthropic Workbench** | Configure end-state checkers per eval task |
| **OpenAI Evals** | Native; most evals are end-state by default |
| **Inspect AI** | First-class via `Scorer` functions over final state |
| **Promptfoo** | Assertion framework against final output |
| **SWE-bench harness** | Pure end-state: apply patch, run tests |
| **GAIA harness** | End-state: final answer vs reference |
| **OSWorld, WebArena, MiniWoB++** | End-state checkers (file state, DOM state, etc.) |
| **Custom** | Code grader + assertion is straightforward |

## Companies / projects using end-state evaluation

- **Anthropic** ✅ — explicit recommendation ([source](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)).
- **SWE-bench team (Princeton, CMU)** ✅ — canonical end-state benchmark.
- **GAIA team (HuggingFace, Meta)** ✅ — end-state for research agents.
- **OSWorld, WebArena teams** ✅ — end-state for computer-use agents.
- **OpenAI** ✅ — most public evals (HumanEval, MBPP, etc.) are end-state.
- **Cognition (Devin)** ✅ — SWE-bench end-state results are publicly reported.
- **GitHub Copilot Workspace, Cursor, Aider, Cline** ⚠ — internal evals are end-state.
- **UK AISI** ✅ — Inspect AI framework emphasizes end-state.

## Further reading

- [Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) — Anthropic Jan 2026
- [SWE-bench: Can Language Models Resolve Real-World GitHub Issues?](https://arxiv.org/abs/2310.06770) — Jimenez et al. 2023
- [GAIA: A Benchmark for General AI Assistants](https://arxiv.org/abs/2311.12983) — Mialon et al. 2023
- [OSWorld: Benchmarking Multimodal Agents](https://arxiv.org/abs/2404.07972) — Xie et al. 2024
- [WebArena: A Realistic Web Environment for Autonomous Agents](https://arxiv.org/abs/2307.13854) — Zhou et al. 2023
- [Inspect AI documentation](https://inspect.ai-safety-institute.org.uk/) — emphasizes end-state graders

---

*Diagram source: [`../diagrams/src/end-state-evaluation.d2`](../diagrams/src/end-state-evaluation.d2)*

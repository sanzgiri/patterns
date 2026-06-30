# Parallelization (Sectioning and Voting)

**Aliases:** parallel LLM calls, fan-out, map-reduce for LLMs, self-consistency at scale, best-of-N
**Category:** Workflows
**Sources:**
[Anthropic — Building effective agents (Dec 2024)](https://www.anthropic.com/research/building-effective-agents) ·
[Wang et al. — Self-Consistency Improves Chain-of-Thought (2022)](https://arxiv.org/abs/2203.11171) ·
[OpenAI — Best-of-N sampling techniques](https://platform.openai.com/docs/) ·
production practice: Anthropic engineering, OpenAI evals, large-document processing

---

## Problem

> [!TIP]
> **ELI5.** Sometimes you want to do *many things at once* instead of one after the other. Two distinct shapes: **Sectioning** — split one big task into independent sub-tasks and process them in parallel (translate 50 paragraphs of a doc concurrently, analyze 20 files of a repo at once). **Voting** — run the *same* task K times and aggregate the answers (sample-and-majority-vote for accuracy; best-of-K for quality). They look similar (multiple parallel LLM calls) but solve different problems. Sectioning gets you *throughput*; voting gets you *reliability*.

Anthropic's [Building effective agents](https://www.anthropic.com/research/building-effective-agents) groups two parallel-execution patterns together because they share infrastructure (concurrent LLM calls, async harness, result aggregation) but they have entirely different goals.

**Sectioning** is the LLM equivalent of map-reduce — divide work into independent pieces, process each, combine results. The reason this works: many real tasks have natural independence (paragraphs of a document, files of a codebase, items in a list) that LLMs can exploit if you structure the workflow to expose it. Without explicit sectioning, the model tries to handle everything in one prompt and either runs out of context or produces worse output.

**Voting** is variance reduction. Run the same prompt K times, get K (probably different) answers, then aggregate — majority vote on classifications, average on numeric scores, best-of-K on quality. Variance reduces approximately as 1/√K. This was the original insight behind [Self-Consistency](../fnd/reasoning-prompts.md) (Wang et al. 2022). Voting is what you reach for when *reliability matters more than cost*.

## How it works

> [!TIP]
> **ELI5.** Both shapes: fire off K LLM calls concurrently, wait, aggregate. The difference is what the K calls *do*. **Sectioning**: each call works on a different piece of the input (paragraph 1, paragraph 2, ...). Combine pieces back together at the end. **Voting**: every call works on the same input. Pick the majority answer (or best, or average) at the end. Both cost K× the tokens; sectioning buys throughput, voting buys reliability.

![Parallelization — sectioning and voting](../diagrams/svg/parallelization.svg)

### Sectioning, in detail

Sectioning decomposes a task into N independent sub-tasks, processes each as its own LLM call (in parallel), then aggregates.

**The aggregation step** depends on the task:
- **Concatenation.** Translated paragraphs reassembled in order.
- **Aggregation function.** Per-file summaries combined into a repo-wide overview.
- **Set union.** Entities extracted from each chunk merged into a deduplicated set.
- **Another LLM call** for synthesis. K chunk-summaries → LLM call to produce a coherent whole.

**Examples in production:**
- **Translating a long document.** Split into paragraphs; translate each in parallel; reassemble.
- **Analyzing a large codebase.** Per-file analysis in parallel; orchestrator synthesizes.
- **RAG-style retrieval.** Retrieve K chunks; ask LLM to summarize each in parallel; merge for final answer.
- **Multi-document summarization.** Summarize each doc in parallel; LLM merges.
- **Per-item processing in a queue.** Customer tickets, log lines, support emails — each is an independent LLM call, parallelized across workers.

**Why it works:**
- **Latency.** N parallel calls take roughly max(call_latency), not sum. For N=10 calls of 5s each, total wall-time is ~5s, not 50s.
- **Context budget.** Each call gets a *fresh* context window with only the relevant chunk. Avoids [context rot](../ctx/context-rot.md) on big inputs.
- **Per-chunk specialization.** Each call's prompt can be tuned for its chunk's type (code file vs. README vs. test file, e.g.).
- **Failure isolation.** A bad chunk doesn't poison the whole task; retry just that one.

**Engineering details:**
- **Concurrency caps.** Don't fire 10,000 parallel LLM calls — hit rate limits, lose API connections. Cap at e.g. 50 concurrent.
- **Backpressure.** If a downstream synthesis step is slow, throttle producers.
- **Per-chunk retries.** Independent retry budgets per chunk.
- **Order-aware aggregation.** Sometimes order matters (paragraphs of a translation); preserve it explicitly.
- **Cost telemetry per chunk.** Helps tune chunk size — too small means many calls; too large means context rot per call.

### Voting, in detail

Voting runs the *same* prompt with the *same* input K times — but at a non-zero temperature, so outputs vary. Then aggregate.

**Aggregation methods:**
- **Majority vote.** Most common discrete answer wins. For classification, math, or any task with a single right answer.
- **Best-of-N.** Pick the highest-quality output, where "quality" is judged by another LLM call or a code grader. Used for generation.
- **Average / median.** For numeric outputs.
- **Set union with frequency filter.** "Entities mentioned in ≥3 of 5 runs" — robust extraction.
- **Consensus building.** A final LLM call synthesizes the K outputs.

**Examples in production:**
- **Self-consistency for reasoning** ([reasoning prompts](../fnd/reasoning-prompts.md)). Run CoT K times; majority-vote final answer. Wang et al.'s original use case.
- **Safety checks.** Run K different safety classifiers (or the same classifier K times); flag if any/most disagree.
- **Code generation with best-of-N.** Generate K candidate solutions; run tests against each; pick a passing one.
- **Robust extraction.** Run extraction K times; emit only entities present in ≥ K/2 outputs.
- **Cross-judge evaluation.** Run K different judge models (different vendors); aggregate verdicts.

**Why it works:**
- **Variance reduction.** Individual LLM calls are noisy; K independent samples reduce noise approximately as 1/√K.
- **Tail safety.** A single call can produce a catastrophic output (hallucination, refusal-bypass). K calls with vote suppress tail failures.
- **Confidence signal.** Disagreement among the K samples is itself useful — "the model is confident" vs "the model is unsure" maps to vote unanimity.

**Engineering details:**
- **Temperature matters.** Voting only helps if the K samples are *different*. Temperature 0 produces identical samples; temperature 0.3-0.7 is typical.
- **K matters.** K=3 helps; K=10 is sweet spot; K>20 has diminishing returns and rapidly-growing cost.
- **K different models** is often better than K samples of one model. Cross-vendor voting cancels per-model biases.
- **Aggregator choice matters more than K.** Bad aggregator (e.g., picking first answer) wastes the K samples; good aggregator extracts value.

### Sectioning + voting (combined)

Some tasks benefit from both: section the input, vote per section, aggregate sections. Example: extract entities from a 100-page document.

1. Section the doc into 20 chunks (parallel).
2. For each chunk, vote K=5 times (parallel within the chunk).
3. Per-chunk: take entities present in ≥3 of 5 votes.
4. Across chunks: union into final entity list.

Total LLM calls: 20 × 5 = 100, but all parallel; wall-time roughly the cost of one call. This is the kind of structure RAG-with-quality production pipelines use.

### Cost analysis

Both shapes cost K× more tokens than the single-call baseline. Trade-offs:

**Sectioning** typically *saves* total tokens compared to one big call: each sub-call has only its chunk's context, not the full document. Latency is much better. Cost depends on overhead — chunking is usually a net win.

**Voting** is strictly more expensive — K× the tokens for one task. Use only when accuracy / reliability matters more than cost. Common production heuristic: use voting on a sampled subset (e.g., 10% of traffic) and use single-call for the rest; aggregate signal for monitoring.

### Common mistakes

- **Sectioning when the task isn't independent.** "Translate this conversation paragraph-by-paragraph" loses coreference across paragraphs. Detect coupling before sectioning.
- **Voting with temperature 0.** Same prompt, same temperature 0 → identical outputs. K=1 dressed as K=10.
- **Bad aggregators.** "First sample wins" is K=1. Use majority, best-of-N with a real judge, or LLM synthesis.
- **No concurrency caps.** Hammers your provider's rate limits, then 429s.
- **Order-blind aggregation when order matters.** Preserve sequence for translations / sequential outputs.
- **Voting on tasks with no clear "right answer."** Voting on creative writing produces averaged-down outputs.
- **K too small.** K=2 gives no real benefit. K=3-5 is the minimum for voting to matter.

### When parallel doesn't help

- **Tasks with strong sequential dependencies.** Each step needs the previous step's output — use [prompt chaining](prompt-chaining.md).
- **Cost-sensitive paths.** Voting is K× more expensive; only worthwhile when reliability matters.
- **Inputs too small to section.** Sectioning a 100-word task into 4 parts wastes overhead.
- **Tasks already at the model's ceiling.** Voting K identical-quality outputs doesn't improve them.

### Production patterns

- **Sectioning** is dominant in RAG, document processing, codebase analysis, batch jobs.
- **Voting** is dominant in safety filtering, high-stakes classification, reasoning tasks.
- **Combined** in some research-grade systems where both throughput and reliability matter.

## Variants & related patterns

- [**Prompt chaining**](prompt-chaining.md) — sequential decomposition; complement to parallelization.
- [**Routing**](routing.md) — multi-label routing fans out to parallel branches.
- [**Orchestrator-workers**](orchestrator-workers.md) — orchestrator plans the parallel structure dynamically.
- [**Evaluator-optimizer**](evaluator-optimizer.md) — sometimes uses K-sampling + judge.
- [**Self-Consistency**](../fnd/reasoning-prompts.md) — voting applied to chain-of-thought reasoning.
- [**Maker-checker**](../agt/maker-checker.md) — voting variant where the K runs are different roles.
- [**Pass@k vs pass^k**](../qua/pass-at-k-vs-pass-power-k.md) — measures the impact of K sampling.
- **Map-Reduce** (Dean & Ghemawat) — the system-design ancestor.
- **Best-of-N sampling** — voting variant from text generation literature.

## When NOT to use

- **Sequentially dependent tasks.**
- **Single-shot small tasks** where overhead exceeds benefit.
- **Cost-extreme paths** for voting (sectioning is usually fine).
- **Tasks already saturated** at the model's ceiling.

## Implementations

| Framework | Sectioning | Voting |
|---|---|---|
| **LangGraph** | Native parallel branches | Self-consistency wrappers |
| **LangChain** | `RunnableParallel`, batch APIs | `SelfConsistencyChain` |
| **OpenAI batch API** | Bulk-processing API | Manual K-sample |
| **Anthropic batch API** | Same | Same |
| **Vercel AI SDK** | `Promise.all` over `generateText` | K-sample + aggregator |
| **DSPy** | Programmatic | Built-in `BestOfN`, ensembling |
| **Promptfoo, Inspect AI** | Eval-time parallel | Multi-sample evals |
| **Mastra, LlamaIndex** | Parallel pipeline primitives | Voting-aware |
| **Plain async (asyncio, Promise.all)** | Often the right answer | Same |

## Companies / products using parallelization

- **Anthropic** ✅ — multi-agent research system uses sectioning extensively ([Jun 2025 post](https://www.anthropic.com/engineering/built-multi-agent-research-system)).
- **OpenAI** ✅ — Operator and ChatGPT use parallel sub-tasks.
- **GitHub Copilot Workspace** ⚠ — parallel codebase analysis.
- **Cognition (Devin)** ⚠ — long-horizon tasks parallelize file-level analysis.
- **Perplexity, You.com** ⚠ — parallel retrieval across sources.
- **AssemblyAI, Speechmatics, Deepgram** ⚠ — parallel transcript-then-LLM pipelines.
- **Enterprise document-processing** ✅ — Microsoft 365 Copilot, Notion AI, Box AI rely on sectioning.

## Further reading

- [Building effective agents](https://www.anthropic.com/research/building-effective-agents) — Anthropic Dec 2024
- [Self-Consistency Improves Chain of Thought Reasoning](https://arxiv.org/abs/2203.11171) — Wang et al. 2022
- [How we built our multi-agent research system](https://www.anthropic.com/engineering/built-multi-agent-research-system) — Anthropic Jun 2025
- [LangGraph parallel branches docs](https://langchain-ai.github.io/langgraph/)
- [Best-of-N sampling and verifier-guided decoding](https://arxiv.org/abs/2110.14168) — Cobbe et al. (gsm8k paper, foundational best-of-N)
- [DSPy ensembling](https://dspy-docs.vercel.app/)

---

*Diagram source: [`../diagrams/src/parallelization.d2`](../diagrams/src/parallelization.d2)*

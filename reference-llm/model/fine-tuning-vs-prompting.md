# Fine-Tuning vs Prompting vs RAG

**Aliases:** the specialization trinity, when to fine-tune, prompt engineering vs fine-tuning, retrieval vs training
**Category:** Model-level Patterns
**Sources:**
[OpenAI fine-tuning documentation](https://platform.openai.com/docs/guides/fine-tuning) ·
[Anthropic fine-tuning docs](https://docs.anthropic.com/en/docs/build-with-claude/fine-tuning) (via Bedrock) ·
[HuggingFace PEFT library](https://huggingface.co/docs/peft) ·
[Hu et al. — LoRA (2021)](https://arxiv.org/abs/2106.09685) ·
[Dettmers et al. — QLoRA (2023)](https://arxiv.org/abs/2305.14314) ·
[Sebastian Raschka — Practical Tips for Finetuning LLMs](https://magazine.sebastianraschka.com/) ·
[Patterns for Building LLM-based Systems & Products](https://eugeneyan.com/writing/llm-patterns/) — Eugene Yan

---

## Problem

> [!TIP]
> **ELI5.** Three ways to make an LLM do what you want. **Prompting**: just write a better prompt. Cheap, fast to iterate, but limited. **RAG**: inject relevant facts at query time. Great for "the model doesn't know my data." **Fine-tuning**: actually update the model's weights using your examples. Bakes in behavior or style; expensive and slow to iterate. The right choice depends on WHAT you're trying to change. Need access to current/private FACTS? RAG. Need consistent FORMAT or BEHAVIOR? Fine-tune. Just want better output from the base model? Better prompt. Most teams pick the wrong one — the most common mistake is fine-tuning when they should have used RAG (fine-tuning doesn't reliably teach facts) or prompting (much cheaper).

The decision flow is one of the most-asked questions in LLM app development. The classic answer (echoed by Eugene Yan, OpenAI, Anthropic, every major teaching resource): **start with prompting; add RAG for facts; fine-tune only as a last resort for specific narrow needs**.

Why fine-tuning is misunderstood:
- "Just train it on my data" SOUNDS like teaching the model facts.
- It's actually closer to teaching the model *behavior patterns*.
- Fine-tuning on facts often gives mediocre fact recall (vs. RAG) AND degrades general capability.
- Modern lightweight fine-tuning (LoRA, QLoRA) made FT cheaper, but didn't change the fundamental trade-off.

The 2024-2026 ecosystem has crystallized this: prompting + RAG handles 80-90% of use cases; fine-tuning is reserved for specific narrow problems where it shines.

## How it works

> [!TIP]
> **ELI5.** Three orthogonal ways to specialize an LLM. Prompting changes WHAT it sees this call (system prompt, examples). RAG changes WHAT FACTS it has access to (retrieved chunks). Fine-tuning changes the MODEL ITSELF (weights). They're complementary — most production systems use prompting + RAG, with occasional fine-tuning for narrow specialty.

![Fine-tuning vs prompting vs RAG](../diagrams/svg/fine-tuning-vs-prompting.svg)

### Prompting: change the prompt

Includes everything you do to the input WITHOUT changing model weights or adding retrieval:

- **System prompt**: role, persona, constraints
- **Few-shot examples**: demonstrations of desired behavior
- **Chain-of-thought**: ask the model to reason
- **Output schema instructions**: format constraints
- **Tool descriptions**: what the model can call
- **Style instructions**: tone, formality, length

What prompting CAN change:
- Output format (with effort + few-shot examples)
- Tone and style (within model's range)
- Tool use behavior
- Reasoning approach
- Refusal behavior (within safety training)

What prompting CAN'T change:
- The model's underlying knowledge (it knows what it knew at training time)
- Truly novel capabilities
- Behaviors the model is RL-trained against
- Strong personality changes against its post-training

**When to use**: always start here. Cheap to iterate. No infrastructure. Apply to every problem first.

### RAG: change what the model knows

Detailed in dedicated pages: [basic-rag](../ret/basic-rag.md), [advanced-rag](../ret/advanced-rag.md), [contextual-retrieval](../ret/contextual-retrieval.md), [graph-rag](../ret/graph-rag.md), [reranking](../ret/reranking.md).

RAG injects facts at query time. The model doesn't *learn* the facts — it uses them in context.

What RAG CAN change:
- The model knowing about your private docs, codebase, KB
- Up-to-date information (current vs. training cutoff)
- Citations and provenance
- Per-tenant data isolation

What RAG CAN'T change:
- The model's underlying reasoning ability
- Format / style consistency
- Behavior on tasks where retrieval doesn't help

**When to use**: anytime you need access to facts the base model doesn't have. Default for "make the model know X." 80%+ of "fine-tune the model on my docs" requests should be RAG.

### Fine-tuning: change the weights

Actually modifying the model parameters via additional training:

- **Full fine-tuning**: update all weights. Expensive; mostly historical.
- **LoRA** ([Hu et al. 2021](https://arxiv.org/abs/2106.09685)): inject small trainable "adapter" matrices alongside frozen base weights. ~0.1-1% of params trained. Most common.
- **QLoRA** ([Dettmers et al. 2023](https://arxiv.org/abs/2305.14314)): LoRA + 4-bit quantization of base. Run on a single consumer GPU.
- **Adapters, prefix tuning, prompt tuning, P-tuning**: family of parameter-efficient methods.
- **PEFT library** (HuggingFace) — the OSS standard for these techniques.

What fine-tuning CAN change well:
- **Output FORMAT consistency** (always emits this JSON schema, etc.)
- **Style and tone** (sound like our brand voice)
- **Narrow task performance** (smaller model matching larger on one task)
- **Tool-calling reliability** (always uses tools correctly)
- **Refusal behavior** (refuse this domain; comply on that one)
- **Latency / cost reduction** (fine-tuned small model replaces big general model)

What fine-tuning DOESN'T do well:
- **Teaching new facts** (RAG is dramatically better; FT facts get fuzzy + hallucinate)
- **Open-ended capability** (general capability often *degrades*)
- **Frequently updated knowledge** (re-training is expensive)
- **Per-user personalization** (one model per user is impractical)

### Cost comparison (rough 2026 numbers)

| Approach | Engineering cost | Inference cost | Time to first result |
|---|---|---|---|
| **Better prompt** | Hours | Same as base | Same hour |
| **RAG** | Days-weeks | + retrieval + larger context | Days |
| **LoRA fine-tune (cloud)** | Days-weeks | ~Same as base | Days-weeks |
| **Full fine-tune** | Weeks-months | Same as base | Weeks-months |

The cost-to-result ratio is why "try prompting first, RAG second, FT last" is universal advice.

### The decision tree

```
Does the model need access to YOUR DATA (private, recent, specific)?
  YES -> RAG
  NO  -> continue

Is the output FORMAT or STYLE inconsistent despite good prompts?
  YES -> Consider fine-tuning
  NO  -> continue

Is the base model just not following instructions well?
  YES -> Try better prompts, few-shot examples first
  NO  -> Probably you're done; no specialization needed
```

A common production stack: fine-tune a small model for **format consistency** (it always emits valid JSON with the right schema) + RAG for **facts** + system prompt for **orchestration**. Each addresses a different problem.

### When fine-tuning genuinely wins

Real production wins for fine-tuning:

- **Sentiment / classification at scale**: fine-tuned small model beats large general model on cost.
- **Domain-specific tone**: brand voice, legal writing style, medical communication.
- **Tool-call format reliability**: always emits the right tool call shape.
- **Code style enforcement**: outputs match team conventions.
- **Refusal calibration**: customize what the model refuses or complies with.
- **Distillation**: compress capability of a large model into a small one for a specific task.

In each case, RAG and prompting alone don't get to the quality bar.

### When fine-tuning is the wrong call

Common anti-patterns where teams reach for FT but shouldn't:

- "Teach the model about our product." → RAG. FT doesn't memorize facts reliably.
- "Make it sound like our docs." → Few-shot + system prompt. FT is overkill.
- "It hallucinates customer names." → RAG with the customer DB.
- "Make it use our API." → Tool descriptions + structured outputs.
- "Personalize for each user." → Per-user prompt + per-user RAG, not per-user FT.
- "Update on new product launches." → RAG with refreshed index.

### Modern lightweight fine-tuning (the LoRA family)

Pre-LoRA, fine-tuning required updating all model weights — expensive, slow, hardware-intensive. LoRA changed this:

- **LoRA**: train small adapter matrices that approximate weight updates. Inject at inference. Train ~0.1-1% of total params.
- **QLoRA**: base model in 4-bit quantization; LoRA adapters in fp16. Runs on a single 24GB consumer GPU for many model sizes.
- **PEFT library**: HuggingFace's open-source implementation.

These made fine-tuning cheaper and faster but didn't change the trade-off about WHAT FT is good for.

### Hosted fine-tuning options (2026)

| Provider | Fine-tuning support |
|---|---|
| **OpenAI** | GPT-4o-mini fine-tuning, GPT-4o fine-tuning, RFT (reinforcement fine-tuning) for o-series |
| **Anthropic (via AWS Bedrock)** | Claude Haiku fine-tuning |
| **Google Vertex** | Gemini Flash / Pro fine-tuning |
| **Open-source (self-host)** | Any open-weights model + PEFT |
| **Together, Anyscale, Fireworks** | Hosted fine-tuning of open models |
| **AWS Bedrock** | Fine-tuning across hosted models |

### Anti-patterns

- **Fine-tuning to teach facts.** RAG is better.
- **Fine-tuning before trying prompting.** Skipping the cheap option.
- **Tiny fine-tuning dataset** (<100 examples). Usually doesn't help.
- **Fine-tuning frequently** (weekly retrains). Operational nightmare.
- **No eval set comparing FT to baseline.** Can't tell if it helped.
- **Fine-tuning hurts general capability.** Always eval on a broader set, not just your task.
- **Full fine-tuning when LoRA would do.** Wasted resources.
- **Fine-tuning instead of fixing the prompt.** The prompt was the problem.
- **Treating fine-tuning as a substitute for understanding** the model's failure mode.

### Production engineering

- **Eval set**: known input → expected output pairs. Required to measure FT effectiveness.
- **Baseline comparison**: always compare FT model to base + prompt-only.
- **Hold-out set**: never trained on; used to detect overfit.
- **Capability eval**: ensure FT didn't degrade general reasoning, math, etc.
- **Version pinning**: which base model version was fine-tuned matters.
- **Retraining cadence**: monthly? quarterly? balance cost vs. drift.
- **Deployment**: rainbow rollout ([rainbow deployments](../eng/rainbow-deployments.md)) for FT models.
- **Cost telemetry**: training cost + per-call inference cost vs. unmodified base.

## Variants & related patterns

- [**Augmented LLM**](../fnd/augmented-llm.md) — the three augmentations: instructions (prompt), context (RAG), tools.
- [**Basic RAG**](../ret/basic-rag.md) — the RAG alternative.
- [**Advanced RAG**](../ret/advanced-rag.md) — what RAG looks like in 2026.
- [**Prompting fundamentals**](../fnd/prompting-fundamentals.md) — the start-here approach.
- [**Reasoning prompts**](../fnd/reasoning-prompts.md) — strong prompt-only technique.
- [**Preference tuning**](preference-tuning.md) — DPO/KTO/ORPO; the alignment cousin of FT.
- [**Brain vs hands**](../eng/brain-vs-hands.md) — fine-tuned small model can be the "hands."
- [**Memory architectures**](../mem/memory-architectures.md) — semantic memory is sometimes called "fine-tuning-lite."
- **PEFT library** — the OSS standard.
- **Distillation** — special form of fine-tuning (small model trained on large model outputs).

## When NOT to fine-tune

- **You haven't tried prompting yet.**
- **You haven't tried RAG yet.**
- **Your problem is fact recall, not behavior.**
- **You have <100 high-quality examples.**
- **You can't measure improvement vs baseline.**
- **You can't retrain when the base model updates.**
- **Your data changes frequently.**

## Implementations

| Approach | Tools |
|---|---|
| **Prompting** | Any LLM SDK; DSPy for programmatic optimization |
| **RAG** | LangChain, LlamaIndex, Haystack, vector DBs, hosted RAG services |
| **Fine-tuning (hosted)** | OpenAI FT API, AWS Bedrock FT, Google Vertex FT, Together, Fireworks |
| **Fine-tuning (self-host)** | HuggingFace TRL + PEFT, Axolotl, LLaMA-Factory, Unsloth, MLX-FineTuner |
| **LoRA / QLoRA** | PEFT (HuggingFace) |
| **Distillation** | DistillKit, custom |
| **RFT (reinforcement fine-tuning)** | OpenAI (2025+), custom |
| **DPO and friends** | TRL, Axolotl ([preference-tuning](preference-tuning.md)) |

## Companies / products fine-tuning at scale

- **OpenAI** ✅ — productizes fine-tuning APIs; uses it internally extensively.
- **Anthropic** ⚠ — does internal FT; offers via Bedrock to customers.
- **Google** ✅ — Vertex FT for Gemini; internal FT for product launches.
- **Customer-support AI** (Intercom Fin, Salesforce, Ada) ⚠ — fine-tune for tone / format consistency.
- **Klarna, Stripe, Shopify** ⚠ — fine-tune for domain tone.
- **Bloomberg GPT** ✅ — fine-tuned for financial domain (an extreme example).
- **Most production LLM teams** ⚠ — selectively fine-tune for narrow tasks.

## Further reading

- [OpenAI fine-tuning docs](https://platform.openai.com/docs/guides/fine-tuning)
- [Anthropic fine-tuning (via Bedrock)](https://docs.anthropic.com/en/docs/build-with-claude/fine-tuning)
- [LoRA paper](https://arxiv.org/abs/2106.09685) — Hu et al. 2021
- [QLoRA paper](https://arxiv.org/abs/2305.14314) — Dettmers et al. 2023
- [Patterns for Building LLM-based Systems & Products](https://eugeneyan.com/writing/llm-patterns/) — Eugene Yan
- [Sebastian Raschka's blog](https://magazine.sebastianraschka.com/) — many practical FT posts
- [HuggingFace PEFT](https://huggingface.co/docs/peft)
- [Axolotl](https://github.com/axolotl-ai-cloud/axolotl) — popular fine-tuning library
- [Unsloth](https://github.com/unslothai/unsloth) — fast/efficient open-source FT

---

*Diagram source: [`../diagrams/src/fine-tuning-vs-prompting.d2`](../diagrams/src/fine-tuning-vs-prompting.d2)*

# Preference Tuning (DPO, KTO, ORPO, GRPO)

**Aliases:** alignment from preferences, direct preference optimization, post-RLHF methods, offline RLHF, preference learning
**Category:** Model-level Patterns
**Sources:**
[Rafailov et al. — Direct Preference Optimization (May 2023)](https://arxiv.org/abs/2305.18290) (canonical DPO paper) ·
[Ethayarajh et al. — KTO: Model Alignment as Prospect-Theoretic Optimization (Feb 2024)](https://arxiv.org/abs/2402.01306) ·
[Hong et al. — ORPO: Monolithic Preference Optimization without Reference Model (Mar 2024)](https://arxiv.org/abs/2403.07691) ·
[DeepSeek-Math GRPO (Feb 2024)](https://arxiv.org/abs/2402.03300) (Group Relative Policy Optimization, used in DeepSeek-R1) ·
[Bai et al. — Constitutional AI / RLAIF (2022)](https://arxiv.org/abs/2212.08073) ·
[HuggingFace TRL library](https://huggingface.co/docs/trl) — the production-grade implementation

---

## Problem

> [!TIP]
> **ELI5.** After [fine-tuning](fine-tuning-vs-prompting.md) on examples of "do this task," you can do better by also showing the model PAIRS of outputs — one preferred, one rejected — and training it to favor preferred outputs. This is "alignment from preferences." The original method (RLHF) was complex: train a reward model + use PPO reinforcement learning. **DPO** (May 2023) was the breakthrough: directly optimize the LLM on preference pairs with a simple classifier-style loss. No reward model, no PPO. Same quality, vastly simpler. By 2024 DPO was the default; **KTO**, **ORPO**, and **GRPO** are extensions handling different data shapes or training stages. This is how virtually every frontier open-weights model (Llama 3+, Mistral, Qwen, Gemma 2+, DeepSeek-V3+) was aligned in 2024-2026.

The pre-DPO state of the art was **RLHF** (Reinforcement Learning from Human Feedback, OpenAI 2022, used to align GPT-3.5 → ChatGPT). RLHF had three stages: supervised fine-tuning, train a reward model on human preference pairs, then PPO-optimize the LLM to maximize reward.

The pain points of RLHF:
- **Three stages, three loss functions**. Many hyperparameters.
- **Separate reward model**. More compute, more drift between RM and LLM.
- **PPO is unstable**. Easy to mode-collapse; hard to reproduce.
- **Expensive**. Most teams couldn't afford it.

Rafailov et al.'s May 2023 paper [Direct Preference Optimization](https://arxiv.org/abs/2305.18290) showed mathematically that the RLHF objective can be expressed *directly* as a loss on preference pairs — bypassing the reward model and PPO entirely. The slogan: "Your language model is secretly a reward model."

DPO immediately became the standard. Within a year, KTO, ORPO, and other variants filled in the gaps for non-paired data and training-stage compression. By DeepSeek-R1 (Jan 2025), the related GRPO had emerged as the method behind reasoning-model training.

For app developers: this is mostly someone else's problem (the base model already went through preference tuning before you ever called it). For model builders / open-source fine-tuners / R&D teams: this is core methodology.

## How it works

> [!TIP]
> **ELI5.** Take pairs of model outputs `(preferred, rejected)` for the same prompt. DPO uses a simple loss: increase the model's probability of producing preferred outputs and decrease it for rejected ones, RELATIVE to a frozen reference copy of the model. No reward model. No PPO. Just supervised-style training on preference data. Works.

![Preference tuning](../diagrams/svg/preference-tuning.svg)

### DPO — the canonical method

Given preference pairs `(prompt, chosen, rejected)`:

The DPO loss for a single example:
```
L = -log σ(β · [log π_θ(chosen|prompt)/π_ref(chosen|prompt)
              - log π_θ(rejected|prompt)/π_ref(rejected|prompt)])
```

Where:
- `π_θ` is the model being trained
- `π_ref` is a frozen reference model (typically the SFT-trained checkpoint)
- `β` is a hyperparameter (typically 0.1)

Intuitively: increase `π_θ`'s odds of `chosen` over `rejected` (compared to the reference), but don't drift too far from the reference (the β regularizes).

**Why it works**: mathematically equivalent to the RLHF objective if reward = `β · log(π_θ/π_ref)`. The model implicitly defines a reward. So you can train directly on preferences without a separate RM.

**What it needs**: a SFT checkpoint as starting point + reference, and high-quality preference pairs.

### KTO — when you don't have pairs

Many real datasets have **unpaired** judgments: a list of "good outputs" and a list of "bad outputs" — not paired by prompt. Or thumbs up/down feedback on individual responses.

KTO ([Ethayarajh et al. Feb 2024](https://arxiv.org/abs/2402.01306)) optimizes on these unpaired labels. The loss is grounded in prospect theory (Kahneman-Tversky) — losses weigh more than equal-size gains.

Practical advantages:
- **Easier data collection** (thumbs up/down feedback from real users)
- **Doesn't require generating matched pairs** (which is expensive)
- **Often matches DPO quality** when paired data is the same total size

Used in: tasks where you collect feedback rather than preference comparisons.

### ORPO — combine SFT + preferences in one stage

ORPO ([Hong et al. Mar 2024](https://arxiv.org/abs/2403.07691)) combines supervised fine-tuning AND preference optimization in a single training stage with one loss.

- Adds an "odds ratio" preference loss to standard SFT
- No reference model needed (saves memory)
- Single-stage training (saves time)

Used in: efficiency-sensitive open-source training (Llama derivatives, Mistral derivatives). When you want one training run instead of two.

### GRPO — group-relative reward (DeepSeek)

GRPO ([DeepSeek-Math paper, Feb 2024](https://arxiv.org/abs/2402.03300)) was the method behind DeepSeek-R1's reasoning RL training.

- Sample K outputs per prompt
- Compute a reward for each (e.g., correctness via rule-based grader)
- Use group-relative advantages (rewards normalized within the group)
- Update policy with REINFORCE-style gradient

GRPO sits between DPO (offline, paired) and PPO (online, reward-modeled). It enabled DeepSeek-R1's reasoning training without a learned reward model.

Used in: reasoning-model training, code/math RL, scenarios where rule-based rewards are available.

### Other variants

The field has many close cousins:
- **IPO** (Identity Preference Optimization) — fixes a theoretical DPO issue.
- **SimPO** (Simple Preference Optimization) — DPO without reference model.
- **CPO** (Contrastive Preference Optimization) — translation focus.
- **DPOP** — handles edge cases in DPO.
- **NCA** (Noise Contrastive Alignment) — alternative loss formulation.
- **Step-DPO** — applies DPO at step-level for reasoning.

For most teams, vanilla DPO is fine. The variants address specific issues (data shape, training stage, theoretical concerns).

### Where preference data comes from

- **Human annotators** — preferred originally; gold standard.
- **AI-feedback (RLAIF)** — use a strong model (Claude, GPT-4) as judge; cheap and scalable.
- **Rejection sampling** — sample many outputs from a strong model; keep best as "chosen."
- **Constitutional AI** ([Bai et al. 2022](https://arxiv.org/abs/2212.08073)) — model critiques and revises its own outputs against constitutional rules; pairs from before/after.
- **User feedback** — thumbs up/down on production responses; usable with KTO.
- **Rule-based** — for math / code where correctness is checkable (used in GRPO).

### Cost analysis

| Method | Compute cost | Data cost | Engineering complexity |
|---|---|---|---|
| **SFT only** | Low | Low (instructions) | Low |
| **DPO** | Medium (training + ref model) | Medium (paired prefs) | Low |
| **KTO** | Medium | Low (unpaired labels) | Low |
| **ORPO** | Medium | Medium | Low (single stage) |
| **GRPO** | High (online RL) | Low if rule-based | Medium |
| **RLHF (PPO)** | High | High (paired prefs) | High |

DPO became dominant because the cost/quality ratio is best for most use cases.

### What preference tuning changes

- **Tone and helpfulness**: model becomes more pleasant, less evasive.
- **Refusal calibration**: refuses correctly more often, less often unnecessarily.
- **Factual grounding**: with the right preference signal, hallucinates less.
- **Format adherence**: follows instructions for output shape.
- **Reasoning quality** (with GRPO + reward-based signal): the core mechanism behind reasoning models.

What it DOESN'T change:
- **Underlying knowledge** (still need RAG / FT for facts).
- **Open-ended capability** (preference signal is narrow).
- **Personality vs. training data** — preferences mostly reinforce learned patterns.

### Who does this work

- **Foundation model builders** (Anthropic, OpenAI, Google, Meta, Mistral, Qwen, DeepSeek): preference tuning is part of every model's post-training.
- **Open-source fine-tuners**: extensively, on derivatives of base models.
- **R&D teams** working on aligned models for specific industries.
- **Most app developers DON'T do preference tuning themselves**. They use the already-tuned base model.

### Anti-patterns

- **Preference tuning without SFT first.** Need a good starting checkpoint.
- **Too few examples.** DPO typically needs 1K-100K pairs to show improvement.
- **Low-quality preference data.** Bad preferences teach bad behavior; "garbage in, garbage out."
- **No held-out eval.** Can't tell if it helped general capability.
- **Reward hacking.** Model finds a way to satisfy the preference signal without actually being better.
- **Mode collapse.** Model converges on a narrow output style; loses diversity.
- **Ignoring KL regularization.** Without it, model drifts arbitrarily far from reference.
- **DPO on math/code without checkable rewards.** GRPO is better here.

### Production engineering

- **Eval suite** including capability evals (don't just measure on preference data).
- **Held-out preference test set** to check generalization.
- **Diversity metrics** — preference tuning can mode-collapse; monitor.
- **Iteration**: typically multiple rounds of (collect prefs → train → deploy → collect more).
- **Hyperparameter tuning**: β in DPO matters; tune it.
- **Reference model choice**: SFT checkpoint is typical; some variants use base model.
- **Data quality > data quantity**: 5K high-quality pairs > 100K noisy pairs.

### Trajectory through 2026

- **Online RL with checkable rewards** (DeepSeek-style GRPO) is dominant for reasoning-model training.
- **Constitutional AI / RLAIF** (AI-generated preferences) scaling beyond human feedback.
- **Step-level preferences** for long-form / multi-step outputs.
- **Combined SFT + DPO single-stage methods** (ORPO and successors).
- **Adaptive preference collection**: actively identify what preferences are needed.
- **Open-source preference datasets**: UltraFeedback, HH-RLHF, etc. are common training inputs.

For most app developers, this remains foundation-model-builder concern. For open-source / specialized model builders, it's central technology.

## Variants & related patterns

- [**Fine-tuning vs prompting**](fine-tuning-vs-prompting.md) — preference tuning is a fine-tuning variant.
- [**Reasoning models**](reasoning-models.md) — GRPO is behind reasoning-model training.
- [**Augmented LLM**](../fnd/augmented-llm.md) — preference tuning aligns the atomic LLM behavior.
- [**Maker-checker**](../agt/maker-checker.md) — constitutional AI is preference-tuning-as-self-critique.
- [**Eval-driven development**](../qua/eval-driven-development.md) — eval sets validate preference-tuning impact.
- [**Brain vs hands**](../eng/brain-vs-hands.md) — different tiers may go through different preference tuning.
- **RLHF** — the predecessor (still used in some setups).
- **Constitutional AI** — alternative to human preferences.
- **PPO** — the optimizer DPO replaced.
- **RLAIF** — AI-generated preferences.

## When NOT to use

- **Tiny preference dataset** (<1K examples).
- **No SFT checkpoint** to use as reference.
- **No held-out capability eval** (can't tell if you hurt general performance).
- **You're an app developer** (base model already preference-tuned).
- **Your problem is fact recall** (FT or RAG, not preference tuning).
- **Your problem is task specialization** (SFT alone may suffice).

## Implementations

| Library | Methods supported |
|---|---|
| **HuggingFace TRL** | DPO, KTO, ORPO, IPO, CPO, GRPO, RLHF (PPO), Constitutional AI |
| **Axolotl** | DPO, KTO, ORPO, full + LoRA |
| **LLaMA-Factory** | DPO, KTO, ORPO, RLHF |
| **Unsloth** | DPO, KTO (efficient) |
| **OpenRLHF** | DPO, PPO, GRPO, ReMax, large-scale |
| **DeepSpeed Chat** | RLHF, DPO |
| **OpenAI Fine-tuning** | RFT (reinforcement fine-tuning) on o-series (preference-tuning-adjacent) |
| **AWS Bedrock** | Custom preference fine-tuning for select models |

## Companies / products using preference tuning

- **Every major foundation model lab** ✅ — Anthropic, OpenAI, Google, Meta, Mistral, Cohere, Alibaba, DeepSeek all use preference tuning in post-training.
- **Llama 3 / 3.1 / 4** ✅ — Meta uses DPO + variants.
- **Mistral, Qwen, Gemma, Phi** ✅ — preference tuning is standard.
- **DeepSeek-R1 / V3** ✅ — GRPO behind reasoning training.
- **HuggingFace community models** ✅ — TRL-based DPO is everywhere.
- **Stable Diffusion / image gen** ⚠ — DPO variants used in image-model alignment.
- **Anthropic Constitutional AI** ✅ — preference-tuning-from-AI-feedback variant.

## Further reading

- [Direct Preference Optimization](https://arxiv.org/abs/2305.18290) — Rafailov et al. May 2023 (canonical DPO)
- [KTO: Model Alignment as Prospect-Theoretic Optimization](https://arxiv.org/abs/2402.01306) — Ethayarajh et al. Feb 2024
- [ORPO: Monolithic Preference Optimization](https://arxiv.org/abs/2403.07691) — Hong et al. Mar 2024
- [DeepSeek-Math (GRPO)](https://arxiv.org/abs/2402.03300) — DeepSeek Feb 2024
- [DeepSeek-R1 Technical Report](https://arxiv.org/abs/2501.12948) — DeepSeek Jan 2025
- [Constitutional AI](https://arxiv.org/abs/2212.08073) — Bai et al. (Anthropic) 2022
- [HuggingFace TRL library](https://huggingface.co/docs/trl)
- [Sebastian Raschka — DPO and friends](https://magazine.sebastianraschka.com/)
- [Argilla preference datasets](https://argilla.io/) — preference data tooling

---

*Diagram source: [`../diagrams/src/preference-tuning.d2`](../diagrams/src/preference-tuning.d2)*

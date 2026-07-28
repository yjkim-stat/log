---
layout: post
title: "SuperThoughts — Reasoning Tokens in Superposition"
date: 2026-07-28 06:15:00 +0900
description: >
  Paper review of SuperThoughts (Xiong, Garg, Yu, Shrivastava, Zhao,
  Kyrillidis, Papailiopoulos, ICML 2026, UW-Madison / Microsoft
  Research / Princeton / Rice) — compressing pairs of consecutive
  chain-of-thought tokens into a single latent representation and
  decoding two tokens per step via a lightweight multi-token-prediction
  module, with a confidence-gated fallback to standard decoding on hard
  reasoning steps. ~20-30% CoT length reduction near accuracy parity.
tags: [reasoning, efficiency, latent-reasoning, inference, llm]
categories: paper-review
toc:
  sidebar: left
related_posts: true
---

**Paper.** Zheyang Xiong, Shivam Garg, Max Yu, Vaishnavi Shrivastava,
Haoyu Zhao, Anastasios Kyrillidis, Dimitris Papailiopoulos.
*SuperThoughts: Reasoning Tokens in Superposition.*
ICML 2026. University of Wisconsin–Madison, Microsoft Research,
Princeton University, Rice University.
[[arXiv]](https://arxiv.org/abs/2606.13862)

---

## 0. The Picture in One Paragraph

Long chain-of-thought reasoning is expensive not because of the total
number of FLOPs alone, but because tokens are generated strictly
**sequentially** — one forward pass, one token, repeat. Prior attempts
to reason in continuous latent space (Coconut, CODI, CoLaR) sidestep
discrete decoding but tend to be unstable to train and don't scale
past short reasoning chains. **SuperThoughts** takes a middle path:
during the CoT phase, a **Compressor** module folds each *pair* of
consecutive CoT tokens into a single latent vector; from that one
latent, a **Main module** predicts the next token and a lightweight
**Multi-Token Prediction (MTP)** module predicts the token after that
— two tokens decoded per forward step, while discrete token
supervision is preserved throughout training (the key stabilizer
missing from pure continuous-latent methods). Because compressing
every step this aggressively is fragile on hard reasoning tokens, a
**confidence-based adaptive mechanism** checks the MTP module's max
softmax probability at each step against a threshold: below it, the
model falls back to the slower, more reliable Main-module decoding.
Evaluated on Qwen2.5-Math models, this yields roughly **20–30% CoT
length reduction with near-parity accuracy** on MATH500, AMC,
OlympiadBench, and GPQA-Diamond.

---

## 1. The Problem — Sequential Decoding Is the Bottleneck

Standard autoregressive CoT generates one token per forward pass, in
order. Even if two consecutive tokens are jointly "easy" to predict
together, the model still pays for two full forward passes. Prior
attempts to fix this by reasoning in continuous latent space entirely
(no discrete tokens at all) run into two problems: **training
instability** (there's no clean supervision signal for a latent that
was never meant to be decoded) and **short-horizon-only** effectiveness
— Coconut, CODI, and CoLaR are reported to work for reasoning chains
of roughly 20–60 tokens but don't transfer to the longer, more complex
chains typical of real math/reasoning benchmarks.

---

## 2. The SuperThoughts Method

### 2.1 Three modules

- **Compressor**: encodes a pair of consecutive CoT tokens into one
  latent vector.
- **Main module**: from that latent, predicts the immediate next
  token — this is the "primary," higher-quality decoding path.
- **MTP module**: a lightweight head that, from the same latent,
  predicts the token *after* the Main module's prediction — a second
  token "in superposition" with the first, decoded without an
  additional full forward pass.

### 2.2 Two-stage training

1. **Latent-space alignment**: the compressed latent space is
   distilled from a discrete-token teacher, so the compressed
   representation starts out anchored to real token semantics rather
   than drifting into an arbitrary continuous space.
2. **Joint end-to-end training**: all three modules are trained
   together, with **discrete token supervision preserved throughout**
   — unlike Coconut-style approaches that drop discrete supervision
   once latent reasoning kicks in. This is the paper's claimed source
   of training stability at longer horizons.

### 2.3 Confidence-gated adaptive inference

At each step, the MTP module's max softmax probability is compared
against a threshold τ. If confidence is high, the fast two-token
superposition path is used. If confidence falls below τ, the model
discards the MTP shortcut and falls back to feeding the latent into
the Main module for standard, single-token decoding — self-regulating
speed so that easy spans compress aggressively while hard reasoning
steps get full-resolution treatment.

---

## 3. Results

Backbones: **Qwen2.5-Math-1.5B, -7B, -14B** (Instruct variants),
evaluated on **MATH500, AMC, OlympiadBench, GPQA-Diamond**.

**With adaptive (confidence-gated) inference:**

| Model | Setting | Accuracy | CoT length reduction |
|---|---|---|---|
| 1.5B | τ = 0.999 | MATH500: 73.0% vs. 72.4% baseline | ~36% |
| 7B | τ = 0.9999 | within 0.9–2.2 pts of baseline across benchmarks | ~30–34% |

**Without adaptive fallback (fixed 2-token superposition on every
step):**

| Model | Length reduction | Accuracy cost |
|---|---|---|
| 1.5B | 47–54% | −14.7 to −21.3 points |
| 7B | 48–53% | −5.6 to −12.1 points |

Two things stand out here. First, the **adaptive mechanism is doing
most of the work** — pure fixed-rate compression is aggressive but
expensive in accuracy, while confidence-gated compression recovers
near-parity accuracy at a more modest (but still substantial)
compression rate. Second, there is a clear **scale-tolerance effect**:
the 7B model degrades less than the 1.5B model under equally aggressive
fixed compression, suggesting larger models have more redundant
capacity to absorb the loss of per-token discrete supervision.

---

## 4. Why Speculative Decoding Isn't the Right Comparison

The paper is explicit that speculative decoding is not a fair
baseline: speculative decoding targets **latency under low hardware
utilization** (using spare compute to verify guesses in parallel), not
a reduction in the **total FLOPs** spent on reasoning — which is
SuperThoughts' actual target. A method judged by tokens/sec under
speculative decoding could look identical to SuperThoughts in latency
while doing strictly more total computation.

---

## 5. Relationship to Prior Work

The MTP module design builds on prior multi-token-prediction work
(Gloeckle et al., 2024, Meta; Liu et al., 2024). Relative to
continuous-latent reasoning methods:

| Method | Supervision during latent phase | Chain length tested |
|---|---|---|
| Coconut | Dropped once in latent mode | Short (~20–60 tokens) |
| CODI | Dropped once in latent mode | Short |
| CoLaR | Dropped once in latent mode | Short |
| **SuperThoughts** | **Preserved throughout (discrete supervision)** | **Long, full benchmark-length CoT** |

Since Qwen2.5 models lack native multi-token-prediction heads, the
authors train the MTP module from scratch; they note that models with
native MTP support (Qwen3-Next, MiMo) would likely align more easily
and could simplify training.

---

## 6. Why It Matters

SuperThoughts is a useful data point in the broader "compress
reasoning without losing discrete supervision" agenda: rather than
betting everything on a fully continuous latent reasoning space (which
has repeatedly struggled to scale past short chains), it keeps one
foot in discrete-token land while still getting a real 2× decode-step
reduction on the fast path. The confidence gate is the load-bearing
piece — it is what turns an aggressive-but-fragile compression scheme
into one that holds up at benchmark-length reasoning chains.

---

## 7. Limitations Worth Knowing

- **Requires training an MTP module from scratch** for backbones
  without native multi-token-prediction support, adding training
  complexity absent in models that ship with MTP already.
- **Non-adaptive (fixed-rate) compression is fragile**, especially at
  smaller scale — the adaptive mechanism is not optional for
  maintaining accuracy.
- **Throughput-doubling is a per-step decode claim**, not an
  independently reported wall-clock tokens/sec measurement in the
  sources reviewed here — real-world speedup will depend on how the
  confidence-gated fallback rate behaves across different task
  distributions.
- **Scale-tolerance trend is only shown up to 14B parameters**; whether
  even larger models could tolerate 3+ tokens in superposition is
  speculated but not tested.

---

## 8. The Takeaway for a First Reader

If you remember three things:

1. **SuperThoughts compresses pairs of consecutive CoT tokens into a
   single latent**, decoding two tokens per forward step via a Main
   module (primary) and a lightweight MTP module (secondary) — while
   keeping discrete token supervision throughout training, unlike
   prior continuous-latent reasoning methods.
2. A **confidence-gated adaptive mechanism** falls back to standard
   single-token decoding on hard steps, which is what makes the method
   hold up on long reasoning chains rather than degrading sharply.
3. **~20–30% CoT length reduction with near-parity accuracy** on
   Qwen2.5-Math models across MATH500, AMC, OlympiadBench, and
   GPQA-Diamond — with larger models tolerating more aggressive
   compression than smaller ones.

---

## References

- Xiong, Z., Garg, S., Yu, M., Shrivastava, V., Zhao, H., Kyrillidis,
  A., & Papailiopoulos, D. (2026). *SuperThoughts: Reasoning Tokens in
  Superposition.* ICML 2026.
  [arXiv:2606.13862](https://arxiv.org/abs/2606.13862).
- Related on this site:
  [SOL]({% link _posts/2026-07-19-sol.md %})
  — token-level efficiency policies via GRPO, a complementary axis of
  inference-time compute reduction;
  [Reasoning Cache]({% link _posts/2026-07-19-reasoning-cache.md %})
  — another approach to controlling the cost of long reasoning traces.

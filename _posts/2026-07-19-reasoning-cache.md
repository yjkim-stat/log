---
layout: post
title: "Reasoning Cache — Continual Improvement Over Long Horizons via Short-Horizon RL"
date: 2026-07-19 11:00:00 +0900
description: >
  Paper review of Reasoning Cache / RC (Wu, Qu et al., ICLR 2026, CMU)
  — an iterative decode-and-summarize algorithm that lets LLMs trained
  on short token budgets extrapolate to reasoning horizons more than
  10× longer at test time. A summary replay buffer trains the model to
  condition effectively on intermediate states. RCT-4B (trained at 16K
  tokens) reaches ~70% on HMMT-Nov-2025 and surpasses Qwen3-30B-A3B
  on IMO-AnswerBench at 256K-token test budget.
tags: [reasoning, rl, test-time-scaling, long-horizon, math]
categories: paper-review
toc:
  sidebar: left
related_posts: true
---

**Paper.** Ian Y.H. Wu, Yuxiao Qu, Amrith Setlur, Aviral Kumar.
*Reasoning Cache: Continual Improvement Over Long Horizons via
Short-Horizon RL.* ICLR 2026. Carnegie Mellon University.
[[arXiv]](https://arxiv.org/abs/2602.03773) ·
[[OpenReview]](https://openreview.net/forum?id=DROMQyqM52) ·
[[code]](https://github.com/IanYHWu/rc)

---

## 0. The Picture in One Paragraph

Hard math and science olympiad problems require far more
inference-time computation than models are trained on. Simply
giving a model a larger token budget at test time does not help —
the model was never trained to exploit that extra budget, and its
learned distribution does not cover ultra-long traces.
**Reasoning Cache (RC)** proposes an iterative decoding algorithm
that sidesteps this distributional shift: instead of generating
one long trace, the model runs *multiple short turns*, summarizing
each turn's work into a compact "reasoning cache" before beginning
the next. The full trace from the previous turn is discarded; only
the summary carries forward. A model trained at a 16K-token budget
can thus reason across 8 × 16K = 128K tokens of effective compute
without ever seeing a context longer than 16K during either training
or inference. A **summary replay buffer** in Stage II of training
teaches the model to leverage mid-trajectory summaries, enabling
monotonic performance improvement as the test-time budget grows.
**RCT-4B** reaches ~70% on HMMT-Nov-2025 at a 512K-token budget
(up from ~40% at 16K), and surpasses Qwen3-30B-A3B-Instruct on
IMO-AnswerBench at a 256K-token test budget.

---

## 1. The Problem — Extrapolation Beyond the Training Budget

RL-trained reasoning models (DeepSeek-R1, QwQ, and their
derivatives) are trained at a fixed token budget — typically
16K–32K tokens per problem. At test time, the hardest problems
need much more compute. The naive fix — just give the model a
longer context — fails:

| Approach | Why it fails |
|---|---|
| Longer context window | Model's learned policy does not exploit the extra tokens; it was never trained to self-correct across a long trace |
| More sampling (best-of-N) | Averages over independent draws; individual quality ceiling unchanged |
| Chain-of-thought length scaling | Distributional shift: very long traces are OOD; model degrades |

What is needed is a way to train at a short budget yet *extrapolate*
to arbitrarily long budgets — the model should be able to keep
improving as more compute is provided, with no plateau.

The paper calls this **extrapolation**: the ability to improve over
reasoning horizons more than one order of magnitude longer than
those seen during training.

---

## 2. The Key Insight — LLMs Can Summarize Better Than They Can Continue

The paper identifies an **asymmetry** in LLM capabilities:

> A model given a *summary* of a failed attempt at a problem can
> produce a better next attempt more easily than it can generate a
> full solution from scratch.

Summarization is an inherently easier task than continuation:
the model condenses what happened into a high-level account and
then starts fresh with that context. This asymmetry is what makes
the RC loop work — each turn is short and in-distribution (≤16K
tokens), but each turn is better than the last because it
conditions on the compacted outcome of all prior turns.

---

## 3. The RC-Decoding Algorithm

RC replaces standard autoregressive decoding with an iterative
loop:

```
Turn 1:  problem → [generate trace, ≤16K tokens] → summarize → cache₁
Turn 2:  problem + cache₁ → [generate trace, ≤16K tokens] → summarize → cache₂
Turn 3:  problem + cache₂ → [generate trace, ≤16K tokens] → summarize → cache₃
...
Turn K:  problem + cache_{K-1} → [generate trace, ≤16K tokens] → answer
```

At each turn:
1. **Generation step.** The model generates a reasoning trace
   conditioned on the original problem plus the current summary
   (empty on turn 1). The trace is capped at the training token
   budget (16K).
2. **Summarization step.** The model summarizes the full trace
   into a compact textual reasoning cache.
3. **Discard.** The full trace is thrown away; only the summary
   carries forward.

Repeat for K iterations. The effective token budget is K × 16K,
but the model's context at each step is bounded to problem +
summary — always in-distribution relative to training.

Main hyperparameters in the experiments: K = 8 turns,
N_summ = 2 summarization steps, per-turn budget = 16K tokens.

---

## 4. Training — Two Stages

### Stage I: Base RC Training

- Train the model (from Qwen3-4B-Instruct or Qwen3-30B-A3B-Instruct)
  to generate and summarize on Turn 1 (no prior summary in the input).
- Reward: outcome-based RL via **GRPO** (Group Relative Policy
  Optimization) — correct/incorrect final answer.
- Dataset: ~5,700 problems subsampled from **AceReason-Math**.
- Result: substantial improvement over the base model at moderate
  test budgets, but limited multi-turn extrapolation without Stage II.

### Stage II: Replay Buffer Training

Stage I teaches Turn 1 well but does not teach the model to
leverage intermediate summaries (Turns 2–K). Stage II activates
a **summary replay buffer** to cover these mid-trajectory states:

1. During Stage I rollouts, store `(problem, summary)` pairs
   encountered at each RC turn in a replay buffer.
2. At each Stage II training step: sample a `(problem, summary)`
   pair from the buffer — a mid-trajectory starting point.
3. Run RC-decoding for T_train = 3 steps from that starting point.
4. Write the new summaries generated back into the buffer,
   replacing older summaries for the same problem.
5. Hard problem augmentation: inject difficult problems from the
   **DAPO** dataset to maintain curriculum difficulty.

The replay buffer is what enables extrapolation: it exposes the
model to the full range of mid-trajectory states it will encounter
during long-horizon test-time RC-decoding, not just Turn 1.

**Ablation:** Stage II without the replay buffer yields only ~4%
improvement at 192K tokens. With the buffer, the improvement grows
to ~9.4% — and continues growing monotonically with budget.

---

## 5. Results

### 5.1 HMMT-Nov-2025 (Math Olympiad)

| Model / Setting | Token budget | Accuracy |
|---|---|---|
| Base Qwen3-4B | 16K | ~40% |
| RCT-4B (Stage I + II) | 512K (via RC) | **~70%** |

+30 percentage points by scaling test-time compute 32× — all
from a 4B model trained at 16K tokens.

### 5.2 IMO-AnswerBench

| Model | Token budget | Accuracy |
|---|---|---|
| Base Qwen3-4B | 16K | ~34% |
| RCT-4B | 256K (via RC) | **~50%** |
| Qwen3-30B-A3B-Instruct | (native) | lower than RCT-4B at 256K |

**RCT-4B surpasses Qwen3-30B-A3B-Instruct** (a 7.5× larger model)
at the 256K-token test budget.

### 5.3 Monotonic scaling

| Model | Budget 16K | Budget 192K | Gain |
|---|---|---|---|
| RCT-4B | baseline | +17% | monotonic |
| RCT-30B | baseline | +12% | monotonic |

No plateau visible in the measured range. Both models improve
continuously as test-time budget grows — the defining property
of extrapolation.

### 5.4 Stage II replay buffer ablation

| Method | Accuracy at 16K | Accuracy at 192K | Δ |
|---|---|---|---|
| Stage I only | — | — | baseline |
| Stage II without buffer | +2.7% | +4% | small |
| Stage II with buffer | +2.7% | +9.4% | grows with budget |

The replay buffer matters more as the budget grows — exactly
the pattern expected if it is enabling multi-turn coordination.

---

## 6. Relationship to Test-Time Scaling

RC fits into the broader test-time compute literature as a
specific instantiation of **iterative self-improvement**:

| Method | How it uses extra compute |
|---|---|
| Best-of-N | Independent samples; no learning across attempts |
| Self-Consistency | Majority vote over samples; same ceiling |
| MCTS-based (e.g., MCTD) | Tree search over continuations; structured branching |
| **RC** | Iterative summarize-and-restart; bounded context per turn |

RC's distinctive property: it trains the model *to use the loop*,
not just to be used in a loop. The replay buffer ensures the model
has learned to leverage mid-trajectory summaries, so the quality
of each turn increases with the number of prior turns.

---

## 7. Why It Matters

Three reasons:

1. **Training at short horizons, deploying at long horizons.**
   The ability to extrapolate test-time compute by more than 10×
   beyond the training horizon — without any architectural change,
   just an iterative decoding loop — is a practically important
   capability. Most deployed reasoning models are trained at fixed
   budgets; RC is a recipe for exploiting more inference compute
   without retraining.
2. **Replay buffer as the key engineering insight.** The Stage I
   → Stage II split, and specifically the need for a replay buffer
   to cover mid-trajectory states, is not obvious. Stage II
   without the buffer provides only marginal gains. This is a
   concrete implementation lesson for anyone trying to train
   multi-turn iterative reasoners.
3. **Summarization as a computational primitive.** RC's approach
   identifies summarization as an *enabling primitive* for
   long-horizon reasoning — not an approximation or a
   context-saving trick, but the core mechanism that lets short
   training generalize to long deployment.

---

## 8. Limitations Worth Knowing

- **Training horizon ceiling.** Reasoning traces during training
  are capped at 16K tokens per turn. The model learns to use
  summaries to reason across turns but is never trained to
  generate intra-turn traces longer than 16K.
- **Summarization quality dependency.** A summary that omits
  a critical insight or introduces an error propagates that
  error into all future turns with no mechanism for recovery.
- **Error accumulation.** Unlike a single long trace (where
  later tokens can in principle revise earlier reasoning), an
  error baked into a summary is a persistent bias in all
  subsequent turns.
- **Domain scope.** Experiments focus on mathematics and science
  olympiad problems. Generalization to code, multi-step agent
  tasks, or open-ended reasoning is not evaluated.
- **Inference latency.** K = 8 sequential model calls per problem
  increases wall-clock latency. Production deployment would need
  optimizations not addressed in the paper.

---

## 9. The Takeaway for a First Reader

If you remember three things:

1. **RC trains at 16K tokens but deploys at any multiple of 16K**
   via an iterative loop: generate a trace → summarize it into a
   compact cache → discard the trace → start the next turn
   conditioned on the cache. The model's context per turn is always
   ≤16K and always in-distribution.
2. **A summary replay buffer (Stage II)** stores `(problem, summary)`
   mid-trajectory pairs and resamples them during RL training,
   teaching the model to leverage intermediate states — not just
   Turn 1. Without the buffer, multi-turn extrapolation fails to
   emerge.
3. **RCT-4B** (trained at 16K tokens) reaches **~70% on HMMT-Nov-2025**
   at a 512K test budget (+30 pp over the base model), and
   **surpasses Qwen3-30B-A3B-Instruct on IMO-AnswerBench** at 256K
   tokens — showing monotonic improvement across the full measured
   budget range.

---

## References

- Wu, I.Y.H., Qu, Y., Setlur, A., & Kumar, A. (2026).
  *Reasoning Cache: Continual Improvement Over Long Horizons
  via Short-Horizon RL.* ICLR 2026.
  [arXiv:2602.03773](https://arxiv.org/abs/2602.03773).
- Code: <https://github.com/IanYHWu/rc>
- Related on this site:
  [MCTD review]({% link _posts/2026-07-02-mctd.md %})
  — inference-time scaling via MCTS inside the diffusion process,
  a complementary approach to test-time compute scaling;
  [RLHF → RULER trend note]({% link _posts/2026-05-25-rlhf-to-ruler.md %})
  — the RL post-training context this work sits within.

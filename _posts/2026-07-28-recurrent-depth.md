---
layout: post
title: "Huginn — Scaling Test-Time Compute via Recurrent Depth in Latent Space"
date: 2026-07-28 06:10:00 +0900
description: >
  Paper review of "Scaling up Test-Time Compute with Latent Reasoning:
  A Recurrent Depth Approach" (Geiping, McLeish, Jain, Kirchenbauer,
  Singh, Bartoldson, Kailkhura, Bhatele, Goldstein — ELLIS Institute
  Tubingen, University of Maryland, Lawrence Livermore National
  Laboratory) — the "Huginn" model, which scales test-time compute by
  looping a shared recurrent block in latent space instead of emitting
  explicit chain-of-thought tokens, trained at 3.5B parameters / 800B
  tokens on the Frontier supercomputer.
tags: [latent-reasoning, test-time-scaling, architecture, reasoning, llm]
categories: paper-review
toc:
  sidebar: left
related_posts: true
---

**Paper.** Jonas Geiping, Sean McLeish, Neel Jain, John Kirchenbauer,
Siddharth Singh, Brian R. Bartoldson, Bhavya Kailkhura, Abhinav
Bhatele, Tom Goldstein.
*Scaling up Test-Time Compute with Latent Reasoning: A Recurrent Depth
Approach.*
arXiv preprint, February 2025. ELLIS Institute Tübingen / Max-Planck
Institute for Intelligent Systems / Tübingen AI Center, University of
Maryland, Lawrence Livermore National Laboratory.
[[arXiv]](https://arxiv.org/abs/2502.05171)

---

## 0. The Picture in One Paragraph

Most test-time scaling scales compute by writing more chain-of-thought
tokens — which requires specialized reasoning training data, burns
context-window budget, and represents "thinking" as literal words.
This paper (whose resulting model is nicknamed **Huginn**, after one
of Odin's two ravens — "Thought," as opposed to its counterpart
Muninn, "Memory") takes a different route: a transformer split into a
**prelude**, a small **recurrent core** (a handful of shared blocks),
and a **coda**, where the recurrent core is looped R times *in latent
space* before the coda decodes an output. No specialized chain-of-
thought data is needed, small context windows suffice (the "thinking"
never leaves latent space to consume context tokens), and R is a
free knob at test time — more recurrence, more effective compute,
without changing the parameter count. Trained at **3.5B parameters and
800B tokens** on Oak Ridge's Frontier supercomputer, the model shows
reasoning-benchmark gains that scale with recurrence depth up to a
compute load roughly equivalent to a **50B-parameter** non-recurrent
model — though it still trails the strongest explicit-CoT models on
some benchmarks like GSM8K.

---

## 1. The Problem — Chain-of-Thought Is One Specific Way to Spend Test-Time Compute

Explicit CoT test-time scaling works, but it has costs baked into its
form: it needs reasoning-annotated or RL-shaped training data, it
consumes the context window with literal reasoning tokens, and it
assumes that useful intermediate computation can always be represented
in words. The paper's premise is that at least some of that
computation is better represented as continued **latent-space
refinement** of a hidden state, unconstrained by having to be
periodically projected back into vocabulary space and re-embedded.

---

## 2. Architecture — Prelude, Recurrent Core, Coda

The model is split into three parts:

- **Prelude** (~2 transformer blocks): embeds the input into an
  initial latent state.
- **Recurrent core** (~4 transformer blocks): a small shared bank of
  blocks that is looped R times, injected with a randomly initialized
  latent state at the start of recurrence and iteratively refining it.
- **Coda** (~2 blocks + unembedding head): reads the final latent
  state after R iterations and decodes the output.

Because the recurrent core is weight-tied across iterations, looping
it R times adds no parameters — only compute. R is chosen at test
time (commonly 16–128 in the paper's experiments), giving **per-token
adaptive compute**: easy tokens can be decoded after few iterations,
hard tokens can keep looping. The design is also compatible with
KV-cache sharing across recurrence steps and with speculative
decoding, since the recurrent core's weight-tying means cached
key/value projections from earlier iterations remain valid.

### Training the recurrence

Rather than fixing R during training, the depth per training step is
sampled from a **Poisson-log-normal distribution with mean 32**, and
backpropagation is **truncated to the last 8 recurrence passes** to
bound memory and compute. This lets the model see a wide range of
recursion depths during training without paying the full
back-through-time cost of the deepest ones.

---

## 3. Training Scale

The model was trained at **3.5B parameters** on **800B tokens**,
using **21 training segments of 4,096 AMD MI250X GPUs** on the Oak
Ridge Frontier supercomputer — a substantial systems effort in its own
right, reflecting the difficulty of training recurrent-depth models at
scale (variable computational graphs per step complicate standard
data/pipeline parallelism).

---

## 4. Results

Reasoning-benchmark performance improves — "sometimes dramatically" —
as recurrence depth R increases, up to a compute budget roughly
equivalent to a **50B-parameter** non-recurrent model, at which point
gains from further recurrence plateau or mildly decline (tested out to
R ≈ 128–256). On GSM8K without explicit CoT prompting, accuracy rises
with more recurrence steps before plateauing — outperforming
similarly-sized non-recurrent open baselines like Pythia-6.9B and
Pythia-12B on ARC and GSM8K. Gains are most pronounced on math- and
logic-heavy benchmarks (GSM8K, ARC-Challenge); they are flatter on
knowledge-heavy benchmarks like MMLU and HellaSwag, where extra latent
recurrence has less to work with. Notably, plain recurrent-depth
scaling still **lags the strongest explicit-CoT models** on GSM8K,
even as it substantially closes the gap against non-recurrent
baselines of similar parameter count.

---

## 5. Emergent Latent-Space Behavior

Qualitative analysis of the recurrent core's trajectories through
latent space finds recognizable geometric patterns tied to task type:

- **Orbits** — rotating, cyclic trajectories on tasks with an
  iterative structure.
- **Sliders** — progressive, monotonic movement through latent space
  on counting-like tasks.
- **Fixed-point convergence** — the latent state settling into a
  stable point once further recurrence stops changing the answer.

The authors also report a genuine **self-correction** behavior: on
some examples, the model detects and revises an earlier wrong
intermediate value during later recurrence steps — computation that
happens entirely in latent space, never surfacing as a visible,
correctable chain-of-thought token.

---

## 6. Why It Matters

This paper is a foundational reference point for the whole
looped/recurrent-depth transformer line — both LoopFormer's
elastic-depth training and Think-at-Hard's selective per-token
iteration (both reviewed alongside this post) explicitly position
themselves relative to it. Huginn establishes that (a) latent-space
recurrence is trainable at real scale (3.5B/800B tokens) without
specialized reasoning data, and (b) depth genuinely functions as a
test-time compute knob with measurable accuracy gains — the base
result that later work refines by making depth *elastic* (LoopFormer)
or *selective per token* (TaH) rather than a single global recurrence
count.

---

## 7. Limitations Worth Knowing

- **Proof-of-concept scale.** The authors describe this as an early
  demonstration; 3.5B parameters is modest by frontier-model
  standards, and it is unclear how the recurrence-depth scaling curve
  behaves at 10× or 100× the parameter count.
- **Gains plateau and can mildly degrade at very high recurrence
  counts** (roughly R ≈ 128–256 in the reported experiments) — more
  test-time compute is not unboundedly beneficial.
- **Still trails top explicit-CoT models on some benchmarks**
  (e.g., GSM8K), meaning recurrent-depth latent reasoning is not (yet)
  a strict replacement for chain-of-thought, more a complementary or
  partially substitutable mechanism.
- **Wall-clock cost of recurrence versus explicit CoT tokens** is not
  directly compared in a hardware-normalized way in the sources
  reviewed here; the "compute-equivalent to 50B parameters" framing is
  a FLOPs-style comparison, not a latency benchmark.

---

## 8. The Takeaway for a First Reader

If you remember three things:

1. **Test-time compute doesn't have to be spent on explicit CoT
   tokens** — this paper scales it instead through R iterations of a
   shared recurrent block in latent space, with no specialized
   reasoning training data required.
2. The architecture is **prelude → recurrent core (looped R times) →
   coda**, trained by sampling R per step from a Poisson-log-normal
   distribution (mean 32) with truncated backprop through the last 8
   passes.
3. At **3.5B parameters / 800B tokens**, the model (nicknamed
   **Huginn**) shows accuracy gains scaling with recurrence up to a
   ~50B-parameter compute-equivalent, with emergent latent-space
   self-correction — while still trailing top explicit-CoT models on
   some benchmarks.

---

## References

- Geiping, J., McLeish, S., Jain, N., Kirchenbauer, J., Singh, S.,
  Bartoldson, B.R., Kailkhura, B., Bhatele, A., & Goldstein, T. (2025).
  *Scaling up Test-Time Compute with Latent Reasoning: A Recurrent
  Depth Approach.*
  [arXiv:2502.05171](https://arxiv.org/abs/2502.05171).
- Related on this site:
  [LoopFormer]({% link _posts/2026-07-28-loopformer.md %})
  — trains a single model for elastic, budget-conditioned loop depth,
  directly building on this recurrent-depth foundation;
  [Think-at-Hard]({% link _posts/2026-07-28-think-at-hard.md %})
  — pushes the same latent-iteration idea down to per-token
  selectivity rather than a single global recurrence count.

---
layout: post
title: "Landscape of Thoughts — Visualizing Where LLM Reasoning Actually Goes"
date: 2026-08-07 09:00:00 +0900
description: >
  Paper review of Landscape of Thoughts (Zhou, Zhu, Li, Galkin, Feng,
  Koyejo, Tang, Han — ICLR 2026, TMLR Group @ HKBU / Stanford / Mila /
  Université de Montréal / HEC Montréal / Intel AI Lab) — the first
  visualization tool that projects every intermediate reasoning state
  of an LLM's chain-of-thought trajectory into a 2D "landscape" relative
  to the answer choices, turning perplexity-based distance into a
  picture of how (and whether) reasoning converges toward the right
  answer.
tags: [reasoning, interpretability, chain-of-thought, visualization, llm, evaluation]
categories: paper-review
toc:
  sidebar: left
related_posts: true
---

**Paper.** Zhanke Zhou, Zhaocheng Zhu, Xuan Li, Mikhail Galkin, Xiao
Feng, Sanmi Koyejo, Jian Tang, Bo Han.
*Landscape of Thoughts: Visualizing the Reasoning Process of Large
Language Models.*
ICLR 2026. TMLR Group @ Hong Kong Baptist University, Stanford
University, Mila – Québec AI Institute, Université de Montréal,
HEC Montréal, Intel AI Lab.
[[arXiv]](https://arxiv.org/abs/2503.22165) ·
[[project page]](https://landscape-of-thoughts.github.io/) ·
[[code]](https://github.com/tmlr-group/landscape-of-thoughts)

---

This is the first post in what will be an ongoing series of reasoning
paper reviews on this blog, tagged
[`reasoning`]({{ "reasoning" | slugify | prepend: '/blog/tag/' | relative_url }}).
It felt right to start with a paper that isn't proposing a new
reasoning *method*, but a way to actually **look at** what existing
reasoning methods are doing — a lens that every later review in this
series can implicitly use.

---

## 0. The Picture in One Paragraph

Chain-of-thought reasoning is evaluated almost exclusively by its
final-answer accuracy — a single scalar that says nothing about *how*
the model got there, or where it went wrong along the way. **Landscape
of Thoughts (LoT)** opens up that black box with a simple but
under-explored idea: take every intermediate reasoning state in a
trajectory (each partial chain-of-thought so far), compute how close
that state is to each candidate answer using **perplexity as a
distance metric** (asking the same LLM how likely it is to complete
the trajectory with each answer choice), and project all these
per-state distance vectors into **2D via t-SNE**. The result is a
literal landscape: the answer choices sit as fixed points, and a
reasoning trajectory becomes a path that either converges toward the
correct answer, drifts, or wanders. This single visualization —
backed by three quantitative metrics (perplexity, uncertainty,
consistency) — turns out to reliably distinguish strong models from
weak ones, correct trajectories from incorrect ones, and can even be
repurposed into a lightweight verifier that improves test-time
scaling.

---

## 1. The Problem — Accuracy Tells You Nothing About the Path

Reasoning research has produced a large zoo of prompting and search
strategies — chain-of-thought, self-consistency, tree-of-thoughts,
MCTS-guided search — and the standard way to compare them is a
leaderboard of final-answer accuracy on math and QA benchmarks. This
is a reasonable summary statistic, but it collapses everything
interesting about *how* a model reasons into one number:

- Two models can reach the same accuracy while one converges
  confidently and the other gets there by essentially guessing late.
- A single model can look identical on aggregate accuracy across two
  datasets while behaving completely differently at the trajectory
  level.
- Failure analysis usually means reading individual chain-of-thought
  transcripts by hand — it doesn't scale, and it's hard to turn into a
  quantitative signal.

LoT's premise is that the *sequence of intermediate states* a
reasoning trajectory passes through is itself informative, and that
information is currently thrown away by accuracy-only evaluation.

---

## 2. The Method — From Text States to a 2D Landscape

### 2.1 Sampling and segmenting trajectories

For a given multi-choice question, a reasoning method (chain-of-thought
being the base case, though the tool is designed to work with its
derivatives too) is used to sample a reasoning trajectory from an LLM.
That trajectory is segmented into individual **thoughts** — the
intermediate reasoning steps — so a single trajectory becomes a
sequence of partial states: state after thought 1, state after
thought 1+2, and so on up to the final answer.

### 2.2 Turning each state into a distance vector

The core trick: for each intermediate state, LoT asks **the same LLM
that generated the trajectory** to estimate how well that partial
reasoning path would lead into each candidate answer, using
**perplexity** as the distance metric. This produces, for every state,
a vector of distances — one number per answer choice — without
needing a separate reward model or verifier. A state's distance vector
is a self-consistent measure of "given what's been reasoned so far,
how compatible is this with each possible answer."

### 2.3 Projecting into a landscape

These distance vectors across all states in all sampled trajectories
are projected into **2D with t-SNE**, with the answer choices treated
as fixed anchor points. A single trajectory then reads as a literal
path across the plot: it starts somewhere ambiguous and either
converges toward the correct-answer anchor, converges toward a wrong
one, or fails to converge at all — visually distinguishable at a
glance, and aggregatable across many trajectories to compare models,
datasets, or reasoning methods side by side.

### 2.4 Three quantitative metrics

The visualization is backed by three scalar metrics that don't require
looking at a plot at all:

- **Perplexity** — the raw distance-to-answer-choice signal itself,
  usable to estimate how compatible a state is with each candidate
  across different thought lengths.
- **Uncertainty** — how confident the model is about its prediction at
  intermediate steps (not just at the final answer).
- **Consistency** — roughly, whether the model already "knows" the
  eventual answer early in the trajectory, rather than only converging
  on it very late (or not at all).

---

## 3. Findings

### 3.1 Scale changes the shape of the landscape, not just the accuracy

Using the Llama-3.1 family swept from 1B to 70B parameters with CoT
prompting on AQuA, the landscape visibly changes shape with scale:
larger models converge faster toward the correct-answer region, with
higher density of late-trajectory states clustered near the correct
answer — consistent with, and visually explaining, their higher
accuracy. Quantitatively, larger models show **higher consistency,
lower uncertainty, and lower perplexity** — the three metrics move
together with scale, not just the final accuracy number.

### 3.2 The landscape separates correct from incorrect trajectories

Across models and datasets, LoT reliably distinguishes correct
trajectories from incorrect ones — correct trajectories show a
different convergence pattern (faster, more monotonic movement toward
the right answer) than incorrect ones. This holds not just as a
qualitative visual pattern but as a quantifiable signal: **convergence
speed itself predicts whether a trajectory will land on the correct
answer**, distinct between success and failure cases.

### 3.3 It surfaces failure modes accuracy alone can't show

Because the tool exposes per-state uncertainty and consistency, it
can flag reasoning patterns that look fine in the final answer but are
fragile underneath — low consistency (the model doesn't settle on an
answer until very late, essentially getting there by chance) and high
uncertainty (the model never becomes confident even when it ends up
correct). These are exactly the patterns that aggregate accuracy
metrics cannot distinguish from confident, well-grounded correct
reasoning.

### 3.4 From visualization to verifier

The same per-state distance/convergence features that make the
landscape interpretable are also useful as a **standalone signal**:
the authors adapt LoT into a lightweight verifier that scores
trajectory correctness using these features, without training a
separate reward model from scratch. Used to select among multiple
sampled trajectories, this verifier **improves reasoning accuracy and
strengthens the test-time scaling effect** — i.e., the same
interpretability tool doubles as a practical component for
best-of-N-style inference.

---

## 4. Why It Matters

Two things make this a useful paper to have read before diving into
method-proposing reasoning papers:

1. **It reframes "does this method work" as "how does this method's
   trajectory move through state space."** Every reasoning method
   review that follows in this series — search-based, budget-aware,
   latent-iteration, whatever the mechanism — is implicitly making a
   claim about *how it shapes the trajectory's path toward the
   answer*. LoT gives a vocabulary (convergence speed, consistency,
   uncertainty) for that claim that doesn't require a new benchmark
   number.
2. **The verifier repurposing is the sharper practical point.**
   A visualization tool that also functions as a free,
   training-free-ish scoring signal for trajectory selection is a
   different kind of contribution than "we made a prettier plot" — it
   suggests that the geometry of intermediate reasoning states carries
   real, exploitable signal about correctness, not just a
   post-hoc explanation of it.

---

## 5. Limitations Worth Knowing

- **Requires multi-choice structure.** The distance-to-answer-choice
  feature construction depends on having a fixed, enumerable set of
  candidate answers — it isn't a direct fit for open-ended generation
  tasks where there's no small set of anchor points to project against.
- **Perplexity as a distance metric is model-relative.** The distance
  vector for a given state is computed using the same LLM that
  produced the trajectory, so landscapes are not directly comparable
  across models without care — a "close to the correct answer"
  reading from one model's self-assessment isn't necessarily
  calibrated the same way as another's.
- **t-SNE distortion.** Like any t-SNE projection, the 2D layout is a
  nonlinear compression of a higher-dimensional space; visual distances
  and cluster shapes can mislead if read too literally, and the
  quantitative metrics (perplexity, uncertainty, consistency) are the
  more trustworthy artifacts than the plot geometry itself.
- **Verifier gains are reported at a general level** in the sources
  used for this review — exact accuracy deltas and which datasets/
  reasoning methods the verifier was tested against were not fully
  confirmed against the primary PDF, so treat the "boosts accuracy and
  test-time scaling" claim as directionally correct pending a closer
  read of the paper's tables.

---

## 6. The Takeaway for a First Reader

If you remember three things:

1. **LoT visualizes reasoning trajectories, not just final accuracy**
   — every intermediate state in a chain-of-thought is turned into a
   distance-to-each-answer-choice vector (via perplexity) and
   projected into 2D with t-SNE, so a trajectory becomes a literal
   path toward (or away from) the correct answer.
2. **Three metrics — perplexity, uncertainty, consistency — move
   together with model scale and correctness.** Bigger models converge
   faster and more confidently; correct trajectories are
   quantitatively distinguishable from incorrect ones by how they
   move, not just where they end up.
3. **The same features double as a lightweight, training-free-ish
   verifier** for selecting among sampled trajectories — turning an
   interpretability tool into a practical test-time-scaling component.

---

## References

- Zhou, Z., Zhu, Z., Li, X., Galkin, M., Feng, X., Koyejo, S., Tang,
  J., & Han, B. (2026). *Landscape of Thoughts: Visualizing the
  Reasoning Process of Large Language Models.* ICLR 2026.
  [arXiv:2503.22165](https://arxiv.org/abs/2503.22165) ·
  [project page](https://landscape-of-thoughts.github.io/) ·
  [code](https://github.com/tmlr-group/landscape-of-thoughts).

---
layout: post
title: "The Expressive Power of Transformers with Chain of Thought"
date: 2026-08-09 09:00:00 +0900
description: >
  Paper review of Merrill & Sabharwal (ICLR 2024, NYU / Allen Institute
  for AI) — a circuit-complexity characterization of what decoder-only
  transformers can compute once they are allowed to generate
  intermediate chain-of-thought tokens, and exactly how many of those
  tokens it takes to buy how much extra power.
tags: [reasoning, chain-of-thought, theory, expressivity, transformers, llm]
categories: paper-review
toc:
  sidebar: left
related_posts: true
---

**Paper.** William Merrill, Ashish Sabharwal.
*The Expressive Power of Transformers with Chain of Thought.*
ICLR 2024 (Poster). New York University, Allen Institute for AI.
[[arXiv]](https://arxiv.org/abs/2310.07923) ·
[[OpenReview]](https://openreview.net/forum?id=NjNGlPh8Wh)

---

Second post in the [`reasoning`]({{ "reasoning" | slugify | prepend: '/blog/tag/' | relative_url }})
series, and a deliberate change of register from the first. [Landscape
of Thoughts]({% post_url 2026-08-07-landscape-of-thoughts %}) asked
*where does reasoning actually go* by looking at trajectories
empirically. This paper asks the question one level down: **what can a
transformer that emits chain-of-thought tokens compute at all**, as a
matter of circuit complexity — independent of any particular model,
dataset, or training run. It's the theoretical floor and ceiling that
every empirical CoT result in this series will implicitly sit inside.

## 0. The Picture in One Paragraph

Without chain of thought, a transformer that has to answer immediately
after reading its input is provably stuck inside **TC0** — the class of
problems solvable by constant-depth, polynomial-size threshold
circuits. That's a real ceiling: TC0 can't even reliably do things like
check whether two nodes in a directed graph are connected, because
that kind of problem is inherently *serial* — each step depends on the
result of the previous one, and a constant-depth circuit has no way to
unroll an unbounded number of sequential steps. Chain of thought
changes the computational model itself: each generated token is fed
back in and attended to, so the transformer effectively gets one more
"layer" of computation per CoT token. The paper's central result is
that this isn't a vague intuition — it's an exact trade curve. Zero or
constant CoT steps: still TC0. Logarithmically many steps: only a
marginal gain. Linearly many steps: the transformer can recognize every
regular language. Polynomially many steps: the transformer recognizes
**exactly P**, no more and no less. Length of chain of thought isn't a
knob you turn for style — it's the resource that literally determines
which complexity class you're computing in.

## 1. The Problem — Why "TC0" Is the Right Question to Ask

Merrill and Sabharwal's own prior work had already pinned down what a
transformer *without* CoT can do: under standard assumptions
(log-precision, poly-size), a transformer's forward pass is simulable
by a uniform TC0 circuit family. TC0 is a narrow class — it's below
NL (nondeterministic log-space, the class containing graph
reachability) and believed to be strictly below P. So there's a
concrete, well-known list of problems a no-CoT transformer *cannot*
solve at scale, no matter how it's trained: directed graph
connectivity, simulating a finite-state automaton, evaluating a
boolean formula with unbounded nesting. These aren't exotic edge
cases — they're the textbook shape of "the next step depends on
everything before it," which is exactly the shape of a lot of
multi-step reasoning.

Empirically, chain-of-thought prompting was already known to help
transformers with exactly this flavor of task. The open theoretical
question was whether that's a real expansion of computational power or
just a more favorable way of asking the same TC0-bounded model for an
answer. The paper's answer is unambiguous: it's a real expansion, and
its size is governed almost entirely by how many CoT tokens you allow.

## 2. The Result — A Trade Curve, Not a Threshold

The framing is circuit complexity throughout: a "decoding step" is one
generated CoT token, fed back into the transformer before the next
step. The paper walks the number of allowed decoding steps up from
constant to polynomial and tracks what complexity class falls out at
each point.

### 2.1 Constant and logarithmic steps — barely more than nothing

A constant number of CoT steps doesn't change the class at all — still
TC0, since a constant number of extra "layers" doesn't help a
constant-depth circuit escape its own definition. Allowing
O(log n) steps (n = input length) buys only a modest amount of extra
power, still well short of what's needed for NL-complete problems like
graph connectivity. This is the paper's most counter-intuitive finding
for anyone who assumes "more CoT tokens = proportionally more
reasoning power": logarithmically many tokens is a rounding error,
computationally speaking.

### 2.2 Linear steps — all regular languages

Once the number of decoding steps scales linearly with input length
(and with a mild architectural condition the paper calls "projected
pre-norm"), the transformer can recognize **every regular language** —
i.e., simulate an arbitrary finite-state automaton step by step,
writing its running state into the CoT and reading it back. This is
the point where the earlier example — "can a transformer track state
across an unboundedly long sequence" — flips from impossible to
solvable.

### 2.3 Polynomial steps — exactly P

With polynomially many decoding steps and a further generalization of
pre-norm, the paper proves the transformer's expressive power is
**exactly P** — not an upper bound, an exact characterization, in both
directions: anything in P can be computed by some CoT transformer with
polynomially many steps, and no CoT transformer with a polynomial step
budget can compute more than P. The general form of the argument is
almost a simulation lemma: any problem decidable in time t(n) can be
solved by a transformer using roughly t(n) chain-of-thought tokens,
because each CoT step can simulate one step of a general computation.
The paper calls this the first exact characterization of a transformer
variant in terms of a standard, unconditional complexity class — most
prior transformer-expressivity results are one-sided (upper bounds
only).

### 2.4 The example problems that carry the intuition

Three recurring examples do the work of making this concrete: **s-t
reachability** in a directed graph (NL-complete, needs more than
log-length CoT), **simulating a finite-state automaton** (the
canonical regular-language task that linear-length CoT unlocks), and
**composing a sequence of permutations** (an inherently serial,
non-parallelizable task used to illustrate why constant-depth circuits
struggle with "chained" operations even when each individual operation
is trivial).

## 3. Why It Matters

For a blog whose reasoning series so far has been almost entirely
architectural — looped transformers, latent iteration, adaptive
compute — this paper is the piece that explains *why* those
architectures are chasing the effect they're chasing. Think-at-Hard,
LoopFormer, and Huginn all spend extra computation per token to
approximate what unrolled, serial reasoning buys you; this paper says
precisely what that serial computation is worth, in classical
complexity terms, when it's spent as explicit CoT tokens instead of
implicit latent iterations. It also gives a principled reading of a
purely empirical observation that recurs across the CoT literature:
short reasoning traces plateau on genuinely serial problems not
because the model "isn't trying hard enough," but because a chain of
thought that's too short is, provably, still stuck below the
complexity class the problem lives in. If a task is NL-hard-flavored,
no amount of prompting cleverness inside a logarithmic-length CoT
budget will get a TC0-bounded model there — the length of the chain
has to actually grow with the problem.

## 4. Limitations Worth Knowing

This is a **computability result, not a learnability result.** The
paper shows what a transformer of appropriate size and weights *could*
compute given enough CoT steps — it says nothing about whether
gradient descent on realistic pretraining data would ever find those
weights, or how much CoT-annotated data that would take. The
polynomial-steps-equal-P result also depends on a specific
architectural assumption ("generalized pre-norm") rather than holding
for every transformer variant unconditionally, so the exact
correspondence is a statement about a well-defined model family, not a
claim that any transformer with enough CoT tokens automatically reaches
P. And "chain of thought" here means *any* generated intermediate
token sequence used as scratch space — the paper is not making claims
about whether *human-readable, semantically coherent* reasoning traces
(the kind actually produced by CoT-prompted LLMs) achieve these bounds
in practice, only that some sequence of tokens of the given length
could.

## 5. The Takeaway for a First Reader

Chain-of-thought length is not a stylistic hyperparameter — it is,
provably, the resource that determines which complexity class a
transformer's answer can come from. Constant or logarithmic CoT barely
moves the needle past TC0; it takes linear CoT to reach all regular
languages and polynomial CoT to reach all of P. Every subsequent paper
in this series that tries to make reasoning cheaper by shortening or
compressing the chain of thought is implicitly trading against this
curve, and it's worth having Merrill & Sabharwal's exact version of the
trade-off in mind as a reference point for what "shorter" is actually
giving up.

## References

- Merrill, W., & Sabharwal, A. (2024). *The Expressive Power of
  Transformers with Chain of Thought.* ICLR 2024.
  [arXiv:2310.07923](https://arxiv.org/abs/2310.07923)
- Related, complementary ICLR 2024 result: Li, Z., et al. *Chain of
  Thought Empowers Transformers to Solve Inherently Serial Problems.*
  [arXiv:2402.12875](https://arxiv.org/abs/2402.12875)
- This blog: [Landscape of Thoughts]({% post_url 2026-08-07-landscape-of-thoughts %}) —
  the empirical companion to this post's theory, first in the
  `reasoning` series.
- This blog: [Think-at-Hard]({% post_url 2026-07-28-think-at-hard %}),
  [LoopFormer]({% post_url 2026-07-28-loopformer %}),
  [recurrent-depth / Huginn]({% post_url 2026-07-28-recurrent-depth %}) —
  architectures that spend extra per-token computation to approximate
  the effect this paper attributes to explicit CoT length.

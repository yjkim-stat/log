---
layout: post
title: "Learning Fast and Slow: A New Recipe for Adapting LLMs Without Forgetting"
date: 2026-05-18 17:30:00 +0900
description: >
  Trend note on the GEPA team's "Learning, Fast and Slow" — Fast-Slow Training
  (FST) interleaves slow RL weight updates with fast in-context prompt
  evolution via GEPA, hitting RL's ceiling with up to 3x fewer samples and
  staying close enough to the base model to keep learning new tasks afterward.
tags: [llm, rl, prompt-optimization, continual-learning, gepa]
categories: trends
toc:
  sidebar: left
related_posts: true
---

**Source.** GEPA team blog post,
[*"Learning, Fast and Slow: Towards LLMs That Adapt Continually"*](https://gepa-ai.github.io/gepa/blog/2026/05/11/learning-fast-and-slow/),
May 11, 2026. Backed by Tiwari, Sareen, Agrawal, Gonzalez, Zaharia, Keutzer,
Dhillon, Agarwal, & Khatri, *Learning, Fast and Slow: Towards LLMs That
Adapt Continually*, [arXiv:2605.12484](https://arxiv.org/abs/2605.12484).

---

## Why This Matters

For two years the implicit assumption in LLM post-training has been "if
you want the model to do a new task well, **update the weights**" — RLHF,
RLAIF, GRPO, DPO, RLOO, REINFORCE++, etc. The cost of that assumption is
becoming visible:

- **Sample inefficiency.** Each new task burns through tens of thousands
  of rollouts.
- **Catastrophic forgetting.** A weights-update on task A erodes
  performance on B.
- **Loss of plasticity.** After a few RL phases the model becomes
  *unable to learn* further tasks — the policy has drifted so far from
  the base distribution that exploration collapses.

"Learning, Fast and Slow" (LFS) proposes that we have been using only
half of the available adaptation surface. There is a second, much
cheaper handle — the **textual context** — that we routinely throw away
between training runs. Combining the two surfaces yields **Fast-Slow
Training (FST)**.

---

## The Two-Speed Adaptation Taxonomy

The framing borrows from Kahneman's *Thinking, Fast and Slow* but
inverts the labels deliberately:

| Component | Role | Cost | Persistence |
|---|---|---|---|
| **Slow — model weights $\theta$** | Long-lived policy / skills | Expensive (gradients, RL rollouts) | Survives across sessions |
| **Fast — prompt / instructions / in-context state** | Task-level shaping | Cheap (text edits) | Per-session, can be swapped freely |

Most modern LLM post-training touches only the slow side. In-context
learning touches only the fast side. Neither alone reaches the joint
optimum the paper demonstrates is reachable.

---

## What GEPA Is (Quick Refresher)

[GEPA](https://arxiv.org/abs/2507.19457) (Genetic-Pareto, ICLR 2026
Oral) is a *reflective prompt evolution* optimizer. Instead of taking a
scalar reward and updating weights, GEPA:

1. **Reads the full execution trace** of a candidate prompt — error
   messages, profiling info, reasoning logs, intermediate outputs.
2. Uses an LLM to **diagnose** the failure in natural language.
3. **Mutates** the prompt with a targeted edit that addresses the
   diagnosis.
4. Maintains a **Pareto frontier** over multiple objectives via an
   evolutionary loop.

The original GEPA paper showed it can **outperform GRPO by ~6% on
average (up to 20%)** using **up to 35× fewer rollouts** across six
tasks. The reason is information density: GRPO compresses an entire
trajectory into one scalar; GEPA reads the trajectory itself.

In FST, GEPA plays the role of the fast learner.

---

## Fast-Slow Training (FST)

The algorithm is intentionally simple — most of the contribution is the
fact that *the simple thing works*.

```
repeat:
  # Fast phase — frozen weights, optimize the context
  c* = GEPA(theta_t, task)            # reflective prompt evolution

  # Slow phase — frozen context, update the weights
  theta_{t+1} = RL_update(theta_t, task, context=c*)
```

Two consequences fall out:

1. **The RL phase starts from a better operating point.** GEPA has
   already pushed the *fast* state toward task-competence, so the slow
   update doesn't need to invent that competence from scratch — it can
   *consolidate* it.
2. **The fast state acts as a regularizer.** Because much of the
   task-specific shaping is held in the prompt, the weight update needed
   to lock in the behavior is smaller, so $\theta$ drifts less from the
   base model.

---

## The Headline Numbers

The paper reports these on a reasoning-task suite plus a continual-learning
benchmark:

| Metric | Result |
|---|---|
| Sample efficiency vs. RL-only | **Up to 3× fewer samples** to reach RL's peak |
| Final reward | **Above** either RL-only or GEPA-only |
| KL divergence from base model at convergence | **Up to 70% lower** than RL-only |
| 3-stage continual learning | **FST near-peak on all three stages; RL completely stalls on stage 2, only partially recovers on stage 3** |
| Plasticity (can the trained checkpoint still learn a new task?) | RL-trained checkpoint barely adapts; FST-trained checkpoint roughly matches the base model |

The continual-learning result is the most striking. The RL-only model
acquires task 1 well, then becomes a brick. The FST model keeps learning.

---

## Why It Works (the Story Beneath the Numbers)

Three ingredients reinforce each other:

1. **GEPA extracts more information per sample than RL.** Reading a
   trace is denser than reading a reward, so the fast loop converges in
   a handful of rollouts.
2. **The fast loop pre-shapes the task.** When the slow loop fires, it
   sees gradients that already point in a useful direction.
3. **The slow loop barely has to move.** Because the fast state carries
   the task-specific shaping, only the *transferable* piece of behavior
   gets baked into weights — which is exactly what you want when the
   next task arrives.

The chain explains the plasticity finding without invoking any new
mechanism: a model that hasn't drifted far from the base distribution
still has room to be moved.

---

## Where This Sits in the Trend Landscape

A few converging threads make FST timely:

- **Reflective prompt optimization is maturing.** GEPA, DSPy's
  optimizers, and the growing skill ecosystem (e.g. the
  [`grill-me`]({% link _explorations/2026-05-18-grill-me-skills.md %})
  family, the
  [research-paper-writing skill]({% link _explorations/2026-05-18-research-paper-writing-skills.md %}))
  show that *prompts are now a first-class optimization target*.
- **The cost of RL is being noticed.** Meta-Harness
  ([review]({% link _posts/2026-05-18-meta-harness.md %})) and FST both
  argue that we have been spending compute on the wrong loop. Meta-Harness
  optimizes the *harness*; FST optimizes the *prompt-conditioned RL
  process*. They are complementary moves in the same direction.
- **Continual learning is back on the agenda.** Plasticity loss has
  been a research curiosity in classic RL for a decade; FST shows that
  it bites in modern LLM post-training too — and that the fast-slow
  decomposition is a credible mitigation.

If you were betting on what 2026's post-training stack will look like,
the rough shape now reads: **fast prompt search + small slow updates +
careful KL control**, rather than the current default of pure RL with
ever-larger batch sizes.

---

## What I'm Watching For

- **Other "fast" choices.** GEPA is one instance of the fast loop;
  retrieval, tool memory, and structured scratchpads are others. The
  FST scaffold doesn't depend on GEPA specifically.
- **Multi-task aggregation.** The reported continual benchmark is a
  3-stage stream. Can FST hold up to dozens of tasks, the way a
  production system would need?
- **Reward-shaping by the fast loop.** A speculative but natural
  extension: let GEPA also propose changes to the *reward signal* the
  slow loop optimizes against. If the fast loop can read traces, it
  can also read what the reward should have been.

---

## TL;DR

LLM adaptation has been treated as a single problem ("update the
weights"). FST argues it is actually two: a **slow** update of
parameters and a **fast** update of context, with GEPA filling the
fast role. Interleaving the two reaches RL's ceiling with **~3× fewer
samples**, climbs **higher**, and — most importantly — leaves the
model close enough to its base distribution to **keep learning** on
new tasks. That last property is what makes this the most interesting
post-training paper of the month.

---

## References

- GEPA team blog (May 11, 2026):
  *[Learning, Fast and Slow: Towards LLMs That Adapt Continually](https://gepa-ai.github.io/gepa/blog/2026/05/11/learning-fast-and-slow/).*
- Paper: Tiwari et al., *Learning, Fast and Slow: Towards LLMs That
  Adapt Continually*, [arXiv:2605.12484](https://arxiv.org/abs/2605.12484).
- Background: Agrawal et al., *GEPA: Reflective Prompt Evolution Can
  Outperform Reinforcement Learning*,
  [arXiv:2507.19457](https://arxiv.org/abs/2507.19457) (ICLR 2026 Oral).
- Related on this site:
  [Meta-Harness review]({% link _posts/2026-05-18-meta-harness.md %}),
  [grill-me skills]({% link _explorations/2026-05-18-grill-me-skills.md %}),
  [research-paper-writing skill]({% link _explorations/2026-05-18-research-paper-writing-skills.md %}).

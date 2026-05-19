---
layout: post
title: "ECHO — Terminal Agents Learn World Models for Free"
date: 2026-05-18 21:30:00 +0900
description: >
  Trend note on ECHO (Shrivastava & Papailiopoulos, 2026) — a one-line
  change to GRPO for CLI agents that stops masking out terminal-output
  tokens, adds an environment cross-entropy loss alongside the policy
  loss, and produces measurable world-model behavior with essentially
  zero extra compute.
tags: [llm, rl, agents, world-models, grpo]
categories: trends
toc:
  sidebar: left
related_posts: true
---

**Source.** Dimitris Papailiopoulos thread on X (May 18, 2026)
introducing *ECHO: Terminal Agents Learn World Models for Free*,
co-authored with Vaishnavi Shrivastava. Microsoft Research AI
Frontiers. Implementation on top of SkyRL.

---

## Why This Matters

Standard agent RL — GRPO and its cousins — trains on the agent's
**actions** and discards the **environment's response**. The terminal
output is in the rollout, in the context, the forward pass has
already computed logits over it, and then the trainer simply *masks
it out of the loss*. ECHO's observation is that this is leaving free
supervision on the table: every command an agent runs produces a
terminal response that is **ground truth about how the action
changed the world**, and the loss can be extended to use it at
essentially zero cost.

The headline finding is simple enough to fit on a slide:

> Add a length-normalized cross-entropy loss on environment-output
> tokens to the standard GRPO loss on action tokens. Same rollouts,
> same forward pass, just a different mask over the logits. Every
> CLI-agent eval improves; **TerminalBench-2.0 pass@1 nearly doubles**
> at both 8B (2.7 → 5.2) and 14B (5.2 → 10.8).

The reason this is worth treating as a trend rather than a single
trick is that the same one-line change opens three doors at once:
faster RL, weaker dependence on expert SFT, and — most surprisingly —
self-improvement with no verifier reward at all.

---

## The Mechanism in One Equation

A CLI rollout already interleaves two token streams: the agent's
**action** tokens $A$ and the terminal's **observation** tokens $O$.
GRPO applies a policy-gradient loss only on $A$. ECHO keeps that and
adds a supervised cross-entropy term on $O$:

$$
\mathcal{L}_{\text{ECHO}} \;=\; \mathcal{L}_{\text{GRPO}}(A) \;+\; \lambda \cdot \mathcal{L}_{\text{env}}(O).
$$

Three design choices in the loss matter:

- **On-policy.** $O$ is the terminal's response to the *current*
  policy's actions. Better policies explore new states; new states
  yield fresh observation supervision. Better feedback prediction
  improves the policy's action prior. Loop.
- **Length-normalized.** Without normalization, long observations
  (e.g. file listings, stack traces) would dominate.
- **Train on real terminal output, not harness warnings.** Warnings
  are easy to memorize and uninformative; filenames, tracebacks, and
  error messages are where the signal lives.

The cost analysis is the part that makes this go viral. The expensive
part of backprop is the matmuls through attention + MLP at every
sequence position — those run *regardless* of which positions
contribute to the loss. The logits at every observation token are
already computed for GRPO. The action mask and the observation mask
just gather different subsets for different loss terms. **No extra
rollouts, no extra forward pass, no teacher model.**

---

## Headline Numbers

Evaluated on Qwen3-8B, OpenThinker-Agent-v1-SFT, and Qwen3-14B; same
GRPO recipe, tasks, rollout/turn budget, and step count for both
arms. The only knob is whether the env loss is on.

| Benchmark | Model | GRPO | **ECHO** |
|---|---|---|---|
| TerminalBench-2.0 pass@1 | Qwen3-8B | 2.7 | **5.2** |
| TerminalBench-2.0 pass@1 | Qwen3-14B | 5.2 | **10.8** |
| TerminalBench-Lite | Qwen3-8B | — | **matches GRPO 500-step result 280 steps earlier (~2.3× faster)** |
| Held-out env cross-entropy | all | barely drops | **drops sharply** |

The qualitative pattern across every panel: ECHO curves separate
early from GRPO and stay above; environment-token cross-entropy on
*trajectories the model didn't generate* (drawn from a stronger
Qwen3-32B teacher) drops with ECHO and barely moves with plain GRPO.
That second number is the falsifiable form of the world-model claim:
ECHO produces policies that are measurably better at compressing
terminal dynamics they did not produce.

---

## Three Surprising Results

### 1. ECHO substitutes for most of expert SFT

The standard recipe for terminal-agent training is
*expert-demonstration SFT* (behavior-cloning a stronger teacher) and
*then* GRPO. ECHO from a base Qwen3-8B with **no expert
demonstrations** recovers:

- **104% of the SFT gain on ITD**
- **89% on TerminalBench-Lite**
- **50% on TerminalBench-2.0**

The interpretation the authors offer: a large chunk of what expert
SFT actually teaches is an *interaction prior* — how to read tracebacks,
which commands expose useful state, which outputs are diagnostic —
*not* the expert's strategic choices. ECHO learns that prior directly
from the environment, without ever imitating the expert.

### 2. Sample efficiency from failed trajectories

In the Qwen3-8B setting, fewer than 15% of on-policy rollouts succeed.
Under standard GRPO, the remaining 85% contribute almost no learning
signal. Under ECHO, **failed trajectories still contain file listings,
stack traces, grep outputs, and other consequences of the agent's
actions** — all of which are now training targets. The
2.3× speed-up on TerminalBench-Lite comes from this conversion of
failed rollouts into supervised data.

### 3. Verifier-free self-improvement

The most provocative experiment: drop the GRPO term entirely. Train
**only** on $\mathcal{L}_{\text{env}}(O)$. No reward, no verifier —
just "act, observe, update by predicting what came back." From the
strongest Qwen3-8B+ECHO checkpoint, 100 additional env-only steps
improved performance on held-out tasks:

| Held-out set | Δ pass-rate |
|---|---|
| val100 (near-distribution) | **+3.8 pp** |
| ITD (clean-rollout filter) | **+5.2 pp** |
| PyTerm (Python-heavy OOD) | **+10.0 pp** |

The pattern is conditional but real: env-only training helps when
the policy is already a "decent explorer" *and* when observations are
informative (Python tasks give dense `code → traceback → fix`
feedback; broader terminal tasks reveal state more indirectly). The
striking part isn't that the agent self-improves perfectly — it's
that **it self-improves at all, with no reward signal whatsoever**.

---

## Where This Sits in the Trend Landscape

ECHO is the in-the-RL-loop, terminal-flavored instance of a broader
shift toward **action-consequence prediction as auxiliary
supervision**. Adjacent threads converging on the same idea:

- *Agent Learning via Early Experience* — uses action-consequence
  signal as a *pre-RL* stage.
- *VAGEN* — adds a world-modeling reward for vision-language agents.
- *RWML* — pretrains on next-state prediction.
- *CWM* — mid-trains a code model on observation-action trajectories.

ECHO's positioning relative to these is *online* and *embarrassingly
cheap*. Same rollout, same forward pass, a one-line loss change. The
authors' bet: some form of environment-token prediction will be
**standard in agent RL trainers by the end of 2026**.

This connects directly to several recent posts on this site:

- [Meta-Harness review]({% link _posts/2026-05-18-meta-harness.md %}) —
  argues the *harness* is the right optimization target while the
  model is held fixed. ECHO is the *RL-objective* version of the same
  argument: hold the model and the rollouts fixed, optimize what we
  learn from them.
- [SOAR review]({% link _posts/2026-05-18-soar.md %}) — RL stalls when
  initial success is sparse. ECHO addresses the same scarcity from the
  other side: when 85% of rollouts fail, *the failure traces become
  the data*.
- [Fast-Slow Training trend note]({% link _posts/2026-05-18-learning-fast-and-slow.md %}) —
  "extract more signal per rollout." ECHO is the cleanest example of
  that principle I've seen this year.

---

## What I'm Watching

- **Beyond the terminal.** The authors flag the obvious extensions:
  browser agents, multi-tool systems, long-horizon coding agents,
  user-facing assistants. Anywhere an agent acts and the world
  responds in tokens, ECHO's recipe applies in principle.
- **Better targets than raw output.** Raw stdout is free but noisy.
  Compact summaries, state diffs, or task-relevant projections may
  give a sharper learning signal. This is the cleanest engineering
  improvement on the horizon.
- **Trajectory filtering.** PyTerm's +10pp came partly from filtering
  to clean tool-call trajectories. A principled trajectory-quality
  estimator is the missing piece between "self-improves sometimes"
  and "self-improves reliably."
- **The λ knob.** Joint-loss balance is the one thing that can go
  wrong: if λ is too large, the policy starts optimizing for
  *predictable* outputs rather than task progress (model-the-world
  Goodharting). The paper notes this; the right schedule is open.
- **Combination with FrontierSmith-style data.** Pair
  [open-ended continuous-score environments]({% link _posts/2026-05-18-frontiersmith.md %})
  with ECHO's free auxiliary supervision and you get a fairly
  complete recipe for verifier-rich *and* observation-rich agent
  training.

---

## TL;DR

ECHO stops masking out terminal-output tokens during GRPO and adds a
length-normalized cross-entropy loss on them. Same rollout, same
forward pass, basically no extra compute. The result is **stronger
RL** (TerminalBench-2.0 pass@1 ~doubles at 8B and 14B), **2.3×
faster training**, **substantial substitution for expert-SFT
pretraining**, and — without any verifier — **measurable
self-improvement from environment interaction alone**. The deeper
claim is structural: **agent rollouts contain far more supervision
than the final reward, and the cheapest move in modern agent RL is
to stop throwing it away**.

---

## References

- Shrivastava, V., & Papailiopoulos, D. (2026). *ECHO: Terminal
  Agents Learn World Models for Free.* Microsoft Research AI
  Frontiers. Code on top of
  [SkyRL](https://github.com/NovaSky-AI/SkyRL).
- Announcement thread:
  [@DimitrisPapail on X, May 18, 2026](https://x.com/DimitrisPapail/status/2056368948870811746).
- Related on this site:
  [Meta-Harness review]({% link _posts/2026-05-18-meta-harness.md %}),
  [SOAR review]({% link _posts/2026-05-18-soar.md %}),
  [FrontierSmith review]({% link _posts/2026-05-18-frontiersmith.md %}),
  [Fast-Slow Training trend note]({% link _posts/2026-05-18-learning-fast-and-slow.md %}).

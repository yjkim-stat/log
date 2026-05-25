---
layout: post
title: "From RLHF to RULER: How the Reward Signal for RL Agents Evolved"
date: 2026-05-25 14:00:00 +0900
description: >
  Trend note tracing the RL reward-signal problem from RLHF's four-model
  PPO stack (2022) through RLVR+GRPO's two-model breakthrough (2025) to
  RULER's LLM-as-judge scoring for non-verifiable agent tasks (2026),
  with a walkthrough of OpenPipe's ART framework.
tags: [llm, rl, grpo, agents, reward-signal]
categories: trends
toc:
  sidebar: left
related_posts: true
---

**Source.** Avi Chawla, *"How Top AI Labs Are Building RL Agents
in 2026,"*
[Daily Dose of Data Science](https://blog.dailydoseofds.com/p/how-top-ai-labs-are-building-rl-agents),
Apr 28, 2026.
Framework: [OpenPipe ART](https://github.com/OpenPipe/ART) (9K+ stars).

---

## Why This Matters

The optimization algorithm for RL on LLMs has been settled for over
a year — **GRPO** handles it. What hasn't been settled is **where the
reward comes from.** The article traces a clean three-act story:

1. **RLHF (2022):** humans rate outputs → reward model → PPO.
   Works, but expensive and requires four full-size models in memory.
2. **RLVR + GRPO (2025):** environment provides a deterministic
   signal (math checker, code compiler) → no reward model, no
   critic. Two-model setup. Dominant recipe for reasoning.
3. **RULER (2026):** LLM-as-judge scores *relative* rankings across
   a trajectory group → plugs into GRPO as-is. Extends RL to
   non-verifiable agent tasks (RAG, support, summarization) without
   writing custom reward functions.

Each transition solves the same problem — *where does the reward
number come from?* — at one higher level of generality.

---

## Act 1: RLHF and the Four-Model Stack

OpenAI's InstructGPT (2022) established the pipeline ChatGPT was
built on:

1. Humans rank model outputs (expensive, one-time).
2. A **reward model** is trained on those rankings.
3. **PPO** uses the reward model to fine-tune the LLM.

PPO requires **four** full-size models in memory simultaneously:

| Model | Role |
|---|---|
| **Policy** | The LLM being trained |
| **Reference** | Frozen copy for KL regularization |
| **Reward model** | Human-preference approximator |
| **Critic (value model)** | Predicts the expected reward per prompt to produce the *advantage* signal |

The critic answers: "was this reward good or bad *relative to what
we'd normally expect for this prompt?*" A raw reward of 0.7 means
nothing in isolation — on a factual question where most outputs
score 0.9 it's below average; on a hard open-ended question where
most score 0.4 it's excellent. The critic learns that baseline.

For a 7B model, that's ~28B parameters in memory at once.

---

## Act 2: RLVR + GRPO — the DeepSeek Breakthrough

In January 2025, DeepSeek R1 replaced both the reward model and the
critic with two simpler ideas:

**RLVR** (Reinforcement Learning with Verifiable Rewards): for math,
check if the answer matches the known solution; for code, run the
compiler. Binary 1/0. No humans, no learned reward model.

**GRPO** (Group Relative Policy Optimization): generate $N$ responses
per prompt (typically 16), normalize rewards *within each group*.
If 4/16 got the answer right, those 4 get positive advantage; the
other 12 get negative. The group itself is the baseline — **no
critic model needed.**

The four-model PPO stack collapses to **two** (policy + reference),
or in practice close to one (reference folded into checkpoint).

The result: **DeepSeek R1-Zero** went from 15.6% to 77.9% on
AIME 2024 with just GRPO + verifiable rewards and zero supervised
fine-tuning. With majority voting, 86.7% — matching OpenAI's o1.
The model developed self-verification, reflection, and chain-of-thought
*on its own*, purely from the binary correct/incorrect signal.

RLVR + GRPO became the dominant recipe for reasoning models through
2025. Every major lab released a reasoning variant using it.

---

## Act 3: The Agent Reward Gap

GRPO is general-purpose — it doesn't care whether the reward comes
from a math verifier, a compiler, a human, or a Python script. It
just needs a number per response and normalizes within each group.

The bottleneck is **where that number comes from** once you leave
the verifiable-task regime:

| Task type | Reward source | Problem |
|---|---|---|
| Math, code, logic | Environment verifier (RLVR) | Solved |
| RAG agent | ? | No single correct answer to string-match |
| Customer support | ? | No compiler to run it through |
| Summarization | ? | Many valid summaries; no deterministic check |

The obvious workaround: **write custom reward functions in Python.**
A RAG reward function might check faithfulness (did it use the
context?), penalize hallucination, reward completeness, and weight
them. This works, but:

- Writing a good one takes **days of iteration**.
- It's **brittle** — change the retrieval pipeline and the reward
  function needs rewriting.
- **Debugging is opaque** — when the agent learns bad behavior,
  is it the reward weights, the training hyperparams, or the data?
- A function that over-weights format and under-weights faithfulness
  produces **beautifully formatted hallucinations**.

This is the primary reason RL has been widely adopted for verifiable
tasks but not for agent workflows.

---

## The Lab Responses

The major labs have been converging on the same gap from different
directions:

- **Anthropic — Constitutional AI.** Write down principles (a
  "constitution"); an AI evaluates its own outputs against those
  principles and generates preference data for RL. A document of
  rules replaced an army of human evaluators.
- **OpenAI — Universal Verifiers.** Extending RL beyond math and
  code into biology, medicine, and general knowledge where answers
  can't be string-matched. Details not public, but the direction is
  general-purpose reward signals across any domain.

---

## RULER — LLM-as-Judge That Maps to GRPO

[RULER](https://github.com/OpenPipe/ART), built into OpenPipe's ART
framework, replaces custom scoring code with a single function call.
It uses an LLM-as-judge to rank multiple trajectories, exploiting the
same property that makes GRPO powerful: **only relative rankings
matter.**

### How it works

1. Generate $N$ trajectories for the same scenario (typically 4–8).
2. Send all $N$ to a judge LLM (o3, o4-mini, or local Qwen3-32B).
3. The judge reads the agent's **system prompt** to understand the
   task, then scores each trajectory 0–1 *relative to the others*.

Two properties make this work:

**Relative scoring is easier than absolute scoring.** LLMs struggle
with calibration on absolute scales but consistently handle "which
of these 4 responses best follows the instructions?" RULER presents
all trajectories together and asks for a ranking.

**GRPO normalizes within each group anyway.** Whether the best
trajectory scored 0.9 or 0.3 in absolute terms doesn't matter —
GRPO computes the mean and standard deviation within the group and
normalizes. RULER's relative rankings map directly onto what GRPO
expects.

### The system prompt *is* the reward function

The key insight: you don't write a reward function — the system
prompt already defines what "good" means. A RAG system prompt that
says *"Answer using ONLY the retrieved context; do not add
information not in the context"* implicitly defines faithfulness,
hallucination, and completeness. The judge applies those criteria
without anyone implementing them in Python.

The article demonstrates this concretely with four RAG trajectories
(faithful, hallucinated, context-ignoring, verbose-but-accurate).
RULER produces scores of 0.98, 0.20, 0.05, and 0.96 — a nuanced
gradient that would take significant engineering to encode in a
rule-based function. And when the system prompt is tightened ("Do
not add information" vs. "Answer accurately"), the judge
automatically tightens its scoring to match.

### Custom rubrics

When the system prompt isn't specific enough:

```python
custom_rubric = """
- Prioritize concise and clear responses
- Penalize emojis or informal language
- Reward responses that cite sources
"""
await ruler_score_group(group, "openai/o3", rubric=custom_rubric)
```

Natural language, not Python. Iterating is fast — change a sentence,
rerun, check the scores — versus editing a reward function where a
misplaced weight silently teaches bad behavior.

### Combining with deterministic verifiers

For tasks that are partially verifiable:

```python
judged_group = await ruler_score_group(group, "openai/o3")
for traj in judged_group.trajectories:
    traj.reward += verify_correctness(traj)  # binary 0/1
```

RULER preserves any rewards assigned during rollout, so you can
layer LLM-judge scoring on top of deterministic verification.

---

## Practical Notes from the Article

- **Judge model choice is a cost-quality tradeoff**, not a hard
  requirement. Qwen3-32B often works; doesn't need o3.
- **4–8 trajectories per group** is the sweet spot. Fewer gives
  the judge too little to compare; more can confuse it.
- **Common-prefix deduplication.** When all trajectories share the
  same system/user messages (usual case), RULER deduplicates
  automatically — significant token savings on long system prompts.
- **Disk caching.** RULER caches judge responses; reruns on the
  same trajectories skip the API. Matters when iterating on the
  system prompt or rubric.

---

## Where This Sits in the Trend Landscape

The article crystallizes a progression that several recent posts on
this site have been touching from different angles:

- [**SOAR**]({% link _posts/2026-05-18-soar.md %}) — RL stalls when
  the model never solves the problem (zero reward). SOAR's fix is to
  change the *problem distribution*. RULER's fix is to change the
  *reward source* — from binary verifier to relative LLM judge.
  Different angles on the same scarcity.
- [**ECHO**]({% link _posts/2026-05-18-echo-terminal-world-models.md %}) —
  "agent rollouts contain more supervision than the final reward."
  ECHO uses env-token prediction as free auxiliary signal; RULER
  uses an LLM judge as a general-purpose signal. Both argue the
  same thing: **stop throwing away information from the rollout.**
- [**Meta-Harness**]({% link _posts/2026-05-18-meta-harness.md %}) —
  optimize the *harness*, not the model. RULER's framing is the
  same: the system prompt is the harness, and RULER's judge scores
  against it. Iterating on the system prompt iterates on the reward
  function.
- [**Fast-Slow Training**]({% link _posts/2026-05-18-learning-fast-and-slow.md %}) —
  the "fast" surface is the prompt; the "slow" surface is the
  weights. RULER makes the fast surface (the system prompt) the
  *evaluation criterion*, not just the conditioning context.

The structural prediction implicit in all of these: **the era of
hand-written reward functions for agent RL is ending.** The system
prompt, a rubric, or a constitution — all natural-language
descriptions of what "good" means — will replace Python scoring
code as the default reward interface.

---

## TL;DR

The bottleneck in applying RL to agents was never the optimization
algorithm (GRPO handles it). It was always the **reward signal**.
Three acts:

1. **RLHF** (2022) — humans → reward model → PPO. Four models in
   memory. Expensive.
2. **RLVR + GRPO** (2025) — environment verifier → binary reward.
   Two models. Dominant for reasoning. But only works when the
   answer is checkable.
3. **RULER** (2026) — LLM judge scores trajectories *relatively*
   within a GRPO group. The system prompt *is* the reward function.
   Works on non-verifiable agent tasks (RAG, support,
   summarization) with **no custom scoring code**. Built into
   OpenPipe's open-source [ART framework](https://github.com/OpenPipe/ART).

---

## References

- Chawla, A. (2026). *How Top AI Labs Are Building RL Agents in
  2026.* [Daily Dose of Data Science](https://blog.dailydoseofds.com/p/how-top-ai-labs-are-building-rl-agents).
- OpenPipe ART framework:
  <https://github.com/OpenPipe/ART>
- Background: InstructGPT / RLHF (Ouyang et al., 2022),
  DeepSeek R1 / GRPO (DeepSeek, 2025),
  Constitutional AI (Anthropic, 2022).
- Related on this site:
  [SOAR review]({% link _posts/2026-05-18-soar.md %}),
  [ECHO trend note]({% link _posts/2026-05-18-echo-terminal-world-models.md %}),
  [Meta-Harness review]({% link _posts/2026-05-18-meta-harness.md %}),
  [Fast-Slow Training trend note]({% link _posts/2026-05-18-learning-fast-and-slow.md %}).

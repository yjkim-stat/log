---
layout: post
title: "Polar — Agentic RL on Any Harness at Scale"
date: 2026-05-25 16:00:00 +0900
description: >
  Paper review of Polar (Xu et al., NVIDIA, 2026) — a rollout framework
  that treats any agent harness as a black box, proxies LLM API calls to
  record token-level interactions, reconstructs token-faithful
  trajectories for RL training, and improves Qwen3.5-4B by up to +22.6
  points on SWE-Bench Verified across four different harnesses.
tags: [llm, rl, agents, infrastructure, llm-serving]
categories: paper-review
toc:
  sidebar: left
related_posts: true
---

**Paper.** Binfeng Xu, Hao Zhang, Shaokun Zhang, Songyang Han,
Mingjie Liu, Jian Hu, Shizhe Diao, Zhenghui Jin, Yunheng Zou,
Michael Demoret, Jan Kautz, Yi Dong.
*Polar: Agentic RL on Any Harness at Scale.*
arXiv:2605.24220, May 2026. NVIDIA.
[[arXiv]](https://arxiv.org/abs/2605.24220) ·
[[code]](https://github.com/NVIDIA-NeMo/ProRL-Agent-Server)

---

## 0. The Picture in One Paragraph

Training a coding agent with RL sounds simple in theory: run the
agent in an environment, collect the trajectory, compute the reward,
update the weights. In practice the **agent harness** — the
scaffolding that manages multi-turn tool calls, context windows,
error recovery, file edits — is a complex piece of software
(Codex CLI, Claude Code, Qwen Code, Pi, Aider, …), and porting it
into an RL environment interface is an engineering project that
**loses training signal along the way** (re-tokenization drift,
dropped metadata, misaligned turn boundaries). Polar sidesteps the
porting problem entirely: it treats the harness as a **black box**,
interposes a proxy between the harness and the inference server,
records every LLM API call at token level, and reconstructs
**token-faithful trajectories** that any RL trainer can consume.
With simple GRPO, this is enough to improve Qwen3.5-4B by **+22.6**
points on SWE-Bench Verified (Codex harness), **+6.2** (Pi),
**+4.8** (Claude Code), and **+0.6** (Qwen Code).

---

## 1. The Problem — Why "Just Run RL" Doesn't Work for Agents

A reasoning RL setup (the DeepSeek R1 / GRPO recipe) has a tight
loop: the model generates a response, a verifier scores it, GRPO
updates. The "environment" is trivial — it's the verifier.

An **agentic** RL setup is different in three structural ways:

1. **Multi-turn, tool-using trajectories.** A single episode is not
   one model call but a *sequence* of calls — read file, edit, run
   tests, read error, edit again — interspersed with tool outputs,
   harness-injected system messages, and context management
   decisions.
2. **The harness is load-bearing.** Whether the agent uses Codex
   CLI, Claude Code, or Pi changes the trajectory format, the tool
   interface, the context window management, and the error-recovery
   logic. These differences are not cosmetic — the same model with
   different harnesses produces dramatically different SWE-Bench
   scores.
3. **Long-running, variable-length episodes.** A single SWE-Bench
   task can take minutes of wall-clock time. Synchronous RL (wait
   for the slowest episode in the batch) wastes GPU time.

The standard approach has been to **rewrite** the harness into an
RL-compatible environment interface. This is expensive, error-prone,
and — critically — introduces **re-tokenization drift**: the tokens
the trainer sees are not exactly the tokens the model produced
during the rollout, because the trajectory was serialized to text
and re-tokenized. Polar's argument is that this entire approach is
backwards.

---

## 2. The Polar Architecture

Polar's design has one key insight and three components that
implement it.

### The insight

> Don't adapt the harness to the trainer. **Proxy the model calls**
> and record the trajectory at the token level. The harness runs
> unmodified; the proxy captures everything the trainer needs.

### 2.1 The API Proxy

Polar interposes a lightweight proxy between the harness process
and the inference server (e.g. SGLang, vLLM). Every LLM API call
the harness makes — chat completions, tool calls, system-message
injections — passes through the proxy, which:

- **Records the full request and response** at token level (token
  IDs, not just text).
- **Timestamps and orders** every interaction for trajectory
  reconstruction.
- **Passes through transparently** — the harness sees a standard
  OpenAI-compatible API endpoint. No code changes needed.

This is what makes Polar harness-agnostic. The harness doesn't know
Polar exists; it just talks to what looks like a normal inference
server.

### 2.2 Rollout Nodes (Gateway)

Each rollout node manages four phases in parallel:

| Phase | What it does |
|---|---|
| **Prewarming** | Spins up the runtime environment (Docker container, repo checkout, dependency install) *before* the model starts generating. Overlaps environment setup with prior episodes' training. |
| **Agent execution** | Runs the harness inside the primed environment. The harness calls the proxy; the proxy calls the inference server. |
| **Trajectory reconstruction** | Converts the recorded API logs into a token-faithful training trajectory — token IDs, turn boundaries, reward assignment points. |
| **Evaluation** | Runs the task's test suite to produce a reward signal (pass/fail on SWE-Bench, or any other verifier). |

The four phases are **pipelined across episodes**: while episode
$N$ is being evaluated, episode $N+1$ is executing, and episode
$N+2$'s environment is prewarming. This is where the compute
utilization gain comes from.

### 2.3 Asynchronous Service Endpoints

Rollout nodes expose **asynchronous service endpoints** that
independent trainers consume. The trainer doesn't wait for a
synchronous batch of episodes — it pulls completed trajectories as
they arrive. This decouples rollout speed from training speed, which
is essential when episode lengths vary by 10× (a simple bug fix vs.
a complex refactor).

```
┌──────────────┐     ┌───────────────────────┐     ┌───────────┐
│ Trainer      │     │ Rollout Node (×N)      │     │ Inference │
│ (GRPO, any)  │◄────│ prewarm → exec → recon │────►│ Server    │
│              │     │           ↕ proxy       │     │ (SGLang)  │
└──────────────┘     └───────────────────────┘     └───────────┘
   pulls async           harness runs here            serves model
```

The three-way decoupling — trainer, rollout, inference — means each
can scale independently and can run different software stacks.

---

## 3. Token-Faithful Trajectory Reconstruction

This is the technical contribution that makes the proxy approach
work for RL training, not just logging.

The problem: RL trainers need **token IDs** aligned exactly with
the model's vocabulary, with correct **loss masks** (which tokens
to update on) and **turn boundaries** (where the model's
contribution starts and ends). A naive approach — serialize the
trajectory to text, re-tokenize at training time — introduces
**re-tokenization drift**: different tokenization of the same text
can shift token boundaries, change sequence lengths, and corrupt
the training signal.

Polar avoids this by recording token IDs directly from the proxy.
The reconstruction step assembles these recorded IDs into the
format the trainer expects, preserving:

- **Exact token identity** — no re-tokenization.
- **Turn boundaries** — which tokens are model output (trainable)
  vs. harness injection / tool output (masked).
- **Reward assignment** — which turn(s) receive the episode reward.

The paper ablates different trajectory reconstruction strategies,
showing that token-faithful reconstruction outperforms text-based
re-tokenization.

---

## 4. Headline Results

All numbers are Qwen3.5-4B trained with simple GRPO, evaluated on
**SWE-Bench Verified**:

| Harness | Δ SWE-Bench Verified (pp) |
|---|---|
| **Codex** | **+22.6** |
| **Pi** | **+6.2** |
| **Claude Code** | **+4.8** |
| **Qwen Code** | **+0.6** |

The variance across harnesses is itself a finding. The same model,
the same RL algorithm, the same reward signal — but four very
different improvement magnitudes depending on which harness wraps
the model. This validates what the
[Meta-Harness review]({% link _posts/2026-05-18-meta-harness.md %})
on this site argued: **the harness is not plumbing; it's a
first-order variable in agent performance.**

The Codex harness result (+22.6) is the most dramatic, suggesting
that Codex's scaffolding provides a particularly learnable
trajectory structure for GRPO to optimize over. The Qwen Code
result (+0.6) is near-flat, suggesting that harness may already
be near-optimal for this model size, or that its trajectory
structure gives GRPO less signal to work with.

---

## 5. Why This Matters

Three reasons:

1. **It eliminates the porting tax.** Every previous agentic RL
   system required rewriting the harness into a custom RL
   environment. Polar's proxy-based approach means any
   OpenAI-API-compatible harness works out of the box. This moves
   the cost of agentic RL from "months of engineering per harness"
   to "point Polar at the harness and run."
2. **It makes the harness itself a variable.** Because Polar is
   harness-agnostic, you can now **compare** the RL-training
   effectiveness of different harnesses on the same model with the
   same algorithm. The SWE-Bench numbers across Codex / Claude Code
   / Qwen Code / Pi are the first controlled experiment of this
   kind I've seen. This opens a new axis of optimization: *which
   harness produces the most learnable trajectories?*
3. **The async architecture solves the utilization problem.** Agent
   RL episodes are long and variable. Synchronous batching wastes
   GPU time waiting for the slowest episode. Polar's decoupled
   rollout → trainer pipeline keeps GPUs busy because the trainer
   pulls trajectories as they arrive, not in synchronized batches.

---

## 6. Where This Sits in the Trend Landscape

Polar is the **infrastructure** answer to a question several recent
papers on this site have been asking from the *algorithm* side:

- [**ECHO**]({% link _posts/2026-05-18-echo-terminal-world-models.md %}) —
  "stop throwing away information from the rollout." Polar makes
  sure none of it is lost in the first place — token-faithful
  reconstruction preserves every signal the rollout produced.
- [**Meta-Harness**]({% link _posts/2026-05-18-meta-harness.md %}) —
  "optimize the harness, not the model." Polar operationalizes
  this: by making the harness a swappable black box, it turns
  harness comparison into a tractable experiment.
- [**RLHF → RULER trend note**]({% link _posts/2026-05-25-rlhf-to-ruler.md %}) —
  "the bottleneck is the reward signal." Polar is orthogonal: it
  assumes you *have* a reward signal (SWE-Bench's test suite) and
  solves the *infrastructure* problem of getting the trajectory
  from the harness to the trainer without corruption.
- [**FrontierSmith**]({% link _posts/2026-05-18-frontiersmith.md %}) —
  "manufacture training environments at scale." Polar is the
  *execution* layer for those environments: once you have the tasks,
  you need a framework that can run arbitrary harnesses on them
  at RL scale.

The structural bet: **agentic RL will standardize on proxy-based
rollout infrastructure** (Polar, ProRL-Agent, SkyRL-Agent) rather
than bespoke per-harness RL environments. The proxy is the right
abstraction because the harness is evolving faster than any RL
environment spec can track.

---

## 7. Limitations Worth Knowing

- **Single model scale shown.** Qwen3.5-4B is small. Whether the
  per-harness variance pattern (Codex +22.6, Qwen Code +0.6) holds
  at 14B / 32B / 72B is the obvious next experiment.
- **SWE-Bench only.** Agent tasks beyond code (web browsing,
  multi-tool workflows, research) aren't evaluated. The
  architecture is task-agnostic, but the results aren't yet.
- **Reward is still binary pass/fail.** SWE-Bench's test suite
  provides a clean signal, but the framework doesn't address what
  happens when you need continuous or subjective rewards (the
  RULER problem from the earlier trend note).
- **Proxy overhead.** Interposing a proxy adds latency per API call.
  The paper argues this is negligible relative to episode length,
  but for short-episode tasks the cost-benefit may flip.
- **Harness as black box has limits.** Polar can't optimize *inside*
  the harness — it can only optimize the model's responses *given*
  the harness. If the harness itself makes bad decisions (e.g.
  truncates context poorly), Polar can't fix that.

---

## 8. The Takeaway for a First Reader

If you remember three things:

1. **Porting agent harnesses into RL environments is the bottleneck
   for agentic RL**, and it introduces re-tokenization drift that
   corrupts the training signal.
2. **Polar treats the harness as a black box**, interposes a proxy
   that records token-level API calls, and reconstructs
   **token-faithful trajectories** — no harness modifications, no
   drift.
3. With simple GRPO on Qwen3.5-4B, Polar produces **+22.6 pp on
   SWE-Bench Verified** (Codex harness), demonstrating that the
   infrastructure problem was real and that solving it unlocks
   substantial agent improvement.

That's the arc: black-box proxy → token-faithful reconstruction →
async rollout pipeline → harness-agnostic RL at scale.

---

## References

- Xu, B., Zhang, H., Zhang, S., Han, S., Liu, M., Hu, J., Diao, S.,
  Jin, Z., Zou, Y., Demoret, M., Kautz, J., & Dong, Y. (2026).
  *Polar: Agentic RL on Any Harness at Scale.*
  [arXiv:2605.24220](https://arxiv.org/abs/2605.24220).
- Code: <https://github.com/NVIDIA-NeMo/ProRL-Agent-Server>
- Related: ProRL Agent
  ([arXiv:2603.18815](https://arxiv.org/abs/2603.18815)),
  SkyRL-Agent
  ([arXiv:2511.16108](https://arxiv.org/abs/2511.16108)),
  NeMo Gym (<https://github.com/NVIDIA-NeMo/Gym>).
- Related on this site:
  [Meta-Harness review]({% link _posts/2026-05-18-meta-harness.md %}),
  [ECHO trend note]({% link _posts/2026-05-18-echo-terminal-world-models.md %}),
  [RLHF → RULER trend note]({% link _posts/2026-05-25-rlhf-to-ruler.md %}),
  [FrontierSmith review]({% link _posts/2026-05-18-frontiersmith.md %}).

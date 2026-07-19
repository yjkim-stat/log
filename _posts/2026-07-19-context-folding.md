---
layout: post
title: "Context-Folding — Scaling Long-Horizon LLM Agents via Branch-and-Fold"
date: 2026-07-19 10:00:00 +0900
description: >
  Paper review of Context-Folding (Sun, Lu et al., ICML 2026, ByteDance
  Seed / CMU) — a framework that lets LLM agents actively manage their
  own working context by branching into sub-trajectories and folding them
  away once complete. FoldGRPO adds token-level process rewards (Unfolded
  Token Penalty + Out-of-Scope Penalty) to teach agents when and how to
  fold. Result: 10× smaller active context, +20% on BrowseComp-Plus,
  +8.8% on SWE-Bench Verified vs. 327K-token ReAct baselines.
tags: [agents, context-management, rl, long-horizon, llm]
categories: paper-review
toc:
  sidebar: left
related_posts: true
---

**Paper.** Weiwei Sun, Miao Lu, Zhan Ling, Kang Liu, Xuesong Yao,
Yiming Yang, Jiecao Chen. *Scaling Long-Horizon LLM Agent via
Context-Folding.* ICML 2026. ByteDance Seed · Carnegie Mellon
University (LTI).
[[arXiv]](https://arxiv.org/abs/2510.11967) ·
[[project page]](https://context-folding.github.io/) ·
[[code]](https://github.com/sunnweiwei/FoldAgent)

---

## 0. The Picture in One Paragraph

LLM agents solving long-horizon tasks — deep web research, large
codebase editing — accumulate interaction history linearly with
task length. The dominant ReAct paradigm concatenates everything
into a single ever-growing context: observations, tool outputs,
reasoning traces, all of it. This produces two compounding failures:
the token budget runs out before the task completes, and performance
degrades as the model loses focus in a long context ("context rot").
Context-Folding proposes a different primitive: the agent can
**branch** into a sub-trajectory to handle a localized subtask,
and **fold** it upon completion — collapsing all intermediate steps
into a concise `return(message)` summary that rejoins the main
context thread. To make this learnable, **FoldGRPO** augments
standard GRPO with two dense token-level process rewards:
an Unfolded Token Penalty that fires whenever the main context
grows too large, and an Out-of-Scope Penalty for branches that
drift off-task. With a **32K active context budget**, the Folding
Agent achieves **62.0% on BrowseComp-Plus** and **58.0% on
SWE-Bench Verified** — outperforming a 327K-token ReAct baseline
by +20.0% and +8.8% respectively, at **10× smaller active context**.

---

## 1. The Problem — Context as the Ceiling on Long-Horizon Agents

The ReAct loop (Reason + Act + Observe) works well for short tasks.
For long-horizon tasks the problem is structural:

```
Step 1: [reasoning] [action] [observation]
Step 2: [reasoning] [action] [observation]
...
Step N: [reasoning] [action] [observation]
        ← all of the above still in context
```

At every step the model must fit the entire task history into the
context window. Two consequences:

- **Context exhaustion.** Token budgets cap around 32K–128K tokens
  in practice. Hard research or engineering tasks easily exceed
  this before resolution.
- **Context rot.** Even within the window, signal-to-noise
  degrades: an observation from step 3 that was crucial is
  competing with sixty later observations for the model's
  attention.

Standard remedies are passive and lossy: truncation drops the
oldest content (losing critical early state), and summarization
compresses at fixed intervals but cannot be end-to-end trained
and introduces systematic information loss.

The question Context-Folding asks: **what if the agent could
actively decide what to forget and when?**

---

## 2. The Move — Branch and Fold

Context-Folding introduces two new primitive actions into the
agent's action space:

**`branch(description, prompt)`** — The agent declares a subtask
description and enters a fresh sub-trajectory to handle it.
From the main context's perspective, `branch(...)` is a single
action. The sub-trajectory has its own growing context, but it
is isolated from the main thread.

**`return(message)`** — Called within a sub-trajectory once the
subtask is resolved. `message` is a concise self-written summary
of what was accomplished. Calling `return(...)` **folds** the
entire sub-trajectory: all intermediate steps disappear, and only
the summary is appended back to the main context thread.

The analogy to programming is direct: `branch` is like calling a
function, `return` is like returning a value. The internals of
the function call are hidden from the caller; only the return
value enters the main context.

```
Main context:
  [problem]
  [action]
  [branch("search for X", "find three sources on X")]
    Sub-trajectory:
      [search query 1] [results 1]
      [search query 2] [results 2]
      [search query 3] [results 3]
      [return("Found: source A (URL), source B (URL), source C (URL)")]
    ← Sub-trajectory discarded
  [Summary: "Found: source A, B, C"]
  [next action based on summary]
  ...
```

The main context remains compact because all multi-step operations
(iterative web searches, long file reads, tool-heavy exploration)
are offloaded to branches. The agent is allowed up to 10 branches
per task, and the main thread is capped at 32K active tokens.
Total tokens *processed* across branches can exceed 100K; the
agent cycles through them without the main thread ever growing
past 32K.

---

## 3. FoldGRPO — Teaching the Agent When and How to Fold

The branching primitive is easy to define but hard to learn.
Vanilla RL with standard GRPO is insufficient: the agent fails
to develop reliable policies for *when* to branch and *what*
to include in the `return(message)` summary.

FoldGRPO makes three modifications to standard GRPO:

### 3.1 Dynamic Folded Context During Training

During RL rollouts, the training environment applies context
folding dynamically in real time. The model sees exactly the
same folded context structure it will see at inference — not
the full unrolled trajectory. This closes the train/inference
distribution gap that would otherwise exist if training on
full-context trajectories but deploying with folding.

### 3.2 Unfolded Token Penalty (UTP)

When the total context length of the main thread exceeds 50%
of the working context limit, every token in the main thread
(except tokens in branch-issuing turns) receives a process
reward of $Q_{i,t} = -1$.

This creates a direct gradient signal against performing
token-intensive operations (long searches, large file reads)
directly in the main context. The agent learns to route these
into branches instead.

### 3.3 Out-of-Scope Penalty (OSP)

For each branch, an LLM judge (GPT-5-nano) evaluates whether
the agent conducted actions outside the declared subtask scope.
Out-of-scope behavior incurs $Q_{i,t} = -0.2$ on all tokens
in that branch. This teaches the agent to maintain disciplined
focus within a branch.

The final reward signal combines the standard sparse
task-completion reward (correct/incorrect) with UTP and OSP
as dense process rewards operating at the token level. Even
on long trajectories where the sparse reward provides no
gradient for hundreds of steps, UTP and OSP keep the learning
signal flowing.

---

## 4. Results

### 4.1 BrowseComp-Plus (deep research)

| Method | Active context | pass@1 |
|---|---|---|
| ReAct (long context) | 327K tokens | ~42% |
| ReAct (short, 32K) | 32K tokens | much lower |
| Summarization-based | 32K effective | lower |
| Folding Agent + GRPO | 32K active | 56.7% |
| **Folding Agent + FoldGRPO** | **32K active** | **62.0%** |

FoldGRPO over vanilla GRPO: **+5.3 pp**.
FoldGRPO over 327K ReAct baseline: **+20.0 pp** with **10× smaller active context**.

### 4.2 SWE-Bench Verified (software engineering)

| Method | Active context | pass@1 |
|---|---|---|
| ReAct (327K long context) | 327K tokens | ~49.2% |
| **Folding Agent + FoldGRPO** | **32K active** | **58.0%** |

FoldGRPO over 327K ReAct baseline: **+8.8 pp**.

### 4.3 Context efficiency

- Folding Agent main trajectory: **~8K tokens average**
- Total tokens processed across all branches: **>100K**
- Compression rate on main context: **>90%**

The agent processes substantially more information than a 32K
ReAct baseline while keeping the active context a fraction of
the size.

### 4.4 FoldGRPO vs. GRPO ablation

Removing UTP: main thread fails to stay compressed; context
overflows more often; performance drops.
Removing OSP: branches drift off-task; scope accuracy degrades.
Together UTP + OSP are each independently necessary.

---

## 5. Why It Matters

Three reasons:

1. **Active context management as a learnable skill.** Prior
   work treated context management as infrastructure (truncation,
   compression, longer windows). Context-Folding treats it as
   agent behavior — something the model must learn to do and
   that can be improved by RL. The branch-and-fold primitive
   is minimal but expressive.
2. **Dense process rewards unlock long-horizon RL.** The sparse
   task-completion signal in long-horizon tasks is too weak to
   drive learning. UTP and OSP provide token-level gradient
   signal at every step — not just at task completion. This is
   the same lesson MCTD ([review]({% link _posts/2026-07-02-mctd.md %}))
   applies in the planning domain: structured intermediate
   feedback is what enables scaling with compute.
3. **Architecture-separable.** FoldGRPO trains the policy to
   use folding; the backbone model is otherwise unchanged. This
   means the approach transfers across backbone updates: as
   better base models appear, FoldGRPO can retrain folding
   behavior on top without redesigning the system.

---

## 6. Limitations Worth Knowing

- **Cold-start dependency.** FoldGRPO needs the base model to
  spontaneously generate at least some valid folding trajectories
  to bootstrap contrastive gradients. Very specialized domains
  with near-zero base policy success are problematic.
- **LLM judge dependency.** OSP uses GPT-5-nano as an external
  judge during training — a proprietary API dependency that could
  propagate judge errors.
- **Hard tasks remain hard.** On the hardest BrowseComp-Plus
  subset, accuracy is only 4%. Folding solves the context
  management problem, not the hard reasoning problem.
- **Summary fidelity unverified.** A poorly written
  `return(message)` silently loses information. The agent has
  no mechanism to detect or recover from bad summaries.
- **Single-level branching.** Recursive folding (branches
  within branches) is not implemented, limiting expressiveness
  on tasks requiring multi-level decomposition.

---

## 7. The Takeaway for a First Reader

If you remember three things:

1. **Long-horizon agents fail because their context grows
   without bound.** Context-Folding gives agents two new actions:
   **`branch()`** to offload a subtask into an isolated
   sub-trajectory, and **`return(message)`** to fold it
   back as a summary. The main context stays ≤32K active tokens
   even as the agent processes >100K tokens total.
2. **FoldGRPO trains this behavior end-to-end** via GRPO with
   two dense process rewards: an **Unfolded Token Penalty**
   (gradient against bloating the main context) and an
   **Out-of-Scope Penalty** (gradient against drifting
   off-task in branches). These provide token-level signal at
   every step, not just task completion.
3. **62.0% on BrowseComp-Plus and 58.0% on SWE-Bench Verified**
   with 32K active context — **+20.0% and +8.8% over a 327K-token
   ReAct baseline at 10× smaller context**.

---

## References

- Sun, W., Lu, M., Ling, Z., *et al.* (2025/2026).
  *Scaling Long-Horizon LLM Agent via Context-Folding.* ICML 2026.
  [arXiv:2510.11967](https://arxiv.org/abs/2510.11967).
- Project page: <https://context-folding.github.io/>
- Code: <https://github.com/sunnweiwei/FoldAgent>
- Follow-ups: AgentFold ([arXiv:2510.24699](https://arxiv.org/abs/2510.24699)),
  FoldAct ([arXiv:2512.22733](https://arxiv.org/abs/2512.22733)).
- Related on this site:
  [MEMENTO]({% link _posts/2026-05-18-memento-context-management.md %})
  — context compaction via KV-cache compression, a complementary approach.

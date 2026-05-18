---
layout: post
title: "Meta-Harness: End-to-End Optimization of Model Harnesses"
date: 2026-05-18 10:00:00 +0900
description: >
  Paper review of Meta-Harness (Lee et al., 2026) — an outer-loop search over
  the *code around the LLM*, driven by a coding-agent proposer that reads
  prior candidates and their raw execution traces through the filesystem.
tags: [agents, harness, llm-serving, optimization, prompt-optimization]
categories: paper-review
toc:
  sidebar: left
related_posts: true
---

**Paper.** Yoonho Lee, Roshen Nair, Qizheng Zhang, Kangwook Lee, Omar Khattab,
Chelsea Finn. *Meta-Harness: End-to-End Optimization of Model Harnesses.*
arXiv:2603.28052, March 2026. Stanford · MIT · KRAFTON.
[[arXiv]](https://arxiv.org/abs/2603.28052) ·
[[project page]](https://yoonholee.com/meta-harness/) ·
[[code]](https://github.com/stanford-iris-lab/meta-harness)

---

## 1. The Problem the Paper Names

LLM applications are not just a model — they are a model plus a **harness**:
the surrounding code that decides what to store, what to retrieve, what to
show the model, what tools to expose, and how the workflow runs. In modern
systems (coding agents, RAG pipelines, long-running assistants) the harness
is large, hand-tuned, and routinely matters more than the choice of base
model.

The paper's claim is simple: **the harness itself should be the optimization
target**, and it should be optimized end-to-end against task performance.

Why this isn't already solved:

- *Prompt optimizers* (DSPy, OPRO, APE, etc.) tune **prompts**, not the
  surrounding code. They cannot rewrite the retriever, the context-management
  policy, or the tool wiring.
- *Text optimizers* that do attempt program-level search compress feedback
  too aggressively — they hand the proposer scores or short summaries
  instead of the raw evidence needed to diagnose failures.

## 2. The Method

Meta-Harness is an **outer loop** over harness code. At each step, an
*agentic proposer* (a coding agent — Claude Code in the reported runs)
generates a new candidate harness, the harness is executed on a task suite,
and the result is committed back into a shared filesystem.

The critical design choice is *what the proposer is allowed to see*:

| Channel | Prior text optimizers | Meta-Harness |
|---|---|---|
| Scores of past candidates | ✓ | ✓ |
| LLM-written summaries of past runs | sometimes | ✓ |
| **Source code of every prior candidate** | ✗ | ✓ |
| **Raw execution traces / logs** | ✗ | ✓ |
| Diagnostic budget per step | ≤ 26K tokens | up to **10M tokens** |

The proposer reads what it needs via standard tools (`grep`, `cat`, `ls`)
over a directory that contains every prior candidate's code, score, and
trace. There is no compression layer between the proposer and the evidence.

In the reported TerminalBench-2 run (10 iterations, Claude Opus 4.6) the
proposer reads a **median of 82 files per iteration**, split roughly:

- 41% — source code of prior harness candidates,
- 40% — execution traces from prior runs,
- 6% — score summaries,
- 13% — other files.

In other words, the proposer spends most of its attention budget on
**counterfactual diagnosis**: it reads logs of failures, identifies the
specific failure mode, and proposes a targeted code change grounded in
concrete evidence.

## 3. Experiments

The paper evaluates on three deliberately heterogeneous domains.

### 3.1 Online text classification

Meta-Harness optimizes a context-management harness against a
state-of-the-art baseline.

- **+7.7 points** over the SOTA context-management system.
- **4× fewer context tokens** at inference time.

This is the cleanest demonstration: the harness improves *and* shrinks
simultaneously, which a prompt-only optimizer cannot achieve.

### 3.2 Retrieval-augmented math reasoning (IMO-level)

A **single** harness discovered by Meta-Harness on one training model is
evaluated on **five held-out models** over 200 IMO-level problems.

- **+4.7 points** average gain across the five held-out models.

The transfer matters more than the absolute number: it suggests Meta-Harness
is not just overfitting to the optimizer's own search model.

### 3.3 Agentic coding — TerminalBench-2

Meta-Harness produces harnesses that **surpass the best hand-engineered
baselines** on TerminalBench-2. The ablation here is the heart of the paper.

## 4. The Ablation You Should Read First

The same outer-loop search is run with three proposer interfaces:

| Proposer sees | Median accuracy | Best accuracy |
|---|---:|---:|
| Scores only | ~34.9 | **41.3** |
| Scores + LLM-generated summaries | ~34.9 | **38.7** |
| Scores + summaries + source code + raw traces (full Meta-Harness) | **50.0** | **56.7** |

Two punchlines:

1. **Raw traces are the active ingredient.** Going from scores → full
   interface is a +15 jump in best accuracy.
2. **Summaries can be actively harmful.** LLM-generated summaries underperform
   scores-only on best accuracy (38.7 vs 41.3). The author's reading is that
   summaries compress away the diagnostically useful detail — the very
   things a proposer needs to localize a bug.

This is a strong empirical statement against the "summarize then propose"
pattern that pervades current text-based optimizers.

## 5. Where This Fits

Several recent trends converge here:

- **Agent harness design** (Anthropic's planner/generator/evaluator,
  Claude Code subagents, LangGraph supervisors). Meta-Harness sits one
  level up: instead of writing the harness, it *searches over* harnesses.
- **Filesystem as agent memory.** Like Anthropic's research-agent setups,
  Meta-Harness treats the filesystem as a long-term scratchpad shared
  across iterations. The shared filesystem is what makes the 10M-token
  diagnostic budget tractable — it's lazily paged in by the proposer's
  own tool calls, not stuffed into one context window.
- **Coding agents as optimizers.** The proposer is just Claude Code with
  read access to past runs. This is a clean instance of *using a coding
  agent as a search operator* rather than as an end-user tool.

## 6. Limitations / What I'm Watching For

- **Cost.** Each outer-loop step runs a coding agent with up to 10M tokens
  of diagnostic context plus a full harness evaluation. The paper reports
  10 iterations on TerminalBench-2; broader sweeps are not cheap.
- **Eval signal.** The method assumes a scalar score per candidate. Tasks
  without crisp automated evals (open-ended writing, subjective UX) need
  a separate evaluator agent — likely the next paper.
- **Search-model leakage.** Meta-Harness optimizes a harness with model
  *A* in the loop; held-out evaluation on models *B–F* gives some
  evidence of transfer, but the search-model's idiosyncrasies could
  still leak into the discovered code.
- **Convergence.** The ablation reports *median* and *best* — it would be
  useful to see learning curves and whether the proposer plateaus or
  keeps improving with more iterations.

## 7. Why I Think This Paper Matters

For the past two years the "scaffolding" around LLMs — retrievers,
context managers, tool routers, agent loops — has been written by hand,
guarded by intuition, and improved by trial and error. Meta-Harness
demonstrates that the scaffolding is itself an object that can be
**optimized**, and that the bottleneck is not the search algorithm but
**how much raw evidence you let the proposer see**.

The takeaway is not "use Meta-Harness." The takeaway is:

> If your optimization loop is summarizing the evidence before the
> optimizer sees it, you are throwing away the signal that makes
> optimization work.

That single observation reframes how I want to design every multi-agent
system from here on.

---

## References

- Lee, Y., Nair, R., Zhang, Q., Lee, K., Khattab, O., & Finn, C. (2026).
  *Meta-Harness: End-to-End Optimization of Model Harnesses.* arXiv:2603.28052.
- Project page: <https://yoonholee.com/meta-harness/>
- Code: <https://github.com/stanford-iris-lab/meta-harness>
- Related: my earlier notes on
  [agent harness design]({% link _explorations/2026-05-06-agent-harness-team.md %})
  and the
  [research agent team rules]({% link _explorations/2026-05-18-research-agent-team-rules.md %})
  that motivated this read.

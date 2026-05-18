---
layout: post
title: "FrontierSmith — Manufacturing Open-Ended Coding Problems to Train Better Code Agents"
date: 2026-05-18 22:30:00 +0900
description: >
  Paper review of FrontierSmith (FrontierCS Team, 2026) — an automated
  pipeline that mutates closed-ended competitive-programming problems
  into open-ended ones, filters them with an "idea divergence" metric,
  and uses the resulting data to substantially boost coding LLMs on
  FrontierCS and ALE-Bench.
tags: [llm, code-llm, data-synthesis, open-ended, benchmarks]
categories: paper-review
toc:
  sidebar: left
related_posts: true
---

**Paper.** FrontierCS Team. *FrontierSmith: Synthesizing Open-Ended
Coding Problems at Scale.* arXiv:2605.14445, May 2026.
UC Berkeley · UC San Diego · UW · Stanford · Princeton · MIT.
[[arXiv]](https://arxiv.org/abs/2605.14445) ·
[[FrontierCS benchmark]](https://github.com/FrontierCS/Frontier-CS) ·
[[blog]](https://frontier-cs.org/blog/harbor/)

---

## 0. The Picture in One Paragraph

The last two years of code-LLM progress are largely a story about
**closed-ended problems with known answers**: feature
implementation, bug fixes, competitive programming with binary
pass/fail. Real-world coding is mostly **open-ended** — heuristic
optimization, system design, "make this as good as you can within
these constraints" — and the field has been bottlenecked by the
absence of *training data* for that regime. FrontierSmith argues the
data shortage isn't fundamental, just unsolved: take the millions of
existing closed-ended problems, **mutate them into open-ended
variants** along three principled axes, **filter the mutations** with
a metric that measures whether different solvers actually use
different ideas, and you can manufacture this data automatically. A
Qwen3.5-9B trained on FrontierSmith data picks up **+8.82** on
FrontierCS and **+306 Elo** on ALE-Bench, beating human-curated
open-ended baselines.

---

## 1. The Setup — Why "Open-Ended" Is the Right Frame

A closed-ended coding problem has *one* correct answer (or one
correct equivalence class). Pass/fail is well-defined; the gradient
signal during RL training is clean; benchmarks are easy to score.
LeetCode, Codeforces, SWE-Bench bug fixes — all closed-ended.

An open-ended problem doesn't have a known optimum. You are scored on
a *continuous* quality measure relative to a baseline or to other
solvers — "minimize the total tour length," "maximize throughput,"
"reduce latency under this constraint." AtCoder Heuristic Contests
(AHC), most real-world systems work, and most research engineering
all live here.

The empirical wall: open-ended coding remains a weak spot for current
models. On FrontierCS, the benchmark introduced alongside this work,
**human experts score 95.41** on the algorithmic split; **Gemini 3.0
Pro scores 29.37**. The gap is not subtle. The paper's bet is that
the gap exists *primarily* because there's almost no training data
shaped like the test distribution — and that we can fix that by
*generating* the data.

---

## 2. The Three Mutation Operators

The starting material is closed-ended competitive-programming
problems, of which the world has many. FrontierSmith mutates each
problem along three principled axes that *reliably* turn pass/fail
into "as good as you can":

| Operator | What it changes | Example transformation |
|---|---|---|
| **Swap the Goal** | The objective | "Find *the* optimal solution" → "Find the **best** solution you can given a compute budget." A problem with a single right answer becomes a continuous optimization. |
| **Tighten the Output** | What counts as acceptable | Add real-world constraints (latency, memory, communication cost) under which the original optimal solution is infeasible — approximation now matters. |
| **Relax the Input** | What inputs you must handle | Drop assumptions that made the problem tractable (small N, structured data) and generalize to inputs at production scale — what worked on the toy version breaks. |

The reason these three are the operators rather than three among many:
each one **provably** removes the property that made the source
problem closed-ended (uniqueness of optimum, feasibility of optimum,
or tractability of the small-N solution).

---

## 3. Filtering: the Idea Divergence Metric

Most mutations produce nonsense — either still-closed-ended problems
or problems whose only "solutions" are degenerate. The pipeline needs
a filter that asks: *given this mutated problem, do different solvers
end up writing fundamentally different programs?* If the answer is no
("everyone still writes greedy"), the mutation didn't actually create
open-endedness.

FrontierSmith operationalizes this with an **idea divergence** metric
that estimates the probability two independent solvers arrive at
different *core algorithms*. Two complementary checks are combined:

1. **Semantic check.** Sample multiple solutions (from strong LLMs);
   have a *judge* LLM read each one and label the underlying strategy
   (greedy, DP, beam search, local search, ILP, …). Divergence ≈ the
   probability that two random solutions get different labels.
2. **Behavioral check.** Run each candidate solution on a battery of
   test cases. Compute *score vectors* (per-instance score). Cluster
   the score vectors. If they cluster tightly, solvers are
   behaviorally equivalent; if they spread, they're not.

Only mutations that score above thresholds on **both** checks pass
through to the next stage. This is the crucial step — without it the
synthesized dataset is mostly junk.

---

## 4. Building Continuous-Score Environments

A surviving open-ended problem still needs an *environment* before it
can be used for training. FrontierSmith builds three pieces
automatically:

- **A verifier** — replaces pass/fail with a continuous quality score
  over [0, 1] (or a domain-specific scale). Examples: tour length,
  schedule makespan, throughput, packing density.
- **Dynamic test-case generators** — programs that emit fresh
  instances at any requested scale. Important: with open-ended
  problems you can't ship a fixed test suite, because models would
  memorize it.
- **Sandboxed execution** — clean, reproducible runtime for safe
  solution execution.

The verifier is what makes RL practical on this data: a continuous
score is a real gradient signal, not just a sparse 0/1.

---

## 5. Headline Numbers

The paper trains **Qwen3.5-9B** on FrontierSmith-synthesized data and
evaluates on two open-ended coding benchmarks:

| Benchmark | Δ vs. Qwen3.5-9B base |
|---|---|
| **FrontierCS** (240 open-ended CS problems; 172 algorithmic) | **+8.82** score |
| **ALE-Bench** (AtCoder Heuristic Contests) | **+306.36 Elo** |

Two comparative findings from the ablations make the contribution
crisp:

- **Beats human-curated open-ended data.** Training on the same
  budget of *hand-written* open-ended problems is *worse* than
  training on FrontierSmith-synthesized data. The bottleneck wasn't
  human quality; it was human throughput.
- **The filter is load-bearing.** Removing the idea-divergence
  filter — i.e., training on all surviving mutations regardless of
  whether they actually became open-ended — degrades the result
  substantially. The filter is doing as much work as the mutation
  operators.

A separate, almost theatrical, validation: at UC Berkeley's official
CALICO programming contest, one problem required contestants to *beat
a frontier AI agent trained on FrontierSmith data*. Out of 2,000+
human contestants, **one** submission did.

---

## 6. The FrontierCS Benchmark

The paper introduces FrontierCS partly to evaluate FrontierSmith
fairly, but it's a worth-knowing artifact on its own:

- **240 open-ended CS problems** across two tracks.
- **172 algorithmic problems** that require only local computation
  (no agentic tool use).
- **Continuous 0–100 scoring** rather than pass/fail.
- **Human-expert reference:** 95.41 / 100 on the algorithmic split.

The benchmark plus a clean leaderboard is the kind of supporting
infrastructure that makes a sub-field move. ALE-Bench from Sakana AI
(AtCoder Heuristic Contests) covers similar territory and is the
natural cross-benchmark.

---

## 7. Why This Matters

Three reasons:

1. **It dissolves a data bottleneck rather than papering over it.**
   The standard response to "we don't have data shaped like task X"
   is either *more human curation* or *more aggressive distillation
   from a stronger model*. FrontierSmith is the third option — derive
   the missing shape from what already exists — and on this task it
   wins outright.
2. **The idea-divergence metric is reusable.** "Does this problem
   genuinely admit multiple distinct strategies?" is a useful filter
   far beyond coding: math olympiad problem synthesis, agent task
   design, RL curriculum construction. Anywhere you need to certify
   open-endedness, the same pair of checks (semantic-by-judge +
   behavioral-by-score-vector) seems applicable.
3. **Continuous environments shift what RL can do for code.** Pass/fail
   reward is what made RLVR work on closed-ended code; continuous
   reward is what makes RL feasible on heuristic engineering. This
   paper is the first I've seen that supplies that environment
   *automatically at scale*, which closes a loop other recent work
   (SOAR, FST, Meta-Harness) was pushing against from different
   directions.

This sits next to recent threads on this site that all argue *the
data and the environment are the leverage, not the model*:

- [SOAR review]({% link _posts/2026-05-18-soar.md %}) — letting a
  model design its *own* curriculum at the edge of learnability.
- [Meta-Harness review]({% link _posts/2026-05-18-meta-harness.md %})
  — fixing the model and optimizing the harness instead.
- [Fast-Slow Training trend note]({% link _posts/2026-05-18-learning-fast-and-slow.md %})
  — extracting more signal per rollout via prompt-level adaptation.

FrontierSmith is the *task-supply* version of the same argument:
the model can be held fixed if the problem distribution is rich
enough.

---

## 8. Limitations Worth Knowing

- **Source dependence.** Mutations start from existing
  competitive-programming problems. Whatever blind spots that source
  has are inherited. A purely greenfield open-ended domain (think:
  scientific simulation tasks no one has competitive-coded) isn't
  covered.
- **Verifier quality is the ceiling.** A continuous-score verifier
  written automatically can be *miscalibrated* (rewarding the wrong
  thing) or *gameable* (admitting trivial high scorers). The paper
  uses LLM-built verifiers; how far they hold under adversarial
  optimization is an open question.
- **Single model scale shown.** Qwen3.5-9B carries most of the
  results. Whether the gains scale to 32B / 70B and whether they
  compound with RL on the same continuous-score environment is the
  obvious next experiment.
- **Benchmark co-evolution risk.** FrontierCS is introduced by the
  same team. The cross-benchmark gain on ALE-Bench (independent,
  Sakana AI) helps a lot, but a third independent eval would settle
  the question.

---

## 9. TL;DR

If you remember three things:

1. **Open-ended coding is a data problem, not a model problem.** The
   gap between humans (95.41) and frontier models (29.37) on
   FrontierCS exists mostly because there's no training data shaped
   like the test set.
2. **Three mutation operators — Swap the Goal, Tighten the Output,
   Relax the Input — convert closed-ended competitive-programming
   problems into open-ended ones**, and an **idea-divergence filter**
   (semantic-by-judge + behavioral-by-score-vector) keeps only the
   mutations that actually became open-ended.
3. **Training Qwen3.5-9B on FrontierSmith data beats human-curated
   open-ended baselines** — **+8.82 on FrontierCS, +306 Elo on
   ALE-Bench** — and the filtered, automatically built
   continuous-score environments are reusable as RL infrastructure
   for future code-LLM work.

---

## References

- FrontierCS Team. (2026). *FrontierSmith: Synthesizing Open-Ended
  Coding Problems at Scale.*
  [arXiv:2605.14445](https://arxiv.org/abs/2605.14445).
- FrontierCS benchmark repository:
  <https://github.com/FrontierCS/Frontier-CS>
- FrontierCS blog (long-context framing):
  <https://frontier-cs.org/blog/harbor/>
- ALE-Bench (Sakana AI):
  [arXiv:2506.09050](https://arxiv.org/abs/2506.09050) ·
  <https://sakana.ai/ale-bench/>
- Related on this site:
  [SOAR review]({% link _posts/2026-05-18-soar.md %}),
  [Meta-Harness review]({% link _posts/2026-05-18-meta-harness.md %}),
  [Fast-Slow Training trend note]({% link _posts/2026-05-18-learning-fast-and-slow.md %}).

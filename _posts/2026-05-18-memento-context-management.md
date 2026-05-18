---
layout: post
title: "MEMENTO: Teaching Reasoning Models to Compress Their Own Thinking"
date: 2026-05-18 21:30:00 +0900
description: >
  Trend note on MEMENTO (Kontonis et al., 2026, MSR/UW-Madison) — a
  training recipe that teaches a reasoning model to segment its own chain
  of thought, compress each segment into a "memento" summary, and evict
  the original KV cache, yielding 2–3× lower peak KV with near-baseline
  accuracy.
tags: [llm, reasoning, kv-cache, inference, context-engineering]
categories: trends
toc:
  sidebar: left
related_posts: true
---

**Source.** Dimitris Papailiopoulos thread on X (May 2026) introducing
*MEMENTO: Teaching LLMs to Manage Their Own Context*, by Vasilis
Kontonis, Yuchen Zeng, Shivam Garg, Lingjiao Chen, Hao Tang, Ziyan
Wang, Ahmed Awadallah, Eric Horvitz, John Langford, and Dimitris
Papailiopoulos (Microsoft Research AI Frontiers Lab / UW-Madison).
[[arXiv:2604.09852]](https://arxiv.org/abs/2604.09852) ·
[[MSR article]](https://www.microsoft.com/en-us/research/articles/memento-teaching-llms-to-manage-their-own-context/) ·
[[OpenMementos data]](https://huggingface.co/datasets/microsoft/OpenMementos)

---

## Why This Matters

Reasoning-trained models routinely emit **hundreds of thousands of
tokens** per inference call. Every one of those tokens lives in the KV
cache and is attended to at every subsequent step — regardless of
whether it still carries useful information. The structural complaint
the MEMENTO authors make is simple:

> *"The model has no built-in mechanism to compact what it has figured
> out, keep the conclusions, and move on."*

External fixes exist — sliding windows, eviction heuristics, summarizer
calls between turns — but they're scaffolds bolted *around* the model.
MEMENTO instead **trains the model to do the compaction itself**, as
part of the same generative loop that produces the reasoning. The KV
cache becomes something the model actively manages, not something the
serving stack tries to manage on its behalf.

That reframing is what makes this worth treating as a trend rather
than a single paper. If models can be taught to author their own
memory hierarchy, several lines of current work (KV-compression
research, agent-memory frameworks, long-horizon RL) start to converge.

---

## The Mechanism in One Picture

```
┌────────── block 1 ──────────┐ ┌── memento 1 ──┐
│ reasoning tokens (~thousands)│ │ dense summary │  ← KV kept
└──────────────────────────────┘ └───────────────┘
                                                    ↓ block 1 KV evicted
┌────────── block 2 ──────────┐ ┌── memento 2 ──┐
│ reasoning tokens             │ │ dense summary │
└──────────────────────────────┘ └───────────────┘
                                                    ↓ block 2 KV evicted
...
```

The model emits a block of chain-of-thought, then emits a structured
memento that captures the block's state. Once the memento is written,
the **original block's KV entries are masked and physically evicted**
from the cache. Subsequent attention sees only the surviving mementos
plus the current block — giving a *sawtooth* memory profile instead of
a monotonically growing one.

---

## How the Model Learns This

The training data is the contribution as much as the recipe. The team
released **OpenMementos** — 228K reasoning traces (54% math, 19% code,
27% science) built on top of OpenThoughts-v3, where every trace is
**segmented** and every segment has a **memento** attached.

The annotation pipeline is itself instructive:

1. **Atomic split** — break the trace into sentences, code blocks,
   equations.
2. **Boundary scoring** — an LLM scores each candidate boundary 0–3
   ("mid-thought" → "major transition").
3. **DP segmentation** — dynamic programming picks segment boundaries
   that balance segment size vs. boundary quality.
4. **Compressor LLM** — writes a candidate memento per segment,
   biased toward *state preservation* (formulas, values, methods, open
   subgoals) rather than vanilla summarization.
5. **Judge + refine** — a judge LLM scores the memento on six axes
   (formula fidelity, numeric values, methods, validation, no
   hallucination, result-first structure) and the compressor revises.
   Single-pass compression hits a 28% pass rate; **two refinement
   rounds bring it to 92%** — the iterative loop is essential, not
   ornamental.

The actual training is a **two-stage SFT curriculum**:

- **Stage 1** — standard causal attention on the full annotated trace.
  The model learns the *format* of mementos without compression
  pressure.
- **Stage 2** — block masking is turned on. Once a memento is emitted,
  the preceding block is invisible to subsequent attention. The model
  is now *forced* to pack all needed state into mementos, because the
  fallback is gone.

Two design choices stand out:

- **Curriculum matters.** Direct stage-2 training from scratch
  underperforms — the model needs to learn the format before it learns
  the discipline.
- **~30K samples × 5 epochs** at 32K context is enough. This is not a
  pretraining-scale intervention.

---

## Headline Numbers

Across Qwen2.5-7B, Qwen3-{8B, 32B}, Phi-4-Reasoning-14B, and
OLMo3-7B-Think:

| Metric | Result |
|---|---|
| Peak KV cache | **2–3× lower** than baseline |
| Throughput (B200) | **~1.75×** (4,290 vs. 2,447 tok/s) |
| Accuracy gap | small initially, **shrinks with scale**, **closed entirely by RL** on Qwen3-8B (66.2% AIME'25) |
| Capability preservation | 96.4% overlap with baseline on solvable problems at k=64 |
| Compression ratio in data | ~6× (≈11K reasoning tokens → <2K memento tokens per trace) |

The pattern is consistent: most of the cost is paid in *consistency*
(single-pass accuracy wobbles a bit) rather than in raw capability;
majority voting at k=3 recovers baseline, and RL fine-tuning closes
the gap entirely on the model where it was applied.

---

## The Interesting Mechanistic Twist

A naive read of "mask the block and keep the memento" suggests the
model is now strictly operating off the memento. The paper's ablations
say otherwise.

**Restart ablation.** If the KV cache is *recomputed from scratch* at
each memento boundary — discarding the residual state — AIME'24 drops
from 66.1% to **50.8%**. So a non-trivial amount of useful information
is *not* in the memento token sequence itself; it's in the **KV
representations of surrounding tokens** that survive masking via the
residual stream.

**Linear-probe study.** The authors inject 5-digit passcodes into
masked blocks and train linear probes on downstream memento KV states.
The probes can read those passcodes back, with leakage that:

- **concentrates in deeper layers**,
- **decays with distance** (still detectable seven blocks away),
- **scales with model capacity**.

That is, masked content survives implicitly through residual + causal
attention even when the explicit tokens are gone. MEMENTO depends on
this implicit channel, which is why a "just restart the cache" baseline
isn't equivalent.

---

## Where This Sits in the Trend Landscape

MEMENTO is the cleanest current expression of a broader move: **treat
the model's own context as something it manages, not something the
serving stack manages**. Adjacent work pulling in the same direction:

- **KV-cache compression research** — ThinKV, R-KV, Breadcrumbs,
  TriAttention — all targeting the same memory wall, but from the
  *serving* side: trained-once policies, eviction heuristics, attention
  approximations. MEMENTO's bet is that the model itself can do better
  than a fixed policy because it knows what it just wrote.
- **Note-taking / scratchpad agents** — the
  [research-agent rulebook]({% link _explorations/2026-05-18-research-agent-team-rules.md %})
  on this site uses explicit files (`intra/`, `extra/`) to externalize
  agent memory. MEMENTO internalizes the same idea inside a single
  reasoning trace.
- **Context-engineering literature** — Anthropic's *Effective Context
  Engineering for AI Agents* argues that long-running agents need
  structured compaction; MEMENTO is the inside-the-model version of
  that argument.
- **Fast-Slow Training (FST)** ([trend note]({% link _posts/2026-05-18-learning-fast-and-slow.md %})) —
  reframes adaptation as two surfaces (slow weights, fast prompt).
  MEMENTO opens a third surface — the model's *self-curated* context —
  and argues it's trainable end-to-end.

If you squint, all four threads are answering the same question:
*what's the right granularity at which a model controls its own state?*
MEMENTO's answer is "segments of its own chain-of-thought, with a
sawtooth KV footprint."

---

## What I'm Watching

- **Scale on the RL recipe.** Closing the consistency gap on Qwen3-8B
  with RL was the headline result; whether the same closes on 32B and
  larger is the obvious next experiment.
- **Cross-family transfer of OpenMementos.** Two-stage SFT works on
  Qwen3 / Phi-4 / OLMo3. Whether memento *format* generalizes — i.e.
  could you swap in a new compressor without re-annotating — would
  determine how viral the dataset becomes.
- **Agent-loop integration.** MEMENTO compresses a single reasoning
  call. The natural extension is multi-turn: each tool call or
  sub-agent return becomes a block boundary, and the model emits a
  memento across the entire trajectory. This would be the cleanest
  bridge between "reasoning model" and "agent" architectures.
- **Memento as a probe.** The residual-stream leakage result is, to my
  eye, the most generally interesting piece. A reproducible knob for
  *what survives an attention mask via residual connections* is a
  useful interpretability instrument on its own.

---

## TL;DR

MEMENTO trains a reasoning model to **segment its own chain of thought,
emit a dense "memento" summary per segment, and evict the original
KV-cache entries** — turning a flat, monotonically growing context into
a sawtooth memory profile. Two-stage SFT on a 228K-trace public dataset
(OpenMementos) is enough; peak KV drops 2–3×, throughput nearly
doubles, and accuracy is preserved (or, with RL on the right base
model, fully restored). The deeper claim is structural: **context
management is a model skill, not a serving-layer problem** — and the
residual-stream leakage that lets evicted blocks still influence
downstream attention is the mechanism that makes it work.

---

## References

- Kontonis, V., Zeng, Y., Garg, S., Chen, L., Tang, H., Wang, Z.,
  Awadallah, A., Horvitz, E., Langford, J., & Papailiopoulos, D.
  (2026). *MEMENTO: Teaching LLMs to Manage Their Own Context.*
  [arXiv:2604.09852](https://arxiv.org/abs/2604.09852).
- Microsoft Research article:
  <https://www.microsoft.com/en-us/research/articles/memento-teaching-llms-to-manage-their-own-context/>
- OpenMementos dataset:
  <https://huggingface.co/datasets/microsoft/OpenMementos>
- Related on this site:
  [Fast-Slow Training trend note]({% link _posts/2026-05-18-learning-fast-and-slow.md %}),
  [Rules for an AI Research Agent Team]({% link _explorations/2026-05-18-research-agent-team-rules.md %}).

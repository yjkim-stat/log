---
layout: post
title: "AlpaServe — Statistical Multiplexing with Model Parallelism for DL Serving"
date: 2026-05-21 11:30:00 +0900
description: >
  Paper review of AlpaServe (Li et al., OSDI 2023) — the systems paper
  that argued model parallelism is not just for fitting big models on
  big clusters but for *statistically multiplexing* bursty multi-model
  workloads, and showed it can serve requests at up to 10× higher
  rates or absorb 6× more burstiness at >99% SLO attainment.
tags: [inference, llm-serving, systems, model-parallelism, scheduling]
categories: paper-review
toc:
  sidebar: left
related_posts: true
---

**Paper.** Zhuohan Li, Lianmin Zheng, Yinmin Zhong, Vincent Liu,
Ying Sheng, Xin Jin, Yanping Huang, Zhifeng Chen, Hao Zhang,
Joseph E. Gonzalez, Ion Stoica. *AlpaServe: Statistical Multiplexing
with Model Parallelism for Deep Learning Serving.* OSDI 2023.
UC Berkeley · Google · Peking University · Stanford · UCSD.
[[arXiv:2302.11665]](https://arxiv.org/abs/2302.11665) ·
[[OSDI paper]](https://www.usenix.org/conference/osdi23/presentation/li-zhouhan) ·
[[code]](https://github.com/alpa-projects/mms)

---

## 0. The Picture in One Paragraph

The textbook reason for **model parallelism** is "the model is too
big to fit on one GPU; split it." AlpaServe argues for a different
reason that became, in retrospect, the foundation under most modern
multi-model LLM serving stacks: **even when each model fits on one
GPU, splitting it across multiple GPUs is the right move whenever
your workload is bursty and multi-model.** The reason is *statistical
multiplexing* — pooling many bursty traffic streams onto a shared
fleet smooths the aggregate, the same way trunk-line provisioning
works in telephony. Replication-and-bin-packing does the opposite:
each model gets its own GPU and a burst on one model starves while
GPUs holding *other* models sit idle. AlpaServe formalizes the
tradeoff between model-parallel overhead and multiplexing gain,
proposes a placement algorithm that searches the joint space of
(GPU-group partition, parallelism strategy, model assignment), and
shows on production traces that the resulting system handles
**~10× higher request rates** or **~6× more burstiness** at
**>99% SLO attainment**.

---

## 1. The Setup — Why Replication-and-Bin-Pack Fails Under Bursts

The natural multi-model serving design is the simple one:

> For each model, allocate enough GPU copies to handle its expected
> load. Pack as many copies as fit on each GPU. Route each request
> to a free copy of the right model.

This is essentially what early systems like **Clipper** did, and it
works well when arrival rates are smooth and predictable. Two things
break it.

**Burstiness.** Real production traffic is not Poisson; it's
*bursty*. The Microsoft Azure traces used in the paper have peaks
that are 10×+ above the long-run average for individual models.
Under a burst, the model's dedicated GPUs queue; SLOs miss; the
"spare" GPUs hosting *other* models can't help, because they don't
have the right weights loaded.

**Head-of-line blocking.** A handful of long requests in a queue
hold up everything behind them. A serving system that can only
process one model per GPU is forced to choose between waiting and
dropping.

The deep structural complaint: **per-model GPU pools are
statistically independent**, which is exactly the wrong thing if you
want bursty streams to cancel each other out.

---

## 2. The Idea — Model Parallelism *as Multiplexing*

Statistical multiplexing is the classical answer: pool many bursty
sources onto a shared channel and the aggregate is much smoother
than any individual source. For DL serving, that means putting
*multiple models on the same set of GPUs* and letting any request to
any model use any GPU.

The catch: each forward pass needs *all the model's weights*. So the
question becomes — how do you make the GPUs of a pool capable of
serving *any* of the colocated models?

AlpaServe's answer is **model parallelism — specifically pipeline
parallelism**. Split each model into $k$ stages across $k$ GPUs.
Multiple models can share the same $k$-GPU group; a burst on model
A and a quieter moment on model B interleave naturally because each
request only occupies a stage at a time, not the whole GPU.

This is counter-intuitive because the conventional wisdom says
model parallelism is *expensive* — extra communication, extra
synchronization, lower per-request throughput than a single-GPU
copy. AlpaServe's contribution is to make this trade *quantitative*:

> Model parallelism has a per-request overhead $O$ (communication +
> pipeline bubble + smaller per-stage compute) but gives a
> multiplexing gain $M$ that grows with the number of colocated
> bursty streams and the burst factor. For bursty multi-model
> workloads, $M$ dominates $O$ over a wide regime.

The whole paper is the formalization, the algorithm to find a good
operating point, and the empirical demonstration that the regime
exists.

---

## 3. The Tradeoff in Pictures

```
   Replication + bin-pack                Model-parallel colocation
   (Clipper-style)                       (AlpaServe-style)

   GPU 1: [Model A copy]                 GPU 1: [A stage 1 | B stage 1 | C stage 1]
   GPU 2: [Model A copy]                 GPU 2: [A stage 2 | B stage 2 | C stage 2]
   GPU 3: [Model B copy]                 GPU 3: [A stage 3 | B stage 3 | C stage 3]
   GPU 4: [Model C copy]                 GPU 4: [A stage 4 | B stage 4 | C stage 4]

   - A burst on A blocks A's queue       - A burst on A streams through all 4 GPUs
   - GPUs 3, 4 are idle                  - B and C still progress in pipeline bubbles
   - per-request latency: low            - per-request latency: slightly higher
   - tail latency under burst: bad       - tail latency under burst: dramatically better
```

The picture on the right is what AlpaServe is selling. The cost is
a constant overhead per request; the benefit is that the *whole
cluster* now backs every model.

---

## 4. The Placement Problem

Once you accept "use model parallelism to multiplex," the system
question becomes: *which* parallelism strategy and *which* model
groupings?

AlpaServe formulates this as a joint search over three things:

1. **GPU-group partition** — split the cluster's $N$ GPUs into
   disjoint groups, each of size $k_g$.
2. **Parallelism strategy per group** — for each group, pick a
   tensor / pipeline / hybrid parallelism configuration.
3. **Model-to-group assignment** — decide which models live in
   which group (a model can be replicated across groups).

The objective is **SLO attainment** under a target arrival
distribution: maximize the fraction of requests that complete
within their latency bound.

This is a hard combinatorial problem. AlpaServe's approach:

- **Simulator-driven evaluation.** A discrete-event simulator
  estimates SLO attainment for any candidate placement. This is
  fast enough to score thousands of placements.
- **Enumeration + greedy search.** Enumerate plausible group
  partitions; for each, enumerate parallelism configurations using
  Alpa's existing auto-parallelization machinery; greedily assign
  models to groups by load.
- **Online refinement.** At runtime, a scheduler does **continuous
  batching** within each group and **request routing** across
  groups, with awareness of pipeline-stage occupancy.

The simulator is doing real work here: the search space is too big
for closed-form analysis, and too coupled for naive heuristics.

---

## 5. Evaluation

The experimental setup matches the paper's argument carefully.

### 5.1 Workloads

- **Microsoft Azure Function trace (MAF1, MAF2)** — production
  bursty arrival patterns.
- **Synthetic Gamma-process arrivals** — to sweep burstiness
  parameters cleanly.
- **A heterogeneous "model zoo"** with a mix of BERT-class,
  GPT-class, and larger models.

### 5.2 Baselines

- **Clipper-style replication + bin-pack.** Dedicated GPU pools per
  model.
- **AlpaSPMD.** SPMD model parallelism *without* statistical
  multiplexing — i.e., model parallelism for capacity reasons but
  one model per group.
- **Various oracle baselines** for sanity.

### 5.3 Headline numbers

| Metric | AlpaServe vs. best baseline |
|---|---|
| Throughput at fixed SLO (99% attainment) | **up to 10× higher request rate** |
| Tolerable burstiness at fixed SLO | **up to 6× more burst-factor** |
| SLO attainment under matched load | **>99%** where baselines drop to single-digit percentages |

The reading of these numbers: under *steady* load, AlpaServe is
comparable to a well-tuned replication system (the model-parallel
overhead is real). Under *bursty* load, AlpaServe pulls
dramatically ahead, because the baselines have no way to absorb
bursts that exceed the dedicated per-model capacity.

### 5.4 The right ablations

- **Vary burstiness** — the AlpaServe gain *increases* with burst
  factor. Confirms the mechanism.
- **Vary the number of colocated models** — multiplexing gain grows
  sublinearly with colocation count, consistent with the law-of-large-numbers
  intuition.
- **Vary parallelism configuration** — pipeline parallelism gives
  the bulk of the multiplexing benefit; tensor parallelism is
  complementary, not a substitute.

---

## 6. Why It Matters

Three reasons, in increasing order of generality.

1. **It changed the default for multi-model LLM serving.** Before
   AlpaServe, "split a model across GPUs only if you can't fit it"
   was conventional. After AlpaServe, model parallelism became a
   *load-balancing* tool. Modern serving systems —
   [vLLM, SGLang, TensorRT-LLM, TGI]({% link _explorations/2026-05-21-why-inference-is-hard-notes.md %}) —
   all assume this regime now.
2. **It formalized a tradeoff people were arguing about without
   numbers.** Pre-AlpaServe debates about "is model parallelism
   worth it for serving" were qualitative; the paper turned them
   into a measurable curve over burstiness and colocation count.
3. **It re-introduced a 100-year-old systems idea (statistical
   multiplexing) into ML serving.** The framing — "ML serving is
   trunk-line provisioning, with model parallelism as the
   trunking mechanism" — is the kind of cross-disciplinary handle
   that keeps generating downstream work (continuous batching,
   prefix-cache sharing, paged KV, RadixAttention) because it
   gives a unifying objective.

For where to read next on this site: the
[Inference Architecture report]({% link _reports/inference-architecture.md %})
is the broader survey; the
[*Why Inference is Hard* notes]({% link _explorations/2026-05-21-why-inference-is-hard-notes.md %})
walk through the modern engine landscape that builds on top of
AlpaServe's argument.

---

## 7. Limitations Worth Knowing

- **Pre-LLM era assumptions.** The paper predates the dominance of
  *autoregressive decoding* as the serving bottleneck. Modern LLM
  serving's main pain point is the **decode** phase's
  memory-bandwidth limit on KV cache, which AlpaServe's framing
  doesn't directly model. The *statistical-multiplexing* argument
  still applies, but the *parallelism choice* in modern systems
  weighs KV-cache placement very differently.
- **Homogeneous GPU assumption.** Real clusters are mixed (A100,
  H100, B200, different memory tiers). AlpaServe's placement
  algorithm assumes a homogeneous device pool; extending it cleanly
  is non-trivial.
- **Static workload profile.** Placement is computed against an
  estimated arrival distribution. Diurnal shifts, traffic regime
  changes, and product launches require re-planning; AlpaServe
  doesn't address online re-placement as a first-class problem.
- **Model-mix assumed known.** The set of served models and their
  approximate load mix is treated as input. Auto-discovery of which
  models to colocate based on observed traffic is left for later
  work.

---

## 8. The Takeaway for a First Reader

If you remember three things:

1. **Replication-and-bin-pack fails under bursts** because
   per-model GPU pools are statistically independent and idle GPUs
   on quiet models can't help busy models.
2. **Model parallelism is multiplexing**, not just capacity
   provisioning. Splitting each model across a shared GPU group
   lets all GPUs back all colocated models, and the
   law-of-large-numbers takes care of bursts.
3. **There is a measurable tradeoff** between model-parallel
   overhead and multiplexing gain. AlpaServe formalizes the
   tradeoff, searches the joint placement / parallelism space with
   a simulator-driven algorithm, and demonstrates **~10× higher
   request rate** or **~6× more burst tolerance** at >99% SLO on
   production traces.

That's the arc: rethink why we use model parallelism → formalize
the tradeoff → find a good operating point → show it on real
traffic.

---

## References

- Li, Z., Zheng, L., Zhong, Y., Liu, V., Sheng, Y., Jin, X., Huang,
  Y., Chen, Z., Zhang, H., Gonzalez, J. E., & Stoica, I. (2023).
  *AlpaServe: Statistical Multiplexing with Model Parallelism for
  Deep Learning Serving.* OSDI 2023.
  [arXiv:2302.11665](https://arxiv.org/abs/2302.11665) ·
  [USENIX](https://www.usenix.org/conference/osdi23/presentation/li-zhouhan).
- Code: <https://github.com/alpa-projects/mms>
- OSDI '23 talk:
  <https://www.youtube.com/watch?v=k0fuwdkN4LA>
- Related on this site:
  [Inference Architecture report]({% link _reports/inference-architecture.md %}),
  [*Why Inference is Hard* notes]({% link _explorations/2026-05-21-why-inference-is-hard-notes.md %}).

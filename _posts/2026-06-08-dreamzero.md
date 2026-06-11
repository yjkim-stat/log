---
layout: post
title: "DreamZero — World Action Models as Zero-shot Robot Policies"
date: 2026-06-08 10:00:00 +0900
description: >
  Paper review of DreamZero (Ye, Ge et al., NVIDIA GEAR, 2026) —
  a 14B World Action Model built on a pretrained video diffusion
  backbone that jointly predicts future video frames and robot
  actions, delivering 2× generalization to unseen tasks vs. state-of-
  the-art VLAs and running real-time closed-loop control at 7Hz.
tags: [robotics, world-models, video-diffusion, vla, zero-shot]
categories: paper-review
toc:
  sidebar: left
related_posts: true
slide_deck: /assets/paper_review_html_slides/DreamZero%20%EB%85%BC%EB%AC%B8%20%ED%95%B4%EC%84%A4%20(standalone).html
---

**Paper.** Seonghyeon Ye, Yunhao Ge, Kaiyuan Zheng, Shenyuan Gao,
Sihyun Yu, George Kurian, Suneel Indupuru, You Liang Tan, Chuning
Zhu, Jiannan Xiang, Ayaan Malik, Kyungmin Lee, William Liang,
Nadun Ranawaka, Jiasheng Gu, Yinzhen Xu, Guanzhi Wang, Fengyuan
Hu, Avnish Narayan, Johan Bjorck, Jing Wang, Gwanghyun Kim,
Dantong Niu, Ruijie Zheng, Yuqi Xie, Jimmy Wu, Qi Wang, Ryan
Julian, Danfei Xu, Yilun Du, Yevgen Chebotar, Scott Reed, Jan
Kautz, Yuke Zhu, Linxi "Jim" Fan, Joel Jang, *et al.*
*World Action Models are Zero-shot Policies.*
arXiv:2602.15922, Feb 2026. NVIDIA GEAR Lab.
[[arXiv]](https://arxiv.org/abs/2602.15922) ·
[[project page]](https://dreamzero0.github.io/) ·
[[code]](https://github.com/dreamzero0/dreamzero) ·
[[HF checkpoint]](https://huggingface.co/GEAR-Dreams/DreamZero-DROID)

> 📑 **Companion slide deck.** A self-contained HTML walk-through
> of this review (in Korean) is available at
> [DreamZero 논문 해설 (standalone).html]({{ page.slide_deck | relative_url }}).
> The deck and this post share the same structure; the post is the
> long-form English version.

---

## 0. The Picture in One Paragraph

State-of-the-art Vision-Language-Action (VLA) models — OpenVLA,
pi-zero, RT-2 — generalize *semantically* (new objects, new
phrasings) but fail to generalize to *new physical motions* in new
environments. They've been trained as policies, conditioned on
massive amounts of robot demonstration data; what they've never
seen, they can't do. DreamZero proposes a different shape: take a
**pretrained video diffusion model** (which has already internalized
how the visual world evolves) and train it to *jointly predict the
next video frames **and** the actions that produce them*. The
result is a **World Action Model (WAM)** that uses video as a dense,
physically-grounded representation of dynamics. With a 14B
autoregressive video DiT backbone, flow-matching, and a KV-cache
trick that swaps generated frames for ground-truth observations
every action chunk, DreamZero hits **62.2% average task progress on
DROID** vs. **27.4% for the best pretrained VLA baseline** (2×+
improvement), reaches **49% task progress on tasks with unseen
verbs** vs. **25–32%** for SOTA VLAs, and runs **closed-loop control
at ~7Hz** on Blackwell-class GPUs.

---

## 1. The Problem — Why VLAs Stall on New Physical Motions

A VLA is fundamentally a *policy*. It maps `(observation, language
instruction) → action`. Training data is human demonstrations of
robot tasks, often in the millions or tens of millions of episodes.

VLAs have proven excellent at one kind of generalization and bad at
another:

| Generalization axis | VLA status | Why |
|---|---|---|
| Semantic (new object names, new phrasings) | **Good** | Inherits VLM's language + vision priors |
| Physical (new motion patterns, new environments) | **Poor** | Trained only on demonstrated behaviors; never had to *predict consequences* |

The structural complaint DreamZero makes: **a policy that has only
ever seen `(obs, lang, action)` triples has no internal model of
*how the world responds to actions*.** It can imitate trained
behaviors, but it can't reason about novel ones because it lacks
the world-dynamics prior. Throwing more demonstration data at the
problem helps with the demonstrated tail; it doesn't fix the
structure.

---

## 2. The Reframing — World Action Models

DreamZero's answer is to **flip the modeling target**. Instead of
modeling

$$
p(a_t \mid o_{\le t}, \ell)
$$

(an action policy), model

$$
p(o_{t+1:t+H}, a_{t:t+H-1} \mid o_{\le t}, \ell)
$$

— a **joint distribution over future video frames *and* future
actions**, given current observations and language. The video
prediction half forces the model to internalize physical dynamics;
the action prediction half is the policy.

The bet is that **video is a dense, scalable representation of how
the world evolves**, and a model that can predict *next frames*
under different intended actions has, implicitly, a model of the
world. The actions are then "decoded" from that internal dynamics
model — making the policy a downstream consequence of the world
model, not a separately learned imitation.

This is structurally close to several recent threads:

- **ECHO** ([trend note]({% link _posts/2026-05-18-echo-terminal-world-models.md %})) —
  train on environment-response tokens, not just action tokens. The
  same instinct: predicting consequences builds an implicit world
  model.
- **Parcae** ([review]({% link _posts/2026-05-18-parcae.md %})) —
  diagnose architectural instability and fix by parameterization.
  DreamZero does similar work for autoregressive video diffusion at
  closed-loop control speed.

---

## 3. Architecture

### 3.1 The video-diffusion backbone

DreamZero is built on a pretrained **image-to-video diffusion
model** — specifically Wan2.1-I2V-14B (40 DiT layers, hidden
dimension 5120). The backbone has already been trained on internet-
scale video to predict next frames given a starting image; this is
the world-dynamics prior the policy will inherit.

### 3.2 Autoregressive DiT with flow matching

Standard video diffusion is **non-autoregressive** — it denoises a
whole clip at once. That's the wrong shape for closed-loop control,
where you need to roll out a few frames, see what the world
actually did, and condition the next prediction on the real
observation.

DreamZero makes the backbone **autoregressive** by training with
flow matching instead of standard denoising and structuring the DiT
to produce frames sequentially. This lets the model emit an
**action chunk** (a few actions for the next ~150ms), execute them,
observe the real next frame, and feed that real observation back as
the next prefix.

### 3.3 Separate decoders for video and action

The DiT outputs to **two separate decoder heads** that share the
backbone:

- A **video decoder** producing the predicted next frames.
- An **action decoder** producing the predicted next action chunk.

Both decoders are trained jointly under flow matching. The shared
backbone is forced to carry information sufficient for *both* —
which is what makes the action predictions "world-aware."

### 3.4 The closed-loop KV-cache swap

The single most important inference-time trick:

> After every action chunk executes in the real world, the resulting
> **ground-truth observation overwrites the generated frame in the
> KV cache**.

Why this matters: autoregressive video models accumulate prediction
error over time (compounding drift). In open-loop generation you
just live with the drift. In *closed-loop control*, you don't have
to — every few frames you get an actual observation back from the
robot. DreamZero feeds that observation into the KV cache as if it
were the model's own prediction, immediately resetting drift while
retaining all the prior computation.

This is what makes a 14B autoregressive video model viable as a
real-time controller, not just an open-loop world simulator.

### 3.5 DreamZero-Flash — decoupled noise schedules

A faster variant for tighter latency. The trick: during training,
**bias video latents toward higher noise states** while keeping
**action noise uniform**. This trains the model to infer actions
from *noisy* visual context — meaning at inference time you can
run fewer denoising steps on the video side without hurting action
quality.

Net effect: fewer sampling steps, same control performance.

---

## 4. Training Data and Recipe

### 4.1 DROID — heterogeneous robot data

DreamZero is trained on **DROID**, one of the most heterogeneous
open-source robot manipulation datasets — many tasks, many
embodiments, many camera setups. The standard wisdom is that
heterogeneity hurts training (mode interference, inconsistent
control conventions). DreamZero's bet is the opposite: a *world-
modeling* objective benefits from heterogeneity because it has more
ways the world can evolve to learn from.

### 4.2 Pretrain-then-finetune from a video prior

Two-stage recipe:

1. **Pretrained video diffusion backbone** (Wan2.1-I2V-14B) supplies
   the visual / physical prior.
2. **Joint video + action training** on DROID adapts the backbone
   to embodied dynamics + action prediction.

No explicit reinforcement learning. The "policy" emerges from the
generative objective.

---

## 5. Headline Results

### 5.1 DROID benchmark (20 seen + 20 unseen-verb tasks)

| Method | Avg task progress | Unseen-verb task progress |
|---|---|---|
| Best pretrained VLA baseline (state-of-the-art) | 27.4% | 25–32% |
| **DreamZero (from scratch)** | **62.2%** | **49%** |

- **Average task progress: 2.27× the best VLA**, despite the VLA
  baselines being pretrained on thousands of hours of cross-
  embodiment robot data.
- **Unseen verb generalization: 49% vs. 25–32%** — this is the
  cleanest test of the "world model → better generalization" claim,
  and DreamZero wins decisively.

### 5.2 Real-time closed-loop control

- **150ms per action chunk** → **~7Hz closed-loop control** with the
  14B model.
- Real-time achievable on **GB200 (Blackwell)**; **H100 lacks the
  throughput** for smooth execution.

The hardware caveat is important: DreamZero is a real-time
controller *on current-generation Blackwell-class GPUs*. The same
recipe is too slow on the previous generation. Inference-side
hardware is now a load-bearing assumption for this class of policy.

### 5.3 Real robot experiments

Across real-world manipulation tasks: **>2× improvement in
generalization to new tasks and environments** vs. SOTA VLAs. The
project page hosts video evidence of the unseen-verb and
unseen-environment evaluations.

---

## 6. What the Method Actually Says

Stripped of acronyms, the structural claim:

> A policy trained on `(obs, action)` pairs has no way to generalize
> to motions it has never seen, because nothing in its training
> objective forces it to model *how the world responds to actions*.
> A policy whose backbone is a video model — trained to predict
> *next frames given proposed actions* — inherits an implicit world
> model, and that world model is what enables physical
> generalization. Real-time closed-loop control is achievable if
> you autoregress the video model and swap generated frames for
> ground-truth observations as they arrive.

This recasts the question "how do we train a better VLA?" into
"what's the *backbone* of a robot policy?" — and the answer
DreamZero ships is: a video world model.

---

## 7. Why This Matters

Three reasons, in increasing order of generality:

1. **A new structural baseline for robot policies.** For two years,
   the default has been "VLA pretrained on demonstration data." If
   the WAM result holds up at scale, the default becomes "video-
   diffusion backbone, jointly trained for video + action." This is
   not a small change — it's a different *kind* of training data
   (internet video) and a different *kind* of objective (joint
   generative).
2. **It validates the video-as-world-model thesis on actual
   embodied control.** Video world-modeling has been a research
   thread (Genie, Sora-as-simulator, OASIS) for a while. DreamZero
   is the first time I've seen the thesis cash out into a *measurable
   policy improvement* on a standardized robot benchmark.
3. **Closed-loop KV swap as a primitive.** The trick of overwriting
   generated frames with ground-truth observations in the KV cache
   is reusable. Any autoregressive generative controller — code
   agents, text agents, browser agents — has an analog: "when the
   environment responds, replace your prediction with the truth and
   keep the rest of the computation." The implementation is
   different in each case; the *principle* is the same. This is one
   of those small mechanical tricks that I expect to show up in a
   dozen unrelated systems over the next year.

The connection to the trend landscape on this site is direct: the
"agent rollouts contain more supervision than the final reward"
argument from
[ECHO]({% link _posts/2026-05-18-echo-terminal-world-models.md %})
is the same instinct DreamZero acts on at the *world-modeling*
layer — environment responses are free supervision, and the system
should be designed to consume them as first-class signal, not
discard them.

---

## 8. Limitations Worth Knowing

- **Hardware-pinned real-time.** ~7Hz at 14B requires Blackwell-
  class GPUs. On H100, smooth real-time execution isn't there.
  Deployments on previous-generation hardware will need
  DreamZero-Flash-style optimization or model-size reductions.
- **DROID-centric evaluation.** The headline numbers are on DROID
  with 20+20 tasks. Whether the generalization pattern extends to
  more diverse embodiments (humanoids, dexterous hands) and
  longer-horizon tasks is the obvious next experiment.
- **No reasoning over plans.** DreamZero is reactive — it predicts
  the next few frames and the next action chunk. It does not do
  multi-step planning over imagined futures. The world model is
  there, but the *planning loop* over it isn't.
- **Backbone dependence.** Results assume a strong pretrained video
  diffusion backbone (Wan2.1-I2V-14B). Whether comparable WAMs can
  be built on smaller / less-trained backbones determines how
  reproducible the recipe is outside large labs.
- **No language grounding stress test.** The unseen-verb benchmark
  is a controlled test of physical generalization. How DreamZero
  handles fully out-of-distribution language (long compositional
  instructions, negations, conditionals) is not the paper's focus.

---

## 9. The Takeaway for a First Reader

If you remember three things:

1. **VLAs fail at *new physical motions* because they're trained as
   policies on `(obs, action)` pairs without ever modeling how the
   world responds.** DreamZero replaces the policy backbone with a
   **video-diffusion world model** and trains it to jointly predict
   next frames + next actions.
2. **A pretrained image-to-video diffusion model (Wan2.1-I2V-14B)
   is the backbone**; an **autoregressive DiT with flow matching**
   makes it usable in closed-loop, and a **KV-cache swap of
   ground-truth observations after each action chunk** eliminates
   compounding drift. **DreamZero-Flash** decouples video/action
   noise schedules for faster sampling.
3. **The result is 2×+ generalization improvement on DROID** vs.
   SOTA VLAs (62.2% vs. 27.4% average task progress; 49% vs.
   25–32% on unseen verbs) with **real-time 7Hz closed-loop control
   on Blackwell-class GPUs**.

That's the arc: identify the structural gap in VLAs → reframe as
joint video+action generation → engineer for real-time closed-loop
control → win the physical generalization test.

---

## References

- Ye, S., Ge, Y., *et al.* (2026). *World Action Models are
  Zero-shot Policies.*
  [arXiv:2602.15922](https://arxiv.org/abs/2602.15922).
- Project page: <https://dreamzero0.github.io/>
- Code: <https://github.com/dreamzero0/dreamzero>
- Checkpoint: <https://huggingface.co/GEAR-Dreams/DreamZero-DROID>
- Backbone:
  Wan2.1-I2V-14B (Alibaba video diffusion model).
- Related on this site:
  [ECHO trend note]({% link _posts/2026-05-18-echo-terminal-world-models.md %}),
  [Parcae paper review]({% link _posts/2026-05-18-parcae.md %}).

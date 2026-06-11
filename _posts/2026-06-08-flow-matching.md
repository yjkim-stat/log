---
layout: post
title: "Flow Matching — The Simulation-Free Recipe Under Modern Diffusion"
date: 2026-06-08 14:00:00 +0900
description: >
  Paper review of Flow Matching for Generative Modeling (Lipman et
  al., ICLR 2023) — the paper that turned Continuous Normalizing
  Flows from "elegant but untrainable" into a simulation-free
  regression problem, introduced optimal-transport conditional
  paths that beat diffusion on FID/NLL/NFE simultaneously, and
  became the training objective under SD3, Wan, and DreamZero.
tags: [generative-models, diffusion, flow-matching, optimal-transport, theory]
categories: paper-review
toc:
  sidebar: left
related_posts: true
slide_deck: /assets/paper_review_html_slides/Flow%20Matching%20%EB%85%BC%EB%AC%B8%20%ED%95%B4%EC%84%A4%20(standalone).html
---

**Paper.** Yaron Lipman, Ricky T. Q. Chen, Heli Ben-Hamu,
Maximilian Nickel, Matt Le.
*Flow Matching for Generative Modeling.* ICLR 2023.
Meta AI · FAIR.
[[arXiv]](https://arxiv.org/abs/2210.02747) ·
[[OpenReview]](https://openreview.net/forum?id=PqvMRDCJT9t)

> 📑 **Companion slide deck.** A self-contained HTML walk-through
> of this review (in Korean) is available at
> [Flow Matching 논문 해설 (standalone).html]({{ page.slide_deck | relative_url }}).
> The deck and this post share the same structure; the post is the
> long-form English version.

---

## 0. The Picture in One Paragraph

Continuous Normalizing Flows (CNFs) are an *elegant* generative
model — sample $x_0$ from a simple prior, integrate an ODE
$\dot{x} = v_\theta(t, x)$, get a sample from a complicated target.
They've been theoretically attractive for years and practically
unusable, because the standard training objective (maximum
likelihood) requires solving the ODE forward and backward on every
training step. That's the "simulation cost." **Flow Matching**
proposes a different objective: instead of training to maximize
likelihood, **directly regress the vector field $v_\theta$ against
a target vector field that produces a chosen probability path**.
The catch is that the marginal target vector field is intractable.
The contribution is showing that you can construct it as a mixture
of *per-sample conditional* vector fields, and the **conditional
flow matching (CFM) loss** — which only ever sees the conditional
field — has the **same gradient as the intractable marginal loss**.
This eliminates simulation entirely. With **optimal-transport
conditional paths** (straight-line interpolation between noise and
data), CNFs trained this way **outperform DDPM on CIFAR-10 and
ImageNet on FID, NLL, *and* NFE simultaneously** — the first time
those three knobs moved together. The recipe is now the training
objective under Stable Diffusion 3, Wan, and DreamZero.

---

## 1. The Background — Why CNFs Were Stuck

A CNF defines a generative model via a learned vector field
$v_\theta: [0,1] \times \mathbb{R}^d \to \mathbb{R}^d$ acting through
the ODE

$$
\frac{dx(t)}{dt} = v_\theta(t, x(t)), \qquad x(0) \sim p_0.
$$

The flow $\phi_t$ pushes the prior $p_0$ forward to a model
distribution $p_t$. Training maximizes the log-likelihood under
the change-of-variables formula:

$$
\log p_1(x) = \log p_0(\phi_1^{-1}(x)) - \int_0^1 \nabla \cdot v_\theta(t, \phi_t^{-1}(x)) \, dt.
$$

This is *gorgeous* — exact likelihood, single network — and
*expensive*: each gradient step needs an ODE solve for every sample,
and gradients flow through the solver. Scaling CNFs to ImageNet
was a known open problem precisely because of this simulation tax.

---

## 2. The Move — Regress the Vector Field Directly

Flow Matching's reframing is to *give up on maximum likelihood as
the training signal* and instead specify a target by saying what
**probability path** $p_t$ you want the model to follow, then
**regress the model's vector field against the field that produces
that path**.

### 2.1 The (intractable) Flow Matching loss

Fix a probability path $p_t$ that interpolates $p_0$ (noise) to
$p_1$ (data). Suppose we know the vector field $u_t$ that generates
$p_t$ (in the sense that pushing $p_0$ along $u_t$ yields $p_t$).
Then the natural loss is

$$
\mathcal{L}_{\text{FM}}(\theta) = \mathbb{E}_{t, x \sim p_t} \big\| v_\theta(t, x) - u_t(x) \big\|^2.
$$

A clean MSE between two vector fields. **Simulation-free**: no ODE
solve, no divergence integral. If we had $u_t$, we'd be done.

The problem: $u_t$ is **intractable** for non-trivial paths. We
can't write it down in closed form for the marginal path; that's
exactly the thing the model is supposed to learn.

### 2.2 The CFM trick — replace the marginal with the conditional

Here's the move that makes everything work. Construct $p_t$ as a
**mixture of conditional paths**:

$$
p_t(x) = \int p_t(x \mid x_1) \, q(x_1) \, dx_1,
$$

where $x_1 \sim q$ is a data sample and $p_t(\cdot \mid x_1)$ is a
**simple conditional path** from the prior to a point mass (or
near-point-mass) at $x_1$. For the conditional path, the generating
vector field $u_t(x \mid x_1)$ is *tractable in closed form* —
it's just the field that smoothly transports prior samples to $x_1$.

Now define the **Conditional Flow Matching** loss:

$$
\mathcal{L}_{\text{CFM}}(\theta) = \mathbb{E}_{t, x_1 \sim q, x \sim p_t(\cdot \mid x_1)} \big\| v_\theta(t, x) - u_t(x \mid x_1) \big\|^2.
$$

The model now regresses against a *conditional* field — which we
have a closed form for — instead of the marginal field — which we
don't.

### 2.3 The theorem that makes this work

The paper's central technical result:

> $\nabla_\theta \mathcal{L}_{\text{FM}}(\theta) = \nabla_\theta \mathcal{L}_{\text{CFM}}(\theta).$

The two losses are **not equal** (they differ by a constant that
depends only on the data and the path), but their gradients are
**identical**. Optimizing the tractable CFM loss is equivalent to
optimizing the intractable FM loss.

The proof uses the fact that the marginal vector field
$u_t(x)$ is the **posterior expectation** of the conditional fields:

$$
u_t(x) = \mathbb{E}_{q(x_1 \mid x_t = x)} \big[ u_t(x \mid x_1) \big],
$$

i.e. the marginal field at $x$ is a weighted average of the
conditional fields toward each possible data point, weighted by how
likely each data point is to have produced $x$ at time $t$. The
MSE regression to a marginal mean is equivalent (in gradient) to
regression to the per-sample target.

This is the same structure that underlies **denoising score
matching** — and the connection is not accidental. Flow Matching
**generalizes** score matching: with a specific choice of
conditional path (the variance-exploding or variance-preserving
diffusion path) and a specific reparameterization, CFM reduces to
the standard score-matching loss. *Diffusion is one instance of
Flow Matching*, not the other way around.

---

## 3. Optimal-Transport Conditional Paths

Now the design freedom: which conditional path do we use? FM works
for any path, but the paper makes a specific recommendation that
turns out to dominate.

### 3.1 The OT conditional path

Pick the conditional path that **linearly interpolates** between a
prior sample and a data sample:

$$
\mu_t(x_1) = t \, x_1, \qquad \sigma_t(x_1) = 1 - (1 - \sigma_{\min}) t,
$$

i.e. $x \sim \mathcal{N}(t x_1, (1 - (1-\sigma_{\min}) t)^2 I)$.
At $t=0$, you're at the standard Gaussian prior; at $t=1$, you're
at $x_1$ with a tiny residual variance $\sigma_{\min}$.

The associated conditional vector field is

$$
u_t(x \mid x_1) = \frac{x_1 - (1 - \sigma_{\min}) x}{1 - (1 - \sigma_{\min}) t},
$$

which — at $\sigma_{\min} \to 0$ — is the **constant velocity**
$x_1 - x_0$ along a **straight line** from prior sample $x_0$ to
data sample $x_1$.

This is the **displacement interpolant** from optimal transport,
restricted to per-sample conditionals.

### 3.2 Why straight lines win

Diffusion conditional paths are **curved** — they push the sample
toward $x_1$ along a noise-scaled trajectory that bends through
high-noise regions before settling. Curved trajectories require
more ODE-solver steps to integrate accurately at inference time.

OT paths are **straight**. A straight integral curve can be
integrated with very few steps — in the limit of an ideal model,
*one step*. Empirically the gain is dramatic: same FID, fraction
of the function evaluations.

### 3.3 What FM-OT inherits

The OT path keeps every good property of the diffusion path while
shortening the integration:

- Closed-form conditional field (no learning $u_t$).
- Closed-form sampling from $p_t(x \mid x_1)$ (just a Gaussian).
- Straight integration → low NFE.

Nothing about the framework changes when you swap the conditional
path; only the per-sample $u_t$ formula changes. The CFM gradient-
equivalence theorem still holds.

---

## 4. Headline Results

All numbers are training the same U-Net architecture with the same
budget on the same hyperparameters; the **only** variable is the
training objective. Lower is better on all three metrics.

### 4.1 CIFAR-10

| Method | NLL ↓ | FID ↓ | NFE ↓ |
|---|---|---|---|
| DDPM | 3.12 | 7.48 | 274 |
| FM (diffusion path) | 3.10 | 8.06 | 183 |
| **FM-OT** | **2.99** | **6.35** | **142** |

### 4.2 ImageNet 32×32

| Method | NLL ↓ | FID ↓ | NFE ↓ |
|---|---|---|---|
| DDPM | 3.54 | 6.99 | 262 |
| **FM-OT** | **3.53** | **5.02** | **122** |

### 4.3 ImageNet 64×64

| Method | NLL ↓ | FID ↓ | NFE ↓ |
|---|---|---|---|
| DDPM | 3.32 | 17.36 | 264 |
| **FM-OT** | **3.31** | **14.45** | **138** |

### 4.4 ImageNet 128×128

FM-OT: NLL **2.90**, FID **20.9** — first time a CNF result is on
this scale.

### 4.5 What's striking about the table

Generative-model literature typically improves *one* of (FID, NLL,
NFE) by hurting the others — better quality usually costs more
function evaluations; better likelihood usually costs visual
quality. **FM-OT improves all three at once** on every dataset.
That kind of clean dominance is rare and is the reason this paper
landed as hard as it did.

---

## 5. What the Method Actually Says

Stripped of acronyms, the structural claim:

> 1. Continuous Normalizing Flows are a useful generative model
>    class, and the obstacle to using them was the simulation cost
>    of maximum-likelihood training.
> 2. You can replace likelihood with **vector-field regression**.
>    The marginal target is intractable, but if you express the
>    target path as a mixture of *per-sample conditional* paths,
>    the conditional regression has the *same gradient* as the
>    marginal regression.
> 3. The choice of conditional path is a design knob.
>    **Straight-line (optimal-transport) paths** are dramatically
>    more efficient at inference time than the curved paths
>    implicit in diffusion training.
> 4. Diffusion training is one instance of this framework, not the
>    other way around. The framework gives you a strictly larger
>    design space, and within it FM-OT is a sweet spot.

This is the kind of result that doesn't just improve a benchmark —
it reorganizes the field around a different abstraction.

---

## 6. Why It Matters — Three Years Later

Reading this paper in 2026, with the benefit of hindsight, three
things stand out:

1. **It became the training objective under most modern generative
   models.** Stable Diffusion 3, Wan, Cosmos, and the
   [DreamZero]({% link _posts/2026-06-08-dreamzero.md %})
   backbone all train with flow matching (often with OT paths)
   rather than the original score-matching diffusion objective.
   What looked like a clever theoretical paper in 2022 became
   *the* default by 2024.
2. **The CFM trick is reusable.** Any time you want to regress a
   network against an intractable marginal target that's a mixture
   of tractable conditionals — molecule generation, video
   synthesis, robot action chunks — the same gradient-equivalence
   argument applies. The paper effectively opened a generic
   recipe.
3. **It exposed a knob the field didn't know it had.** Before this
   paper, "what probability path do we use?" wasn't a question
   you saw in diffusion papers — the variance-preserving and
   variance-exploding paths were treated as *given*. Flow Matching
   made the path itself a design choice, which is what unlocked
   straight-line OT paths and, later, more exotic choices
   (rectified flows, consistency flow matching, etc.).

---

## 7. Limitations Worth Knowing

- **Theory is asymptotic.** The gradient-equivalence theorem holds
  in expectation over infinitely many samples. At finite batch
  size, CFM has higher variance than directly fitting the marginal
  field would (if we could). In practice this is fine, but it's
  worth knowing the theorem is not a free lunch.
- **OT path is *displacement* OT, not a true Monge optimal map.**
  The "optimal transport" path is per-sample straight-line
  interpolation between a noise sample and a data sample — *which
  noise sample pairs with which data sample is random*. True OT
  would pair them to minimize transport cost, which the original
  paper does not do. The follow-up *minibatch OT* work
  ([Tong et al., 2023](https://arxiv.org/abs/2302.00482)) closes
  that gap.
- **No conditional generation in the original.** The paper covers
  unconditional generative modeling. Class- or text-conditioned FM
  required follow-up engineering; Stable Diffusion 3 and Wan are
  the modern conditional implementations.
- **Architecture not the contribution.** The U-Net used is the
  standard one from the diffusion literature. The contribution is
  the *training objective*; the architecture is a control variable.
  This is also why the recipe transferred so cleanly to other
  backbones (DiT, transformer-based video models) later.

---

## 8. The Takeaway for a First Reader

If you remember three things:

1. **CNFs are simulation-free trainable** if you regress the
   model's vector field against a target field instead of
   maximizing likelihood. The target field is intractable in the
   marginal but tractable in the conditional, and the
   **conditional MSE has the same gradient as the marginal MSE**.
2. **The conditional path is a design choice.** **OT
   (straight-line) paths** dominate diffusion paths because
   straight curves integrate cheaply at inference time.
3. **The result is the first generative training recipe that wins
   on FID, NLL, *and* NFE simultaneously** on CIFAR-10 and
   ImageNet, and the recipe became the default training objective
   under Stable Diffusion 3, Wan, and modern video / world models.

That's the arc: maximum-likelihood CNFs were stuck → reframe
training as vector-field regression → use the conditional trick to
make the regression tractable → use OT paths to make inference
cheap → end up with the substrate under most modern generative
modeling.

---

## References

- Lipman, Y., Chen, R. T. Q., Ben-Hamu, H., Nickel, M., & Le, M.
  (2023). *Flow Matching for Generative Modeling.* ICLR 2023.
  [arXiv:2210.02747](https://arxiv.org/abs/2210.02747).
- Tong, A. *et al.* (2023). *Improving and generalizing flow-based
  generative models with minibatch optimal transport.*
  [arXiv:2302.00482](https://arxiv.org/abs/2302.00482).
- Background: DDPM (Ho et al., 2020); Score-based generative
  modeling (Song et al., 2021).
- Related on this site:
  [DreamZero paper review]({% link _posts/2026-06-08-dreamzero.md %})
  — the world-action-model backbone that depends on this objective;
  [Parcae paper review]({% link _posts/2026-05-18-parcae.md %})
  — for the discretization tricks Parcae borrows from SSMs and FM
  shares structural family with.

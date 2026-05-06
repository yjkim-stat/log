---
layout: post
title: "Direct Preference Optimization: A Closer Look"
date: 2026-05-06 10:00:00 +0900
description: A close read of DPO — its derivation from RLHF, what assumptions it inherits, and where empirical results begin to fray.
tags: [llm, rlhf, alignment, optimization]
categories: paper-review
related_posts: true
toc:
  sidebar: left
related_publications: true
pretty_table: true
featured: false
---

Direct Preference Optimization (DPO) {% cite rafailov2023direct %} reframes
RLHF as a classification problem on preference pairs, sidestepping the
reward-model-plus-PPO {% cite schulman2017proximal %} pipeline that
InstructGPT-style training popularized {% cite ouyang2022training %}.
This post walks through the derivation, the implicit assumptions, and the
places where DPO behaves differently from a reward-model-based pipeline in
practice.

## Setup: the RLHF objective

The standard RLHF objective for a policy $$\pi_\theta$$ given a reference
policy $$\pi_\text{ref}$$ and a reward $$r$$ is

$$
\max_\theta \; \mathbb{E}_{x \sim \mathcal{D},\, y \sim \pi_\theta(\cdot \mid x)}
\big[ r(x, y) \big] - \beta\, \mathrm{KL}\!\left( \pi_\theta \,\|\, \pi_\text{ref} \right).
$$

The closed-form maximizer of this objective is the well-known reweighting

$$
\pi^*(y \mid x) \;\propto\; \pi_\text{ref}(y \mid x) \exp\!\left( \frac{1}{\beta}\, r(x, y) \right).
$$

## DPO's reparameterization

DPO inverts the closed-form maximizer to express the reward in terms of the
policy itself:

$$
r(x, y) \;=\; \beta \log \frac{\pi_\theta(y \mid x)}{\pi_\text{ref}(y \mid x)} + \beta \log Z(x).
$$

Plug this into a Bradley–Terry preference model and the partition function
$$Z(x)$$ cancels across the chosen/rejected pair, leaving the DPO loss

$$
\mathcal{L}_\text{DPO}(\theta) = -\,\mathbb{E}_{(x, y_w, y_l) \sim \mathcal{D}}
\Big[ \log \sigma \big( \beta\, \Delta_\theta(x, y_w, y_l) \big) \Big],
$$

where $$\Delta_\theta(x, y_w, y_l) = \log \tfrac{\pi_\theta(y_w \mid x)}{\pi_\text{ref}(y_w \mid x)} - \log \tfrac{\pi_\theta(y_l \mid x)}{\pi_\text{ref}(y_l \mid x)}$$.

## What DPO assumes (that the press release skips)

1. **The Bradley–Terry model is the right preference model.** Annotator
   noise, ties, and within-annotator inconsistency are all collapsed into a
   single logistic.
2. **The optimal policy lies in the family $$\pi_\theta$$.** When the
   reference is far from optimal, the implicit reward DPO recovers may
   under- or over-weight rare tokens.
3. **No on-policy correction.** DPO trains on offline preference data; the
   policy never explores. This is why pairing DPO with iterated rounds of
   on-policy preference collection (à la {% cite bai2022training %}) tends
   to outperform a single offline pass.

## Empirical takeaways

| Setting | DPO | PPO + RM |
|---|---|---|
| Stable to train? | Yes (one loss, one model) | Sensitive to RM quality, KL coefficient |
| On-policy gains | Limited unless iterated | Built-in |
| Compute | ~SFT-level | RM training + RL rollouts |

In practice, DPO is the right starting point for most preference-tuning
projects, but the gap closes — and sometimes reverses — once you can afford
on-policy data collection or once the preference distribution shifts away
from the reference policy.

## What I want to see next

- A clean theoretical account of when DPO underperforms PPO at fixed data
  budget, beyond the standard "off-policy vs on-policy" intuition.
- Better diagnostics for the implicit reward $$r_\theta$$ DPO is fitting —
  most papers report win rates without inspecting whether the recovered
  reward agrees with the explicit reward an RM would learn from the same
  data.

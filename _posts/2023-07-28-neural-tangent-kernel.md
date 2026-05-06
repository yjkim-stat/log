---
layout: post
title: "Neural Tangent Kernel: Infinite-Width Networks as Kernel Methods"
date: 2023-07-28 01:00:00 +0900
description: How wide neural networks behave as kernel regression — the NTK framework.
tags: [theory]
categories: paper-review
related_posts: true
---

## Setup

The **Neural Tangent Kernel (NTK)** framework (Jacot, Gabriel, Hongler, 2018) provides a striking analytical result: in the infinite-width limit, training a neural network with gradient descent is equivalent to **kernel regression** with a fixed kernel — the NTK.

For a network $f(x; \theta)$, the NTK is

$$K_\text{NTK}(x, x') = \mathbb{E}_\theta\!\left[\langle \nabla_\theta f(x; \theta), \nabla_\theta f(x'; \theta) \rangle\right]$$

In the infinite-width limit and with appropriate parameterization:
1. The NTK at initialization concentrates around its expectation
2. The NTK **does not change** during training
3. Training dynamics become *linear* in the parameters

## Implications

- **Convergence**: gradient descent provably converges to a global minimum at a rate determined by NTK eigenvalues
- **Generalization**: the bias of NTK regression characterizes which functions wide networks prefer — a form of *implicit bias*
- **Lazy training**: the regime where NTK applies is sometimes called "lazy" because parameters move infinitesimally relative to their initial values

## Limitations

NTK is exact only in the infinite-width limit; finite networks deviate, and crucially, the NTK assumption misses **feature learning**. Real networks update their internal representations during training in ways that lazy-regime theory cannot capture — which is precisely what makes deep learning powerful.

This has spawned active research on **mean-field** and **feature-learning** regimes that go beyond NTK.

## Further reading

- [CMU ML Blog — Ultra-Wide Deep Nets and NTK](https://blog.ml.cmu.edu/2019/10/03/ultra-wide-deep-nets-and-the-neural-tangent-kernel-ntk/)
- Jacot, Gabriel, Hongler (2018), *Neural Tangent Kernel: Convergence and Generalization in Neural Networks*

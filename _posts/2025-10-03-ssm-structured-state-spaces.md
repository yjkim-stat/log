---
layout: post
title: "SSM Part I: Modeling Sequences with Structured State Spaces"
date: 2025-10-03 00:00:00 +0900
description: Study notes on structured state space models (S4) — recurrent, convolutional, and continuous views unified.
tags: [ssm]
categories: notes
toc:
  sidebar: left
related_posts: true
---

## Sequence Models

A sequence model is a parameterized mapping where inputs and outputs are sequences of feature vectors, with parameters learned by gradient descent. Different architectures excel in different regimes:
- **RNNs** for stateful sequential processing
- **CNNs** for perceptual signals with local structure
- **Transformers** for complex long-range interactions
- **Neural Differential Equations** for irregular time series

Key challenges: efficient training over full sequences, efficient inference at single timesteps, and modeling long-range dependencies without vanishing gradients.

## State Space Models (SSMs)

SSMs offer a unified framework that is **continuous, recurrent, and convolutional** simultaneously. The continuous-time formulation:

$$x'(t) = A x(t) + B u(t), \quad y(t) = C x(t) + D u(t)$$

By choosing the right discretization, the same model becomes:
- A **recurrent** model for sequential inference
- A **convolutional** model for parallel training (via the State Space Kernel)

## Structured SSMs (S4)

Naive SSMs require $O(N^2 L)$ operations and $O(N L)$ space — prohibitive for long sequences. **S4** achieves $O(N + L)$ complexity by structuring the state matrix.

### Long-Range Dependencies via HIPPO

The **HIPPO** (High-Order Polynomial Projection Operator) framework treats sequence modeling as **online function approximation** — the hidden state encodes a polynomial approximation of the input history. This gives principled handling of long-range dependencies.

### Efficient Computation

Three discretization options bridge continuous and discrete domains:
- **Euler's method**: simple but unstable
- **Bilinear**: standard for SSMs
- **Zero-Order Hold**: used in Mamba

## Diagonal Plus Low-Rank (DPLR)

To overcome computational bottlenecks, S4 uses the **DPLR structure**: $A = \Lambda + PQ^*$. Three techniques exploit this:

1. **Kernel Generating Functions**: compute outputs in frequency domain via FFT, avoiding explicit kernel materialization
2. **Woodbury Identity**: efficient matrix inversions for the low-rank correction
3. **Cauchy Matrices**: reduce complexity to $O((M+N)\log(M+N))$

## Numerical Stability

**Hurwitz matrices** (eigenvalues with negative real parts) ensure stable dynamics. The **Hurwitz DPLR form** $A := \Lambda - PP^*$ guarantees numerical stability while preserving the computational structure.

## Takeaways

SSMs provide a principled and efficient alternative to attention for long-range modeling. The continuous → discrete bridge, the convolutional ↔ recurrent equivalence, and the structured matrix algebra all work together to make linear-time sequence modeling practical.

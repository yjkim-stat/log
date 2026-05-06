---
layout: post
title: "Neural Collapse: Terminal Phase of Deep Network Training"
date: 2023-02-02 00:00:00 +0900
description: The four manifestations of neural collapse and what they reveal about deep learning's implicit biases.
tags: [theory]
categories: paper-review
toc:
  sidebar: left
related_posts: true
---

## Phenomenon

In the **terminal phase** of training (TPT) — long after training error reaches zero — deep classifiers exhibit a remarkable geometric structure dubbed **Neural Collapse**, characterized by four interconnected properties:

### NC1 — Variability Collapse

Within-class variation in the penultimate-layer features collapses: all activations from class $k$ converge to their class mean $\mu_k$.

$$\Sigma_W \to 0$$

### NC2 — Convergence to Simplex ETF

Class mean vectors organize into a **simplex equiangular tight frame**:
- Equal lengths
- Equal pairwise angles ($\cos\theta = -1/(K-1)$ for $K$ classes)

This is the most symmetric arrangement possible — maximally separated, equiangular.

### NC3 — Self-Duality

Classifier weights and class means converge to each other (up to scale):
$$W_k \propto \mu_k$$

The classifier becomes "dual" to the feature representation.

### NC4 — Nearest Class Center Classification

The trained network's decision rule simplifies to: assign each input to the class whose mean is nearest in Euclidean distance. The deep classifier becomes equivalent to a simple template-matching rule on its features.

## Why This Happens

The reviewer's interpretation: **cross-entropy loss contains an implicit inductive bias** favoring these geometric arrangements. The softmax's reliance on dot products encourages alignment between classifiers and features, reducing angular distances and driving the simplex ETF structure.

## Limitations

The original analysis focuses on:
- Primitive models (often single-layer classifiers on top of features)
- Balanced datasets
- Standard cross-entropy loss

Open questions remain about whether and how these regularities scale to deeper networks, imbalanced data, and more complex losses.

## Significance

Neural collapse suggests that deep learning's success is partly explained by the **implicit constraints of the training objective**, not just architectural inductive biases. The geometry of the terminal phase is more constrained than commonly recognized.

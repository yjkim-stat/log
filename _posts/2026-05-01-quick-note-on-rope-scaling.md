---
layout: post
title: "Quick note on RoPE scaling"
date: 2026-05-01 22:00:00 +0900
description: A short observation while reading recent position-embedding tricks for long-context models.
tags: [llm, position-embeddings]
categories: notes
toc: false
related_publications: true
---

Reading through the various RoPE {% cite su2024roformer %} extension tricks
(NTK-aware scaling, YaRN, position interpolation), one detail kept tripping
me up: most of the tricks are not really about *positions* — they're about
keeping the *frequency content* of the rotary basis inside the regime the
model was trained on.

If $$\theta_i = b^{-2i/d}$$ is the base frequency for the $$i$$-th pair of
dimensions, then doubling the context length doubles the largest argument
$$n \theta_i$$ that the attention layer ever sees at the high-frequency end.
The model has never seen those phases at training time, so it generalizes
poorly. Position interpolation rescales $$n$$ down; NTK-aware scaling
rescales $$b$$ instead, with the explicit goal of leaving the high-frequency
dimensions alone (since those are the ones whose phase patterns the model
*has* observed) and stretching only the low-frequency dimensions where
extrapolation is well-behaved.

This reframing also makes it obvious why fine-tuning helps: fine-tuning lets
the model learn the new high-frequency phase patterns, at which point you
no longer need any scaling trick at all. The scaling tricks are just a
zero-shot patch.

Worth re-reading the original RoPE paper with this lens.

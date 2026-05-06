---
layout: post
title: "Scaling Laws for Neural Language Models"
date: 2023-09-03 00:00:00 +0900
description: Kaplan et al. (2020) — power-law relationships between model size, compute, data, and loss in language models.
tags: [llm, theory]
categories: paper-review
related_posts: true
---

Kaplan et al. identify smooth power-law relationships governing language model performance:

$$L(N) \propto N^{-\alpha_N}, \quad L(C) \propto C^{-\alpha_C}, \quad L(D) \propto D^{-\alpha_D}$$

where $N$ = model parameters, $C$ = training compute, $D$ = dataset size.

Key findings:
- Performance depends most strongly on scale, not on architectural details (depth vs. width)
- Model size is the dominant factor for a fixed compute budget
- Data and compute requirements scale together: undertrained models waste compute
- The "compute-optimal" model size given a fixed FLOP budget is larger than practitioners typically train

These results were later refined by Chinchilla (Hoffmann et al., 2022), which showed the original paper *underweighted* data relative to parameters — optimal scaling allocates roughly equal compute to parameters and tokens.

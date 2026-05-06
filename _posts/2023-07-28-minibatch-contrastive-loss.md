---
layout: post
title: "Mini-batch Optimization of Contrastive Loss"
date: 2023-07-28 00:00:00 +0900
description: Sample complexity and optimization dynamics of contrastive loss under mini-batch training.
tags: [representation-learning]
categories: paper-review
related_posts: true
---

Contrastive learning relies on contrasting positive pairs against many negatives — and most practical implementations use **mini-batch negatives** rather than the full dataset. This paper analyzes the optimization and sample-complexity properties of the resulting mini-batch contrastive loss.

Core questions addressed:
- How does mini-batch size affect convergence?
- What is the bias introduced by replacing the full negative distribution with a mini-batch sample?
- How does the gradient signal scale with batch size and dataset size?

The analysis provides theoretical justification for the empirical observation that **larger batch sizes improve contrastive learning** — a property that distinguishes it from supervised loss training, where batch-size effects are more subtle.

Practical takeaways:
- Larger batches reduce the variance of the negative-distribution estimate
- The effective sample complexity has a clean dependence on batch size
- Memory bank and momentum encoder tricks (MoCo, etc.) can be understood as approximating large-batch behavior at small actual batch sizes

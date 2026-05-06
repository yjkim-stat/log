---
layout: post
title: "Sliced Mutual Information for Memorization and Generalization"
date: 2023-12-12 01:00:00 +0900
description: Using sliced mutual information as a tractable analytical framework for studying neural network memorization vs. generalization.
tags: [information-theory, theory]
categories: paper-review
related_posts: true
---

Standard mutual information is notoriously difficult to estimate in high-dimensional neural network representations. **Sliced mutual information (SMI)** offers a tractable alternative by averaging mutual information over random one-dimensional projections — analogous to sliced Wasserstein distance.

This paper applies SMI to study **memorization vs. generalization** in neural networks:
- Provides a measurable quantity for "how much information about training data is retained in representations"
- Tracks how memorization evolves through training
- Distinguishes representations that memorize specific examples from those encoding generalizable patterns

The work sits at the intersection of information theory and deep learning theory, contributing a practical tool for the long-running question of *why* deep networks generalize despite enormous capacity.

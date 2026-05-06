---
layout: post
title: "Singular Value Representation: A Graph Perspective on Neural Networks"
date: 2023-07-10 00:00:00 +0900
description: Recasting neural network analysis through singular value decomposition and graph-theoretic structure.
tags: [theory]
categories: paper-review
related_posts: true
---

This paper develops a **graph-theoretic perspective** on neural networks built on the singular value decomposition (SVD) of weight matrices.

The core idea: each layer's weight matrix admits an SVD, and the right and left singular vectors form a graph connecting the input and output bases. Properties of this graph (sparsity, connectivity, spectral structure) reflect what the layer *does* in terms of mixing input dimensions — providing a more interpretable lens than treating weights as a black box.

This perspective enables:
- Diagnosing redundancy via low-rank structure
- Tracking how representations transform across layers
- Connecting initialization, training dynamics, and final structure through spectral evolution

The framework offers a principled tool for analyzing trained networks rather than just measuring their performance.

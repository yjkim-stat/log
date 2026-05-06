---
layout: post
title: "Attention Is All You Need"
date: 2022-02-27 00:00:00 +0900
description: Review of the Transformer paper — self-attention mechanism, encoder-decoder architecture, and why it replaced RNNs.
tags: [nlp, llm, attention]
categories: paper-review
toc:
  sidebar: left
related_posts: true
---

## Why Self-Attention?

RNNs and Seq2Seq models struggle with three issues: sequential computation prevents parallelism, long-range dependencies suffer from gradient problems, and computational cost scales poorly. The Transformer resolves all three by replacing recurrence with self-attention.

Attention formula: $\text{Attention}(Q, K, V) = \text{softmax}\!\left(\frac{QK^\top}{\sqrt{d_k}}\right)V$

The $\sqrt{d_k}$ scaling prevents softmax from saturating when the dot products grow large with dimensionality.

## Architecture

**Encoder**: 6 stacked layers, each with multi-head self-attention (attending to all positions) + position-wise FFN + residual + LayerNorm.

**Decoder**: Same stacked structure, but self-attention is *causal* (masked to prevent attending to future tokens), plus cross-attention where decoder queries attend to encoder outputs.

## Multi-Head Attention

Rather than a single attention operation, project embeddings into $h$ subspaces, run attention independently, and concatenate:

$$\text{MultiHead}(Q,K,V) = \text{Concat}(\text{head}_1, \ldots, \text{head}_h)W^O$$

This allows the model to jointly attend to information from different representation subspaces — effectively an ensemble of attention patterns.

## Positional Encoding

Since attention is permutation-equivariant, absolute position must be injected explicitly. Sinusoidal encodings $\sin(\text{pos}/10000^{2i/d})$ and $\cos(\ldots)$ vary across dimensions and allow the model to attend to relative positions via linear combinations.

## Takeaways

- O(1) path length between any two positions → better long-range dependency capture
- Fully parallelizable training
- Multi-head attention as ensemble of relationship types

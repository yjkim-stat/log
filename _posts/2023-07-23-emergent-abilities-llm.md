---
layout: post
title: "Emergent Abilities of Large Language Models"
date: 2023-07-23 00:00:00 +0900
description: Capabilities that appear discontinuously at scale — and the debate over whether emergence is a real phenomenon or measurement artifact.
tags: [llm, theory]
categories: paper-review
related_posts: true
---

This paper documents **emergent abilities** in LLMs: capabilities that are not present in smaller models but appear sharply once a critical scale is reached, often with little improvement before that threshold.

Examples documented:
- Multi-digit arithmetic
- Multi-step word problems
- Following complex chained instructions
- Programmatic transformations

The argument: scaling is not just a smooth quantitative improvement; some capabilities exhibit **phase transitions** as a function of compute or parameters.

## Subsequent Debate

A follow-up paper ([Schaeffer et al., NeurIPS 2023](https://arxiv.org/abs/2304.15004)) argued emergence may be partly a **measurement artifact**: discontinuous metrics (e.g., exact-match accuracy) hide smooth underlying improvements that would be visible under continuous metrics (e.g., per-token log-likelihood).

The empirical question is unresolved: even with smoother metrics, qualitative shifts in *what tasks the model can do at all* remain — but the precise role of metric choice in the appearance of emergence is now part of the discussion.

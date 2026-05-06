---
layout: post
title: "Calibration, Entropy Rates, and Memory in Language Models"
date: 2023-07-16 00:00:00 +0900
description: Why perplexity fails to capture long-term properties — entropy rate calibration and mutual-information memory.
tags: [information-theory, nlp]
categories: paper-review
toc:
  sidebar: left
related_posts: true
---

## Gap in Standard Metrics

Prior work on long-term properties of language models focused on architectural improvements. This paper identifies a more fundamental issue: standard metrics like **perplexity and cross-entropy loss do not actually reflect long-term properties** of generated text.

## Entropy Rate Calibration

The authors propose using **entropy rate** for calibration. The key result (Theorem 4.4): calibration improves both:
- Conventional training outcomes (lower perplexity)
- Long-term dependency modeling

Experimental validation (Figure 2) shows that entropy-rate calibration *simultaneously* improves perplexity, demonstrating that the two objectives are not in conflict — better long-term modeling and better short-term modeling go together.

## Memory via Mutual Information

The paper formalizes **memory** in language models using mutual information between a generated token and earlier context tokens at varying distances.

Empirical finding: during generation, language models rely heavily on **recent context** and progressively less on distant tokens — even with sufficient context window, the effective memory horizon is shorter than the architectural maximum.

## Significance

The work bridges information theory and language modeling: it provides a principled metric for "long-term behavior" beyond perplexity, and a quantitative tool for analyzing how much of the context window LMs actually use.

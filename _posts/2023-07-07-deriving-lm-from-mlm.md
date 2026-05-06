---
layout: post
title: "Deriving Language Models from Masked Language Models"
date: 2023-07-07 00:00:00 +0900
description: Computing valid joint distributions from MLMs via Markov Random Fields, with regularization to relax conditional independence.
tags: [information-theory, nlp]
categories: paper-review
related_posts: true
---

MLMs like BERT define **conditional** distributions $p(x_i \mid x_{\setminus i})$, but these conditionals do not generally correspond to a valid joint distribution — extracting a coherent language model from an MLM requires care.

This paper compares methods for constructing joint distributions from MLM conditionals:
- **Markov Random Field (MRF)** factorization, treating the unmasked tokens as cliques
- Pseudo-likelihood approximations
- Other consistency-enforcing methods

Evaluated through metrics like P-PPL (pseudo-perplexity) and U-PPL (unigram-conditioned perplexity), the analysis reveals tradeoffs in how each method handles the **conditional independence assumptions** baked into the MLM.

The paper proposes regularization techniques to relax these assumptions, sitting at the intersection of information theory, stochastic processes, and probabilistic modeling — a useful theoretical lens on why MLMs are powerful but not "true" language models.

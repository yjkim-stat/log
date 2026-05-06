---
layout: post
title: "Auto-Regressive Next-Token Predictors Are Universal Learners"
date: 2023-10-10 00:00:00 +0900
description: Theoretical result showing that next-token prediction is a universal learning objective for sequence functions.
tags: [llm, theory]
categories: paper-review
related_posts: true
---

This paper provides a theoretical justification for why next-token prediction (the pre-training objective of GPT-style models) is so general: it proves that autoregressive next-token predictors are **universal learners** in a formal sense — they can approximate any computable function on sequences given sufficient capacity and data.

The result connects the empirical success of LLMs to a clean theoretical guarantee: the objective is not accidentally powerful, but fundamentally capable of learning arbitrary sequential mappings. This also provides a theoretical framing for in-context learning as implicit Bayesian inference.

---
layout: post
title: "Frequency Effects on Syntactic Rule Learning in Transformers"
date: 2023-07-25 00:00:00 +0900
description: ACL 2021 — how token frequency in training data shapes syntactic generalization in Transformers.
tags: [nlp, theory]
categories: paper-review
related_posts: true
---

This ACL 2021 paper investigates how the *frequency* of tokens and constructions in pre-training data shapes the syntactic rules that Transformers learn to generalize. The central finding is that frequency has a non-trivial effect on syntactic generalization: models trained on natural language distributions may learn frequency-based heuristics rather than genuine syntactic rules, complicating interpretations of "grammatical knowledge" in language models.

Understanding frequency effects is important for designing pre-training corpora and interpreting probing experiments.

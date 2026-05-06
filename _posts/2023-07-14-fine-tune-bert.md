---
layout: post
title: "A Closer Look at How Fine-tuning Changes BERT"
date: 2023-07-14 00:00:00 +0900
description: ACL 2022 — empirical analysis of what changes in BERT's internal representations during fine-tuning.
tags: [nlp, llm]
categories: paper-review
related_posts: true
---

This ACL 2022 paper systematically examines how fine-tuning modifies pre-trained BERT across three dimensions: *performance outcomes*, *internal representations*, and *layer-specific behavior*.

Key findings:
- Fine-tuning adapts higher layers more dramatically than lower layers
- Task-specific information emerges primarily in upper layers
- The pre-trained representations provide a strong scaffold that fine-tuning reshapes incrementally rather than overwriting

Understanding these dynamics is important for deciding how many layers to fine-tune and for explaining transfer learning success.

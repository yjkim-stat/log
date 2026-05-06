---
layout: post
title: "Graph of Thought: Boosting Logical Reasoning in LLMs"
date: 2023-08-31 00:00:00 +0900
description: Graph-structured thought representation for improving logical reasoning in large language models.
tags: [llm, reasoning]
categories: paper-review
related_posts: true
---

This paper proposes representing intermediate reasoning steps as a **graph** rather than a linear chain. While Chain-of-Thought (CoT) forces reasoning into a sequence, graph-structured representations allow:
- **Branching**: exploring multiple reasoning paths simultaneously
- **Merging**: combining conclusions from different sub-problems
- **Cycles**: iterative refinement of intermediate conclusions

The graph-of-thought framework generalizes CoT and Tree-of-Thought and shows improvements on multi-step logical reasoning benchmarks that require non-linear deduction.

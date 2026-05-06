---
layout: post
title: "Beyond Chain-of-Thought: Graph-of-Thought Reasoning in LLMs"
date: 2023-09-04 00:00:00 +0900
description: Review of a paper extending CoT prompting to graph-structured reasoning for large language models.
tags: [llm, reasoning]
categories: paper-review
related_posts: true
---

This paper argues that Chain-of-Thought prompting, while effective, is fundamentally limited by its linear structure. Real problem-solving often requires non-linear reasoning — revisiting earlier steps, exploring parallel branches, and merging independent inferences.

**Graph-of-Thought** represents reasoning as a directed acyclic graph where:
- Nodes are intermediate thoughts or conclusions
- Edges represent inference or dependency
- Multiple paths can be evaluated simultaneously

Compared to CoT and Tree-of-Thought, GoT achieves better performance on tasks requiring complex compositional reasoning while using fewer tokens in some settings, since graph structure prevents redundant re-derivation of shared sub-conclusions.

---
layout: post
title: "Graph of Thoughts: Solving Elaborate Problems with LLMs"
date: 2023-08-29 00:00:00 +0900
description: Generalizing CoT and ToT to arbitrary graph-structured reasoning over LLM thoughts.
tags: [reasoning, llm]
categories: paper-review
related_posts: true
---

**Graph of Thoughts (GoT)** generalizes Chain-of-Thought (CoT) and Tree-of-Thought (ToT) by representing intermediate LLM reasoning as an arbitrary directed graph rather than a chain or tree.

## Why a graph?

Chain-of-Thought forces a strictly linear reasoning trajectory. Tree-of-Thought adds branching but no cross-branch interaction. Real problem solving often involves:
- **Aggregation**: combining results from multiple branches into a single conclusion
- **Refinement**: iteratively improving a thought by feeding it back through the model
- **Cross-pollination**: using insights from one branch to inform another

A graph naturally accommodates all three operations as edges.

## Operations on Thoughts

GoT defines a small set of graph transformations:
- **Generate** — expand a node into successors
- **Aggregate** — merge multiple nodes into one
- **Refine** — replace a node with an improved version
- **Score / select** — evaluate and prune nodes

These compose into reasoning workflows tailored to the problem structure.

## Results

GoT outperforms CoT and ToT on tasks where compositional structure matters — sorting, set intersection, document merging, keyword extraction — often using fewer LLM calls because shared sub-results aren't recomputed across branches.

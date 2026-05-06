---
layout: post
title: "Chain-of-Thought Prompting: Key Papers and Variants"
date: 2023-12-12 00:00:00 +0900
description: A reading list of important Chain-of-Thought papers and variants — decomposition, plan-and-solve, faithfulness.
tags: [reasoning, llm]
categories: paper-review
related_posts: true
---

A reading list and short notes on Chain-of-Thought (CoT) prompting and its variants.

## Question Decomposition Improves the Faithfulness of Model-Generated Reasoning

Decomposing a question into smaller sub-questions and answering each separately produces reasoning that is **more faithful** to the model's actual computation — i.e., the stated reasoning is more likely to reflect the actual cause of the answer. This addresses a long-standing concern with CoT: that the explanation may be post-hoc rationalization rather than genuine reasoning.

## Plan-and-Solve Prompting

**Plan-and-Solve** improves zero-shot CoT without requiring task-specific exemplars. The prompt explicitly instructs the model to first *plan* (devise a strategy), then *solve* (execute the plan), separating high-level reasoning from low-level computation. This two-phase structure reduces the rate of computational mistakes that plague single-pass CoT.

## Common Themes

Across these variants, the recurring insight is that CoT's quality depends heavily on:
- **Structural decomposition** of the reasoning process
- Separation of *planning* from *execution*
- Mechanisms that make the reasoning audit-able and faithful, not just plausible

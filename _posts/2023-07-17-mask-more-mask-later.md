---
layout: post
title: "Mask More and Mask Later (ACL 2022)"
date: 2023-07-17 00:00:00 +0900
description: Efficiency improvements to MLM pre-training by restructuring which information flows dominate.
tags: [nlp, llm]
categories: paper-review
toc:
  sidebar: left
related_posts: true
---

## Core Idea

The paper identifies two types of information in each token: **token information** and **position information**. Through attention, masking creates four distinct information flows combining masked/unmasked entities with self-loop and transfer mechanisms.

The key observation: two flows are particularly valuable for pre-training —
1. *Unmasked tokens gathering context from each other*
2. *Transferring that context to masked positions for prediction*

## Proposals

**Mask More**: Use a higher masking rate than the standard 15%, since the masking rate determines which flows dominate. Higher rates strengthen the valuable context-transfer flow.

**Mask Later**: Use token embeddings during final estimation while *ignoring* the `[MASK]` token's own information flow during intermediate computations. This disentangles the `[MASK]` token's dual role as both a target and a context provider.

## Result

Restructuring which information flows are emphasized during pre-training achieves greater computational efficiency while maintaining model quality — effectively doing more with each training step.

---
layout: post
title: "Memorization Without Overfitting in Large Language Models"
date: 2023-09-07 00:00:00 +0900
description: Analyzing when and how LLMs memorize training data without generalizing worse.
tags: [llm, theory]
categories: paper-review
related_posts: true
---

This paper examines a counterintuitive phenomenon: LLMs can memorize specific training examples *without* suffering the generalization penalty typically associated with memorization in smaller models.

Key findings:
- Memorization grows with model scale but does not necessarily correlate with reduced generalization
- Rare and unique sequences are memorized disproportionately
- Training dynamics: memorization of individual examples follows predictable curves during training
- Implications for privacy: models can regurgitate verbatim training text, creating data extraction risks

The work raises fundamental questions about what "overfitting" means for foundation models trained on internet-scale data.

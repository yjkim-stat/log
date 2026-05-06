---
layout: post
title: "Blessing of Class Diversity in Pre-training"
date: 2023-07-08 00:00:00 +0900
description: Theoretical analysis of why MLM works as a pre-training objective through the lens of class diversity.
tags: [nlp, llm]
categories: paper-review
related_posts: true
---

This paper provides a theoretical perspective on why Masked Language Modeling (MLM) is an effective pre-training task by showing that *class diversity* in the pre-training data is a key driver of transferability to downstream tasks.

The analysis formalizes how the diversity of token contexts during pre-training shapes the quality of learned representations — connecting unsupervised pre-training objectives to supervised downstream performance in a principled way.

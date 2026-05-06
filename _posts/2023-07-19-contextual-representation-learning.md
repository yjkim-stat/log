---
layout: post
title: "Contextual Representation Learning beyond Masked Language Modeling"
date: 2023-07-19 00:00:00 +0900
description: ACL 2022 — extending contextual representation learning beyond the standard MLM objective.
tags: [nlp, representation-learning]
categories: paper-review
related_posts: true
---

This ACL 2022 paper explores methods for contextual representation learning that go beyond the standard MLM objective. The work examines what properties a pre-training objective needs to produce strong transferable representations, and proposes extensions that capture richer contextual information than token-level masking alone.

The key insight is that MLM, while effective, operates at the token level and may miss higher-order contextual dependencies that matter for downstream tasks.

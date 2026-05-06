---
layout: post
title: "RL Study Notes — Lecture 1: MDP, Behavior Cloning, DAgger"
date: 2025-09-22 02:00:00 +0900
description: Foundational concepts for reinforcement learning applied to language models — MDPs, value functions, and imitation learning.
tags: [rl, rlhf]
categories: notes
toc:
  sidebar: left
related_posts: true
---

## Markov Decision Process (MDP)

The standard formalism for RL: $\mathcal{M} = (\mathcal{S}, \mathcal{A}, P, R, \gamma)$
- $\mathcal{S}$: state space
- $\mathcal{A}$: action space
- $P(s' \mid s, a)$: transition probability
- $R(s, a)$: reward
- $\gamma \in [0, 1)$: discount factor

**Markov property**: the future depends only on the *current* state-action pair, not the full history.

## RL Objective

Maximize expected discounted return under policy $\pi$:

$$J(\pi) = \mathbb{E}_{\tau \sim \pi}\!\left[\sum_{t=0}^{\infty} \gamma^t r_t\right]$$

This measures long-run performance, with $\gamma$ trading off immediate vs. future reward.

## Value Functions

- **State-value**: $V^\pi(s) = \mathbb{E}_\pi[\sum_t \gamma^t r_t \mid s_0 = s]$ — how good is being in state $s$?
- **Action-value**: $Q^\pi(s, a) = \mathbb{E}_\pi[\sum_t \gamma^t r_t \mid s_0 = s, a_0 = a]$ — how good is taking action $a$ in state $s$?

Both satisfy **Bellman equations** linking immediate reward to discounted future value.

## Imitation Learning

**Behavior Cloning (BC)**: supervised learning from expert demonstrations.
- Cross-entropy loss for discrete actions, squared error for continuous
- **Compounding error problem**: small mistakes drift the agent into states the expert never visited, where the policy is undefined

**DAgger (Dataset Aggregation)**: iterative refinement
1. Train policy on expert data
2. Roll out the learned policy to collect *its own* state distribution
3. Query expert for correct actions on those states
4. Aggregate and retrain

This progressively aligns training data with the learner's actual deployment distribution, fixing BC's distribution shift problem.

*(Optimal value functions and inverse RL deferred to a later lecture.)*

---
layout: post
title: "RL Study Notes — Lecture 2: Value Functions and Bellman Equations"
date: 2025-09-29 00:00:00 +0900
description: Value functions, Bellman equations, optimal policies, and a Gridworld example.
tags: [rl, rlhf]
categories: notes
toc:
  sidebar: left
related_posts: true
---

## Value Functions Revisited

Two complementary value functions measure expected long-term return:

**State-value** $V^\pi(s)$: expected return starting from state $s$ following policy $\pi$.

**Action-value (Q)** $Q^\pi(s, a)$: expected return starting from state $s$, taking action $a$, then following $\pi$.

Relationship: $V^\pi(s) = \sum_a \pi(a \mid s) Q^\pi(s, a)$

## Bellman Equation

The recursive structure underlying all value-based RL:

$$V^\pi(s) = \mathbb{E}_{\pi, P}\!\left[r + \gamma V^\pi(s')\right]$$

In words: **current value = immediate reward + discounted next value**.

This recursion enables iterative computation (value iteration, policy evaluation) instead of summing infinite trajectories.

## Optimal Value Functions

The *best* possible value over all policies:

$$V^*(s) = \max_\pi V^\pi(s), \quad Q^*(s, a) = \max_\pi Q^\pi(s, a)$$

**Bellman optimality equation**:

$$V^*(s) = \max_a \mathbb{E}_P[r + \gamma V^*(s')]$$

This is the foundation for value iteration, Q-learning, and policy improvement theorems.

## Gridworld Example

A simple navigation grid illustrates how rewards and transitions combine into value estimates. Walking through the iteration by hand makes the recursion tangible: each state's value depends only on its neighbors' values plus immediate rewards.

The exercise also demonstrates how $\gamma$ controls the planning horizon — higher $\gamma$ propagates reward signal further from terminal states.

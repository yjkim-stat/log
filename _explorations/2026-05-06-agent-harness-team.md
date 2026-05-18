---
layout: post
title: "Building an Agent Team with a Harness: Concepts and References"
date: 2026-05-06
description: >
  A curated reference on agent harnesses — the scaffolding that lets a team
  of AI agents collaborate across long-running, multi-context sessions.
tags: [agents, multi-agent, harness, llm-serving, architecture]
series: Agent Team Architecture
chapter: 1
toc:
  sidebar: left
---

## What Is an Agent Harness?

An **agent harness** is the scaffolding layer that wraps around one or more
LLM-based agents to give them the structure they need to operate across
multiple context windows, hand off work between one another, and recover from
partial failures.

A single-context prompt is fine for short tasks. Long-running tasks — writing
a full-stack application, running a research pipeline, executing a multi-step
experiment — exceed any single context window. The harness solves this by:

1. **Initializing** state once at the start (env setup, goal decomposition,
   artifact scaffolding).
2. **Routing** each subsequent session to a specific agent role (planner,
   coder, evaluator).
3. **Persisting** progress artifacts so each new agent can resume where the
   last one left off.

---

## Architecture Patterns

### 1. Single agent + tool loop

The simplest baseline. One agent calls tools (search, code execution, file
read/write) in a loop until the task is complete. Works well when the task
fits in one context window and doesn't require subjective evaluation.

### 2. Orchestrator + subagents (parallel workers)

A **lead agent** (orchestrator) decomposes a query, spawns specialized
**subagents** in parallel, collects their results, and synthesizes an answer.
Each subagent operates in its own context window with its own tool set.

Anthropic uses this pattern in their internal research pipeline:
a `LeadResearcher` orchestrates multiple search-and-summarize subagents that
explore different facets of a question simultaneously.

```
Query
  └─ LeadResearcher (orchestrator)
        ├─ SubAgent A: web search + summarize
        ├─ SubAgent B: paper retrieval + summarize
        └─ SubAgent C: code analysis + summarize
              → synthesized answer
```

### 3. Planner → Generator → Evaluator (three-agent harness)

Anthropic's most recent harness design separates three concerns that benefit
from independent prompts and context windows:

| Role | Responsibility |
|---|---|
| **Planner** | Reads requirements; produces a step-by-step execution plan with checkpoints |
| **Generator** | Follows the plan; writes code / content; emits progress artifacts |
| **Evaluator** | Runs tests / checks; marks tasks passing or failing; feeds results back |

Separating *generation* from *evaluation* is the key insight: the evaluator
can apply subjective judgment without polluting the generator's context, and
objective test results stay reproducible. The planner's artifact (the plan
file) becomes the shared contract between all three roles across sessions.

```
Session 1:  Planner  → plan.md + requirements.md (artifacts)
Session 2:  Generator→ reads plan.md → writes code → updates progress.md
Session 3:  Evaluator→ runs tests → marks plan.md items ✓/✗
Session 4:  Generator→ reads failing items → fixes → ...
```

---

## Key Resources

### Anthropic Engineering

| Resource | What it covers |
|---|---|
| [Building effective agents](https://www.anthropic.com/research/building-effective-agents) | Foundational patterns: augmented LLM, workflows, multi-agent |
| [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) | Two-agent (initializer + coder) and three-agent (planner + generator + evaluator) harness design |
| [Harness design for long-running application development](https://www.anthropic.com/engineering/harness-design-long-running-apps) | Practical harness engineering — artifacts, checkpoints, recovery |
| [How we built our multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system) | LeadResearcher + parallel subagents; interleaved thinking; context management |
| [Scaling managed agents](https://www.anthropic.com/engineering/managed-agents) | Decoupling the "brain" (model) from the "body" (harness runtime) |
| [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) | Context compaction, structured note-taking, multi-agent context hygiene |
| [Claude Code sub-agents docs](https://docs.anthropic.com/en/docs/claude-code/sub-agents) | Sub-agent API: custom system prompts, tool scoping, independent permissions |

### LangGraph

| Resource | What it covers |
|---|---|
| [Multi-agent concepts](https://langchain-ai.github.io/langgraphjs/concepts/multi_agent/) | Supervisor vs. swarm topologies |
| [Hierarchical agent teams tutorial](https://langchain-ai.github.io/langgraph/tutorials/multi_agent/hierarchical_agent_teams/) | Top-level supervisor → mid-level supervisors → worker nodes |
| [Multi-agent network tutorial](https://langchain-ai.github.io/langgraph/tutorials/multi_agent/multi-agent-collaboration/) | Peer-to-peer agent collaboration graph |

### AutoGen (Microsoft)

| Resource | What it covers |
|---|---|
| [AutoGen stable docs](https://microsoft.github.io/autogen/stable/) | Core framework; event-driven multi-agent runtime |
| [AgentChat user guide](https://microsoft.github.io/autogen/stable//user-guide/agentchat-user-guide/index.html) | High-level API: preset agent behaviors, team patterns |
| [Agent and multi-agent concepts](https://microsoft.github.io/autogen/stable//user-guide/core-user-guide/core-concepts/agent-and-multi-agent-application.html) | Message passing, agent lifecycle |

---

## My Experiment

> *Coming soon — building a three-role agent team (planner / generator /
> evaluator) that processes a batch of recent ML papers and produces a
> structured weekly digest.*

Design notes will go here once the first run is complete.

---

## Takeaways (so far)

- The **harness design** matters as much as model capability. The harness
  encodes assumptions about what the model can't do on its own — test those
  assumptions early.
- **Artifact-first thinking**: design the files each agent reads and writes
  *before* writing prompts. The artifacts are the API between agents.
- Start with the simplest topology (single agent + tools); add roles only
  when a specific failure mode demands it.

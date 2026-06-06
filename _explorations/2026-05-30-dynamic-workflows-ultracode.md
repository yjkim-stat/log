---
layout: post
title: "Dynamic Workflows and Ultracode — Anthropic Ships the Orchestrator Pattern"
date: 2026-05-30 14:00:00 +0900
description: >
  An exploration of the two Claude Code features released alongside
  Opus 4.8 on May 28, 2026 — Dynamic Workflows (Claude writes a
  JavaScript orchestration script that fans out up to 1,000 subagents)
  and Ultracode (a session-level setting that triggers workflows
  automatically at xhigh reasoning effort) — and what they imply for
  hand-rolled agent harnesses.
tags: [agents, claude-code, multi-agent, orchestration, workflows]
series: Agent Team Architecture
chapter: 6
toc:
  sidebar: left
---

On May 28, 2026, Anthropic released **Claude Opus 4.8** alongside
two Claude Code features that, taken together, are the first
vendor-shipped implementation of the orchestrator pattern the
[Agent Team Architecture series]({% link _explorations/2026-05-06-agent-harness-team.md %})
has been writing about by hand: **Dynamic Workflows** and
**Ultracode**.

This post covers what they are, what's interesting about how they're
designed, where they fit (and don't fit) in the broader landscape,
and what they imply for the hand-built CCTT-style
([Chapter 5]({% link _explorations/2026-05-28-cctt-research-team-repo.md %}))
agent teams the series has been documenting.

---

## 1. What Was Actually Released

**Dynamic Workflows.** When you describe a complex task, Claude
writes a **JavaScript orchestration script** — not a plan in
natural language, but actual code — that spawns subagents in
parallel, dispatches work to them, validates their outputs, and
folds the results back. The script runs in a background runtime;
your main session stays responsive.

**Ultracode.** A *setting*, not a model effort tier — though it
sends `xhigh` reasoning effort to the model in addition to its
other effects. When Ultracode is on, Claude **auto-decides when to
fan out** to a workflow without you having to invoke one
explicitly. It lasts for the current session and resets on a new
session.

Together: Dynamic Workflows is the *capability*; Ultracode is the
*automatic trigger* for it.

Availability — research preview in the Claude Code CLI, Desktop
app, and VS Code extension (Max / Team / Enterprise plans), plus
the Claude API, Amazon Bedrock, Vertex AI, and Microsoft Foundry.

Shipped in Claude Code v2.1.154.
[[announcement]](https://claude.com/blog/introducing-dynamic-workflows-in-claude-code) ·
[[docs]](https://code.claude.com/docs/en/workflows)

---

## 2. The Architecture in One Picture

```
       ┌──────────────────────────────────────┐
       │  Main session (your Claude window)   │
       │  - stays responsive                  │
       │  - writes the orchestration script   │
       └────────────────┬─────────────────────┘
                        │ dispatches
                        ▼
   ┌────────────────────────────────────────────────┐
   │  JavaScript orchestration script (runtime)     │
   │  - spawns subagents                            │
   │  - holds intermediate results in variables     │
   │  - cross-checks outputs                        │
   └─────┬──────────┬──────────┬─────────────┬─────┘
         ▼          ▼          ▼             ▼
       sub-1      sub-2      sub-3   …   sub-1000
       (own ctx) (own ctx) (own ctx)    (own ctx)
       up to 16 concurrent, 1000 total per workflow
       all subagents run in acceptEdits mode
```

A few things in this diagram are worth pausing on.

### The orchestration target is JavaScript, not natural language

This is the design choice that distinguishes Dynamic Workflows from
"Claude with sub-agent tool calls in a loop." Claude doesn't *plan
in prose* and then dispatch — it *writes a program* that does the
dispatching. That program is inspectable, version-controllable, and
**re-runnable**. The workflow becomes a durable artifact, not a
one-shot conversation.

The implication: a workflow that works once can be saved and
applied to similar problems later, without re-deriving the plan.

### Intermediate results live in script variables

Subagent outputs are stored as JavaScript values inside the
orchestration script — *not* in the main session's context window.
The main Claude only sees the workflow's final report. This is the
piece that makes "1,000 subagents" tractable: if every subagent's
output went back into the main context, you'd blow the window in
seconds.

The script is doing context-management work that, in a hand-rolled
agent team, you'd have to design manually with
[the `intra/extra/` folder split]({% link _explorations/2026-05-18-when-and-how-to-use-agents.md %}).
Here it's baked in.

### Subagents run in `acceptEdits` mode

Each subagent has its own context window *and* full file-edit
authority — no per-call permission prompts. The validation step
happens *after* edits, not before.

This is the part that requires the most user trust. It also
implies that the *real* gate is the workflow's validation logic,
not the model's permission system. If the validators are weak, the
edits land anyway.

### Validation by adversarial cross-check

The orchestration pattern Anthropic ships isn't just "fan out and
collect." It's:

> Subagent A produces a result → Subagent B tries to refute it →
> iterate until the answers converge → only then fold back to the
> orchestrator.

This is the same shape as the rubric-driven
[CCTT loop]({% link _explorations/2026-05-28-cctt-research-team-repo.md %})
and the
[research-agent-team rules]({% link _explorations/2026-05-18-research-agent-team-rules.md %}):
generation and evaluation happen in separate contexts, and only
convergence — not single-pass success — counts as done.

---

## 3. The Limit Numbers (and Why They Matter)

| Limit | Value |
|---|---|
| Concurrent subagents | **16** |
| Total subagents per workflow | **1,000** |
| Subagent edit mode | `acceptEdits` (no permission prompts) |
| Session scope of Ultracode | one session; resets on new |
| Effort level Ultracode sends to model | `xhigh` |

The 16-concurrency cap is the more interesting one. It says: the
runtime is not betting on infinite parallelism, but on **deep
pipelines of moderate-width fan-out**. 1,000 sequential rounds of
16 each is a very different shape from 1,000 simultaneous workers.

This shape matches what's actually useful for code tasks:
*per-file* parallelism (each file gets its own subagent), with the
file-list being the depth dimension. Bun's Zig→Rust port (the
flagship example: ~750,000 lines, 99.8% test pass, ~6–11 days) used
exactly this shape: one workflow mapped Rust lifetimes for every
struct field; the next wrote each `.rs` as a behavior-identical
port of its `.zig` counterpart, with two reviewer subagents per
file.

---

## 4. When to Use Each Mode

Three usage levels are now available:

| Mode | When |
|---|---|
| **Plain Claude Code** | Tasks that fit comfortably in one context window: one file, one bug, one short refactor. |
| **`/workflow` (explicit)** | You know the task is large and parallelizable. You want to inspect Claude's orchestration script before letting it run. |
| **Ultracode (automatic)** | You don't want to think about whether a task warrants a workflow. You accept that Claude may fan out when it judges the task large enough. |

The cost asymmetry matters: workflows can consume *substantially*
more tokens than a single session because each subagent has its
own context. The Anthropic post and the third-party reviews
converge on the same recommendation: **run Ultracode on a scoped
task first**, see how it decomposes and how many subagents spawn,
and decide whether the budget makes sense before pointing it at a
real workload.

The hidden trap with Ultracode is that the *easy* tasks now look
the same as the *expensive* tasks from the user's side. A small
refactor that doesn't need a workflow might still trigger one,
because Claude's "is this big enough?" judgment is conservative on
the safe side.

---

## 5. What This Validates About the Architecture Series

If you read this series straight through, the trajectory has been:

| Chapter | Argument |
|---|---|
| [1]({% link _explorations/2026-05-06-agent-harness-team.md %}) | Multi-agent harnesses are the natural next step for long-running work |
| [2]({% link _explorations/2026-05-18-when-and-how-to-use-agents.md %}) | The right way to think about it is decision axes + an `intra/extra/` communication layout + a rubric harness |
| [3]({% link _explorations/2026-05-18-research-agent-team-rules.md %}) | Concrete rules for a research team |
| [4]({% link _explorations/2026-05-25-implicit-instruction.md %}) | Cross-session learning via evolving instructions |
| [5]({% link _explorations/2026-05-28-cctt-research-team-repo.md %}) | An actual deployed repository running all of the above |

Dynamic Workflows now ships, from the vendor, **the same shape**:

- A *script* (the JavaScript orchestrator) plays the role of the
  rubric harness's lead — it dispatches, scores, and aggregates.
- *Variables* in the script play the role of the `intra/` folder —
  inter-agent state that the user doesn't read.
- The *final report* plays the role of the `extra/` folder — the
  user-facing summary.
- *Cross-checking subagents* play the Critic / Judge role from the
  CCTT team layout.

The architecture wasn't speculative; the vendor agreed. What
changes is *who maintains the orchestration code*: in CCTT it was
the team-lead agent writing markdown rubrics; in Dynamic Workflows
it's Claude itself writing JavaScript at runtime.

---

## 6. What This Doesn't Replace

Three things the series argued for that Dynamic Workflows
**doesn't** address:

### 6.1 Implicit instruction across sessions

Workflows are scoped to a single session. Ultracode resets on new
session. There's no built-in mechanism to **accumulate lessons**
from yesterday's workflow into tomorrow's orchestration script.
[Chapter 4's implicit-instruction loop]({% link _explorations/2026-05-25-implicit-instruction.md %})
is still the open problem; workflows are the *per-session* engine,
not the *across-session* learning loop.

The handoff hook in
[CCTT]({% link _explorations/2026-05-28-cctt-research-team-repo.md %})
still has to live somewhere.

### 6.2 Domain-specific rubrics

The orchestration script's validation logic is what Claude infers
on the fly. For domains with **specific, non-obvious quality
criteria** — research-paper-writing has *noun-of-claim accuracy*,
mathematical proofs have *lemma ordering*, system papers have
*ablation completeness* — the on-the-fly validator is going to
under-perform a domain-tuned rubric.

This is where skills like
[research-paper-writing]({% link _explorations/2026-05-18-research-paper-writing-skills.md %})
and the CCTT
[`extend-experimental-results`]({% link _explorations/2026-05-28-cctt-research-team-repo.md %})
skill still earn their keep — even *inside* a workflow,
domain-specific rubrics improve the quality bar of the workflow's
validators.

### 6.3 The decision frame

[Chapter 2's three axes]({% link _explorations/2026-05-18-when-and-how-to-use-agents.md %}) —
purpose (automation vs. exploration), problem complexity (depth
vs. breadth), human-in-the-loop (scope agreement vs. mid-loop
review) — still applies. Ultracode automates the *how* (fan-out
when needed); it doesn't automate the *when* (should I be doing
this with agents at all?). That decision is still yours.

---

## 7. What I'm Watching

- **Will workflows become user-editable artifacts?** Right now the
  JavaScript orchestration script is written by Claude on demand.
  If users can *save* and *parameterize* workflows for reuse —
  effectively turning them into named procedures — workflows
  become a new layer of the Claude Code skill system. The docs
  hint at re-runnability; the practice will tell.
- **The 16-concurrency cap.** This is the right number for code
  tasks; it may be the wrong number for research / planning tasks
  where you want a wider exploratory frontier. Whether the cap
  gets configurable is the structural question.
- **Cross-checking honesty.** The "A proposes, B refutes,
  converge" pattern is only as good as B's incentive to refute.
  If both subagents share the same prior and the same prompt
  scaffolding, convergence may be premature agreement, not real
  validation. Adversarial subagent design is the next research
  frontier inside the workflow runtime.
- **Cost transparency.** Workflows can easily spend 10–100× a
  normal session. The session-level token accounting will need to
  surface workflow cost separately, or users will quietly discover
  they spent the monthly budget on one Ultracode-triggered audit.
- **The Ultracode/MEMENTO pairing.** Subagent contexts under
  Ultracode are independent — each one re-builds its working set
  from scratch. If
  [MEMENTO's context-management training]({% link _posts/2026-05-18-memento-context-management.md %})
  lands in production, every subagent inside a workflow could
  start with compressed memory and the per-workflow budget
  multiplier drops significantly.

---

## 8. TL;DR

Dynamic Workflows is the vendor-shipped orchestrator-and-subagent
pattern the
[Agent Team Architecture series]({% link _explorations/2026-05-06-agent-harness-team.md %})
has been writing about by hand: a JavaScript script Claude
generates that fans out up to 1,000 subagents (16 concurrent),
holds intermediate state in script variables instead of the main
context, and validates by adversarial cross-check. Ultracode is
the session-level setting that turns this on automatically at
`xhigh` reasoning effort.

What it replaces: the *per-session* orchestration plumbing of a
hand-rolled team. What it doesn't replace: cross-session learning
([Chapter 4]({% link _explorations/2026-05-25-implicit-instruction.md %})),
domain-specific rubrics
([Chapter 3]({% link _explorations/2026-05-18-research-agent-team-rules.md %})),
or the
[decision frame for *whether* to use agents at all]({% link _explorations/2026-05-18-when-and-how-to-use-agents.md %}).

The architecture series and the vendor have converged. The next
interesting work is in the layers above and below — durable
workflow artifacts, domain rubrics, and the across-session memory
the vendor hasn't shipped yet.

---

## References

### Anthropic
- [Introducing dynamic workflows in Claude Code](https://claude.com/blog/introducing-dynamic-workflows-in-claude-code) (May 28, 2026).
- [Orchestrate subagents at scale with dynamic workflows — docs](https://code.claude.com/docs/en/workflows).
- [Claude Opus 4.8 announcement](https://www.anthropic.com/news/claude-opus-4-8).

### Third-party coverage
- [InfoQ — Dynamic Workflows for Parallel Agent Coordination](https://www.infoq.com/news/2026/06/dynamic-workflows-claude-code/).
- [MarkTechPost — workflows capped at 1,000 subagents](https://www.marktechpost.com/2026/05/28/anthropic-ships-claude-opus-4-8-alongside-dynamic-workflows-and-cheaper-fast-mode-with-workflows-capped-at-1000-subagents/).
- [Bun Zig→Rust port via Dynamic Workflows](https://medium.com/illumination/claude-codes-dynamic-workflows-the-ai-agent-architecture-that-just-rewrote-750-000-lines-of-code-d605a1d9b6d4).

### Related on this site
- Series:
  [Ch. 1]({% link _explorations/2026-05-06-agent-harness-team.md %}),
  [Ch. 2]({% link _explorations/2026-05-18-when-and-how-to-use-agents.md %}),
  [Ch. 3]({% link _explorations/2026-05-18-research-agent-team-rules.md %}),
  [Ch. 4]({% link _explorations/2026-05-25-implicit-instruction.md %}),
  [Ch. 5]({% link _explorations/2026-05-28-cctt-research-team-repo.md %}).
- [MEMENTO]({% link _posts/2026-05-18-memento-context-management.md %}),
  [Meta-Harness]({% link _posts/2026-05-18-meta-harness.md %}).

---
layout: post
title: "When and How to Use Agents: A Practitioner's Frame"
date: 2026-05-18 20:00:00 +0900
description: >
  A working frame for deciding when to use an agent team and how to set it
  up — three axes (purpose, problem complexity, human-in-the-loop), a
  communication-folder layout for intra-team vs. human-facing messaging,
  and a rubric-driven Pareto harness that co-evolves with the teammates.
tags: [agents, multi-agent, harness, rubric, context-engineering]
toc:
  sidebar: left
---

I keep getting asked the same question in two forms: *"when should I
reach for an agent team?"* and *"how do I set one up so it doesn't fall
apart after a day?"* This post is my current answer.

It builds on the earlier
[harness references]({% link _explorations/2026-05-06-agent-harness-team.md %})
and the
[research-agent rulebook]({% link _explorations/2026-05-18-research-agent-team-rules.md %}),
but the focus here is the **decision frame** — three axes I run a task
through before spinning anything up — plus the **communication layout**
and **rubric harness** that make the team improve over time instead of
plateauing.

---

## 1. Three Axes for Deciding *When*

I cross three independent axes when sizing up a task. Each axis changes
the team I'd build.

### 1.1 Purpose — automation vs. methodology exploration

| Purpose | Shape of the agent use |
|---|---|
| **Repetitive-task automation** | A *small, stable* team. The success criterion is throughput and predictability. Rubric is short, mostly objective. Harness changes rarely. |
| **New-methodology exploration** | A *larger, more disposable* team. The success criterion is *what we learned*, not what we shipped. Rubric is broad and evolving; many runs are throwaways. |

The trap is using exploration-shaped teams for automation work (too much
ceremony) or automation-shaped teams for exploration (premature
convergence on the first plausible path).

### 1.2 Problem complexity — depth vs. breadth

This one is counterintuitive: how I use the team flips depending on **my
own familiarity** with the problem.

| My familiarity | Agent strategy | Why |
|---|---|---|
| **Low — I barely know the field** | Use agents to **go deep**: long surveys, careful taxonomies, lots of citation-following | I lack the prior that lets me prune. Depth compensates for my missing map. |
| **High — I know the terrain** | Use agents to **go wide**: many parallel branches, aggressive variation | I can prune efficiently. The bottleneck is variation, not understanding. |

The first time I built an agent team for a topic I knew well, I made it
go deep — and got back a beautifully thorough document I could have
written myself. The leverage was nil. Breadth, not depth, was what I
couldn't produce on my own.

### 1.3 Human-in-the-loop — when do *I* sit in?

Two moments matter, and they're different jobs:

1. **At the start — scope agreement.** Before the team spawns anything,
   I read the team lead's interpretation of the task and sign off.
   This is where 90% of wasted runs are prevented. The artifact is
   typically a short `HANDOFF.md` + a `plan.md` the lead drafts and I
   redline.
2. **In the middle — feedback and leadership.** Once running, I drop in
   as a *peer to the team lead*: review rubric scores, override
   priorities, mark which Pareto branch to keep. I don't write code or
   prose; I move the rubric.

If I find myself doing a third kind of intervention — copy-pasting
outputs between agents, or rewriting their drafts — that's a signal the
**harness** is wrong, not that I should keep intervening.

---

## 2. A Reference Architecture (the Pattern Claude Uses)

The team shape I default to is the one Anthropic has converged on in its
own engineering posts: **planner → generator → evaluator**, with an
optional **lead** that fans out parallel subagents for breadth tasks.

```
                      ┌─────────────────────┐
                      │   Human (me)        │
                      └──────────┬──────────┘
                                 │ HANDOFF.md, redlines, rubric edits
                                 ▼
                      ┌─────────────────────┐
                      │   Team Lead (Opus)  │
                      └──────────┬──────────┘
                                 │ spawns, rubric, reviews
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                  ▼
       ┌────────────┐     ┌────────────┐     ┌────────────┐
       │ Planner    │     │ Generator  │     │ Evaluator  │
       │ (Sonnet)   │     │ (Sonnet ×N)│     │ (Sonnet)   │
       └────────────┘     └────────────┘     └────────────┘
```

Pointers to the source material:

- Anthropic — *[Building effective agents](https://www.anthropic.com/research/building-effective-agents)*: augmented-LLM, workflows, multi-agent.
- Anthropic — *[Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)*: two- and three-agent harnesses.
- Anthropic — *[How we built our multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system)*: lead + parallel subagents pattern.
- Anthropic — *[Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)*: compaction, structured notes, multi-agent context hygiene.
- LangGraph — *[hierarchical agent teams](https://langchain-ai.github.io/langgraph/tutorials/multi_agent/hierarchical_agent_teams/)*: supervisor-of-supervisors topology.

What I take from these: **roles are cheap, contexts are not.** Splitting
generation from evaluation is worth it because the evaluator can apply
subjective judgment without contaminating the generator's context. The
lead is worth it because *somebody* has to own the rubric.

---

## 3. Communication Layout: `intra/` vs. `extra/`

This is the piece I think gets under-discussed. Agents communicate
through files, not chat history — so the **folder schema is the
protocol**. I split the project tree into two halves by *audience*:

```
project/
├── HANDOFF.md              ← human → team lead (scope, success criteria)
│
├── intra/                  ← agent ↔ agent (formerly: team/)
│   ├── rubric.md           ← lead-owned scoring contract
│   ├── assignments.md      ← who's doing what
│   ├── teammate-<id>.md    ← per-teammate status file
│   ├── review-<round>.md   ← lead's scored review of an output
│   └── scratch/            ← throwaway notes, partial outputs
│
├── extra/                  ← human ↔ team (formerly: docs/)
│   ├── plan.md             ← lead's plan, ready for human redline
│   ├── decisions.md        ← human-visible decision log
│   ├── results/            ← polished outputs the human reviews
│   └── digests/            ← daily/weekly summaries for the human
│
└── artifacts/              ← the actual deliverable (code, draft.tex, …)
```

The naming distinction matters more than the exact words:

- **`intra/`** is the agents' working surface. High churn, terse, written
  in whatever shorthand the team converges on. The human only reads this
  when debugging.
- **`extra/`** is the human-facing surface. Lower churn, written in full
  sentences, indexed by date. This is where I check in.

### Why this separation pays off

1. **Context windows stay small.** A fresh teammate only needs to read
   `intra/assignments.md` + its own `teammate-<id>.md`; it doesn't have
   to wade through human-facing prose.
2. **Human attention stays small.** I open `extra/` and see only what
   I need to decide on. I never have to scroll a 4,000-line scratch log.
3. **Compaction has obvious rules.** When the project grows, `intra/`
   gets rolled into a summary in `extra/digests/`. The two halves
   compact differently and on different schedules.
4. **Failure modes are local.** If teammates start talking past each
   other, the bug is in `intra/`. If I'm losing the thread, the bug is
   in `extra/`. The folder is the diagnosis.

This is essentially **structured note-taking** in the sense of
Anthropic's context-engineering post, with one twist: the *audience*
dimension is made explicit in the directory tree.

---

## 4. The Rubric Harness: Pareto and Co-Evolution

This is the part that turns a multi-agent team from a fancy way to
generate drafts into a thing that **actually improves**.

### 4.1 Rubric = multi-criterion scorecard

For each deliverable type the team produces, the lead maintains a
**rubric** — a small set of criteria, each on a numeric scale, each with
a one-line definition. For a related-work paragraph, that might be:

| Criterion | Definition | Scale |
|---|---|---|
| Coverage | Are the relevant lines of work present? | 1–5 |
| Taxonomy clarity | Are the categories crisp and non-overlapping? | 1–5 |
| Positioning sharpness | Is "what we do differently" stated, not implied? | 1–5 |
| Citation accuracy | Are claims about prior work checkable? | 1–5 |
| Voice | Reads like a researcher, not a summary bot | 1–5 |

Every teammate output is scored on every criterion, and the scores live
in `intra/review-<round>.md`. Numeric scores are the trick that prevents
the lead from rubber-stamping.

### 4.2 Pareto, not weighted sum

Critical move: **don't collapse the rubric to one number.** Two outputs
with the same average can be radically different teammates. Instead, the
lead keeps a **Pareto frontier** of recent outputs and asks:

> "Is this new output dominated by something already on the frontier?
> If not, add it and explain *which trade-off it represents*."

This does two things:

- It surfaces the **trade-offs** the team is actually facing (e.g.
  "coverage costs us positioning sharpness"). Trade-offs that stay
  invisible can't be reasoned about.
- It keeps diversity alive. A weighted sum will converge to one style;
  a Pareto frontier keeps several styles in play until the human picks.

Picking from the frontier is one of the few decisions I, the human,
genuinely add value to.

### 4.3 The rubric evolves — and so do the teammates

The rubric is **not fixed**. It's a living artifact:

- When the human reviews and gives feedback ("the positioning is sharp
  but the paragraph is hard to skim"), the lead **adds a new criterion**
  (`Skimmability: 1–5`) to the rubric.
- When two criteria stop differentiating outputs (every recent output
  scores 5/5 on `Citation accuracy`), they get **retired or tightened**
  (raise the bar, or merge into a stricter combined criterion).
- Rubric edits are committed alongside outputs, so we can later look at
  *which rubric version produced which output*.

This is the **harness loop** in a sentence:

> *Outputs are scored against a rubric; the rubric is scored against the
> human's feedback; both improve together.*

When the rubric improves, the teammates' next outputs improve — not
because the model got better, but because the **target** got sharper.
That's the kind of compounding I want from an agent team and don't get
from a fixed prompt.

A reference point worth naming explicitly: this is the same shape as
[Meta-Harness]({% link _posts/2026-05-18-meta-harness.md %}) — *the
harness is the optimization target, the model is held fixed*. The
research-agent rubric loop is a small, hand-driven version of that
idea, run inside one project.

---

## 5. Putting It Together — a Worked Decision

Suppose the task is: *"survey RL methods for LLM reasoning published in
the last 12 months and produce a positioned related-work section."*

I run the three axes:

- **Purpose:** methodology exploration (one-shot survey, not a recurring
  job) → larger, more disposable team.
- **Complexity:** I know the area moderately well → bias toward
  **breadth** (many parallel sub-surveys), not a single deep one.
- **HITL:** I want to redline scope up front (which subfields count?
  what's the positioning angle?) and then check in once per day on the
  Pareto frontier.

So I spin up:

- `HANDOFF.md` with the scope.
- One **team lead (Opus)** to own the rubric + spawn.
- **Five generator teammates (Sonnet)** in parallel, each owning one
  subfield (DPO-likes, GRPO-likes, process-reward, curriculum/self-play,
  inference-time search).
- One **evaluator teammate (Sonnet)** to score each subfield write-up
  against the rubric and feed it back to the generator.

The lead maintains the rubric and the Pareto frontier in `intra/`. The
human-facing `extra/digests/` gets a daily one-page summary I actually
read. After two days I look at the frontier, pick the trade-off I want
("more positioning, less coverage"), and the lead tightens the rubric
accordingly. Three days later there's a related-work section that's
defensible and a paper trail that explains every choice.

The same task with the same models, without this scaffolding, would
have produced a competent but unsharp summary I'd have rewritten by
hand.

---

## 6. Mental Checklist

When I'm about to start an agent project, I now run through:

- [ ] **Purpose** — automation or exploration? Sized the team accordingly?
- [ ] **Complexity** — depth or breadth? Does the topology match?
- [ ] **HITL** — clear scope-agreement step? Clear mid-loop review cadence?
- [ ] **Layout** — `intra/` vs. `extra/` separation explicit?
- [ ] **Rubric** — written down? Numeric? Pareto, not summed?
- [ ] **Co-evolution** — am I planning to update the rubric, or pretending it's fixed?

If any of these is "no" or "kind of," I fix it before spawning anything.
The most expensive failure mode in agent work is *running for a day
before noticing the team was solving the wrong problem* — and every item
on this list is a cheap check against that.

---

## References

- Anthropic Engineering posts cited inline above
  ([building effective agents](https://www.anthropic.com/research/building-effective-agents),
  [harnesses](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents),
  [multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system),
  [context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)).
- LangGraph — *[hierarchical agent teams](https://langchain-ai.github.io/langgraph/tutorials/multi_agent/hierarchical_agent_teams/).*
- Related on this site:
  [Building an Agent Team with a Harness]({% link _explorations/2026-05-06-agent-harness-team.md %}),
  [Rules for an AI Research Agent Team]({% link _explorations/2026-05-18-research-agent-team-rules.md %}),
  [Meta-Harness review]({% link _posts/2026-05-18-meta-harness.md %}).

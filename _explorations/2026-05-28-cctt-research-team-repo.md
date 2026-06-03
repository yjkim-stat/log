---
layout: post
title: "CCTT — A Working Agent Team for Research Paper Writing"
date: 2026-05-28 11:00:00 +0900
description: >
  A look at CC-Research-Team (CCTT) — my open repository that
  operationalizes the agent-team-rules framework as a concrete
  Claude Code deployment. A team-lead, parallel teammates
  (Professor / Critic / Judge / Writer), three skills, a
  rubric-driven loop, and a handoff hook — packaged so anyone can
  clone it and point it at a new research topic.
tags: [agents, multi-agent, claude-code, research-workflow, skills]
series: Agent Team Architecture
chapter: 5
toc:
  sidebar: left
---

[Chapter 3 of this series]({% link _explorations/2026-05-18-research-agent-team-rules.md %})
wrote down the **rules** I use to run a research agent team —
team-lead in Opus, teammates in Sonnet, a rubric-driven loop, a
LaTeX paper-draft contract.
[Chapter 4]({% link _explorations/2026-05-25-implicit-instruction.md %})
added the temporal dimension: how the team gets better across
sessions.

This chapter is the **implementation**. The repository
[`yjkim-stat/CC-Research-Team`](https://github.com/yjkim-stat/CC-Research-Team)
(CCTT — Claude Code Thinktank) is the concrete Claude Code project
where those rules and skills actually live as code. You can clone
it, point it at a topic, and run the loop end to end.

---

## 1. What CCTT Is

CCTT is a **multi-agent research collaboration system** that runs
inside Claude Code. One user proposes a research topic; multiple AI
teammates explore it in parallel; a team-lead enforces quality
through an evolving rubric; the output is an ICLR-formatted paper
draft.

Three principles sit underneath the whole design:

1. **User as final arbiter.** Rubric scores and teammate consensus
   are not completion criteria. *"All outputs must ultimately align
   with user requirements."* When a rubric and the user's
   intuition conflict, the rubric is rewritten — not the user
   overruled.
2. **Parallel diversity over single output.** Instead of one
   submission per round, multiple teammates are spawned with
   different perspectives. The *differences themselves* are the
   material the team-lead uses to improve.
3. **Perpetual rubric evolution.** Rubric design intentionally
   advances each round — no "good enough" stopping points. New
   evaluation axes are added or existing thresholds raised every
   loop.

Each principle has a teeth: the first kills consensus-driven
mediocrity, the second kills early convergence, the third kills
plateau.

---

## 2. The Team

```
        User (supreme judge)
           ↓
      Team-lead (main agent, Opus)
      ├─ Designs and refines the rubric
      ├─ Provides feedback
      └─ Does NOT directly edit outputs
           ↓ (parallel spawn, Sonnet)
    ┌─────────┬─────────┬────────┬────────┐
   Professor  Critic    Judge    Writer
```

The roles aren't generic — they're shaped by what a research draft
actually needs reviewed:

| Role | Job |
|---|---|
| **Professor** | Domain framing, positioning, related-work coverage |
| **Critic** | Adversarial review: what would a reviewer reject this for? |
| **Judge** | Calibration: are the rubric scores actually defensible? |
| **Writer** | Sentence-level prose, structural flow, ICLR formatting |

The team-lead's job is **not to write** — it's to **harness**. It
designs the rubric, reads teammate outputs, scores them, gives
feedback, and decides which directions to keep. Teammates do the
generation work in parallel.

---

## 3. Repository Layout

```
CC-Research-Team/
├── .claude/
│   ├── agents/                              ← role definitions
│   ├── hooks/handoff.py                     ← session handoff automation
│   └── skills/
│       ├── research-paper-writing/
│       ├── extend-experimental-results/
│       └── iterative-revision-collaboration/
├── workspace/{topic}/                       ← per-topic isolated folders
├── team/                                    ← rubrics & collaboration logs
├── template/                                ← ICLR 2026 LaTeX template
└── writing_examples/                        ← style references
```

The split matches the
[`intra/` vs `extra/` separation]({% link _explorations/2026-05-18-when-and-how-to-use-agents.md %})
from Chapter 2:

- **`.claude/`** — the scaffolding (agents, skills, hooks). Stable.
- **`team/`** — inter-agent collaboration surface (rubrics, review
  rounds). High churn during a project.
- **`workspace/{topic}/`** — per-topic isolated working directory.
  Actual research artifacts go here.
- **`template/` + `writing_examples/`** — fixed contracts for the
  final paper output.

The repository ships only the *scaffolding*. Actual paper drafts
and research outputs are gitignored — each user's topics are their
own.

---

## 4. The Rubric System

Each rubric aspect requires **three explicit components**:

| Component | What it specifies |
|---|---|
| **Purpose** | What the aspect measures and why it matters |
| **Criteria** | Concrete benchmarks per score level (not vague descriptions) |
| **Scale** | A clear numerical anchor with named levels |

Rubrics must cover **both experimental and theoretical
completeness**. A rubric that scores only "is the paper
well-written?" without "are the experiments load-bearing?" is
incomplete.

The rubric is **not fixed**. The team-lead is required to:

- Raise a threshold once every teammate hits it consistently.
- Add a new aspect when a failure mode appears that no current
  aspect catches.
- Demote or remove an aspect that no longer discriminates.

This is exactly the
[rubric harness pattern]({% link _explorations/2026-05-18-when-and-how-to-use-agents.md %})
made concrete: numeric scores, multi-criterion Pareto frontier, and
co-evolution with the user's feedback.

---

## 5. The Three Skills

### 5.1 `research-paper-writing`

A condensed version of the
[Master-cai paper-writing skill]({% link _explorations/2026-05-18-research-paper-writing-skills.md %})
adapted for ML/CV/NLP papers. The activation contract:

1. **Story clarification before editing** — the narrative is
   established before any sentence-level work.
2. **Section-specific guidance** — separate references for Abstract,
   Introduction, Methods, Experiments, Conclusion.
3. **Paragraph-by-paragraph revision** — one focused message per
   paragraph, no batching.
4. **Reverse outlining** — after each section, validate that the
   logical flow survives a backward read.
5. **Claim-evidence alignment** — every major assertion is mapped
   to its supporting experimental evidence before finalizing.

Built-in checks: a paragraph clarity check (*one explicit message
per paragraph? key terms readable in isolation?*) and a
five-dimension self-review (contribution, clarity, experimental
strength, evaluation completeness, method soundness).

### 5.2 `extend-experimental-results`

The most interesting skill, because it's the one that prevents the
team from cheating. The core reorientation:

> *Do not verify a theory's prediction at a single operating point
> and report pass/fail. Map where and when the theory wins.*

Three forbidden moves are written into the skill as hard constraints:

- **No breadth-first point-checking** across unrelated setups.
- **No result inflation** via seed selection or saturation
  avoidance.
- **No silent truncation** of coverage bounds.

The operating principle: **"keep the direction, raise only the
confidence."** Honest nulls define regime boundaries — they are
not failures.

This skill is what stops the rubric harness from converging on
"results that look good" instead of "results that are real."

### 5.3 `iterative-revision-collaboration`

The team-lead's playbook for the *user-in-the-loop* portion. The
activation signals are recognizable:

- Short directional input from the user ("이 방향으로 가자",
  "고민해봐"), not full rewrites.
- One sentence or one concept per round.
- User edits directly between rounds.

Eight operating principles, with the high-leverage ones:

- **Generate options, don't decide.** Produce 2–4 candidates,
  recommend one, apply only after user picks.
- **One sentence per round.** If a downstream sentence also needs
  adjusting, *flag it* — do not silently fix it.
- **Noun-level accuracy is a hard constraint.** Verify that the
  noun-of-claim matches the noun that actually carries the
  property. (This is the single most common silent failure in
  ML-paper prose.)
- **Reject vague phrasing.** Words like "reliably" or
  "shed light on" lack concrete referents.
- **User retains edit rights.** Always re-read current state before
  proposing; flag regressions explicitly.

Each round is documented in `./team/` with feedback files and
closeout summaries — the rubric escalates with the document.

---

## 6. The Handoff Hook

`hooks/handoff.py` is the piece that closes the loop across
sessions. It registers on Claude Code session events
(`PreCompact`, `PostCompact`, `Stop`, `SessionEnd`) and:

1. **Scans the transcript** for files modified via Write / Edit /
   MultiEdit / NotebookEdit tool calls.
2. **Generates a handoff entry** with sections for modified files,
   status, summary, completed work, decisions, next steps, issues,
   and context — with TODO placeholders for Claude to fill in.
3. **Appends to `docs/handoff/HANDOFF.md`**, archiving old handoff
   files when they exceed 500 lines.
4. **Uses a template** if `.claude/templates/handoff-format.md`
   exists; falls back to a built-in format otherwise.

This is the practical bridge into the
[implicit-instruction loop]({% link _explorations/2026-05-25-implicit-instruction.md %})
from Chapter 4: every session ends with a structured artifact the
next session can read to come up to speed without spelunking the
conversation history.

---

## 7. Writing Standards

Five rules govern the actual paper output. These exist because
they're the failures I keep catching in agent-generated drafts:

1. **Complete sentences.** Use `(`, `)`, `;`, `:`, `-` to connect
   ideas — do not structure paragraphs via bullet symbols when the
   target is prose.
2. **Section order is fixed.** Introduction → Related Work →
   Formulation → Results → Conclusion.
3. **Experimental paragraphs follow a fixed pattern.** State the
   finding, explain the significance, close with "This
   demonstrates…" Predictable shape, fast to skim.
4. **Theoretical results stay in narrative.** Main theorems flow
   inline; proofs go to the appendix.
5. **`.tex` examples are the style reference.** The
   `writing_examples/` folder ships gold-standard files; teammates
   are required to match their voice.

These rules are written down so that the rubric has something
specific to score against. *"Improve the writing"* is not a useful
review; *"the experimental paragraph in §4.2 doesn't close with the
significance sentence"* is.

---

## 8. How a Project Runs End-to-End

```
1. User proposes a topic in HANDOFF.md
   ↓
2. workspace/{topic}/ is created; team-lead drafts initial rubric
   ↓
3. Teammates (Professor, Critic, Judge, Writer) spawn in parallel
   ↓
4. Each teammate writes against the rubric; output to team/
   ↓
5. Team-lead scores each output; writes review-<round>.md
   ↓
6. Teammates revise based on feedback
   ↓
7. Team-lead refines the rubric (new aspect or higher threshold)
   ↓
8. Loop until: experimental + theoretical + user-intent completeness
   ↓
9. Writing phase: template/ copied into workspace; ICLR formatting
   ↓
10. handoff.py captures the session; next session resumes
```

The loop is intentionally tight — each round is one rubric pass,
one teammate spawn, one team-lead review. The rubric carries the
state forward; the handoff carries it across sessions.

---

## 9. Where This Sits in the Series

| Chapter | What it covers |
|---|---|
| [1 — Harness Concepts]({% link _explorations/2026-05-06-agent-harness-team.md %}) | Architecture patterns, key resources |
| [2 — When and How to Use Agents]({% link _explorations/2026-05-18-when-and-how-to-use-agents.md %}) | Three decision axes, intra/extra layout, rubric harness |
| [3 — Research Agent Team Rules]({% link _explorations/2026-05-18-research-agent-team-rules.md %}) | The rulebook this repo implements |
| [4 — Implicit Instruction]({% link _explorations/2026-05-25-implicit-instruction.md %}) | Cross-session learning loop |
| **5 — CCTT (this post)** | The working repository that runs all of the above |

If Chapters 1–4 are theory, Chapter 5 is the system you can clone
and run today. The architecture matches Chapter 3's rules; the
folder layout matches Chapter 2's `intra/extra` split; the handoff
hook is the mechanism Chapter 4 needs to make implicit instruction
work in practice.

---

## 10. What I'm Watching Next

- **Skill drift.** As I use CCTT across more topics, do the three
  skills hold up, or do new failure modes demand new skills?
- **Rubric library.** Right now each project rewrites its rubric
  from scratch. The natural next step is a starter rubric per paper
  type (theory paper, empirical paper, system paper) that the
  team-lead specializes.
- **Cross-topic memory.** Lessons compiled in one
  `workspace/{topic}/` are currently invisible to the next. Wiring
  the
  [implicit-instruction loop]({% link _explorations/2026-05-25-implicit-instruction.md %})
  in as a project-level skill — `.claude/skills/cctt-learnings/` —
  would close that gap.
- **Multi-user.** CCTT is single-user. Letting two users contribute
  to the same topic with separate rubric views (advisor vs. student)
  is the most-requested extension I haven't built.

---

## References

- Repository:
  [github.com/yjkim-stat/CC-Research-Team](https://github.com/yjkim-stat/CC-Research-Team).
- Series chapters:
  [1]({% link _explorations/2026-05-06-agent-harness-team.md %}),
  [2]({% link _explorations/2026-05-18-when-and-how-to-use-agents.md %}),
  [3]({% link _explorations/2026-05-18-research-agent-team-rules.md %}),
  [4]({% link _explorations/2026-05-25-implicit-instruction.md %}).
- Related skill posts:
  [grill-me]({% link _explorations/2026-05-18-grill-me-skills.md %}),
  [Master-cai paper-writing skill]({% link _explorations/2026-05-18-research-paper-writing-skills.md %}).

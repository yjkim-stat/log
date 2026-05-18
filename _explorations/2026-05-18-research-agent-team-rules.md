---
layout: post
title: "Rules for an AI Research Agent Team: From Topic to Paper Draft"
date: 2026-05-18
description: >
  A set of working rules I use to spin up a research agent team — a team
  lead (Opus) coordinating teammate agents (Sonnet) through a rubric-driven
  feedback loop that culminates in a formal LaTeX paper draft.
tags: [agents, multi-agent, harness, research-workflow, llm]
toc:
  sidebar: left
---

The previous post collected
[references on agent harness design]({% link _explorations/2026-05-06-agent-harness-team.md %}).
This post writes down the **actual rules** I have been using to drive a
research agent team. The goal is reproducibility: anyone who reads this
should be able to copy the rules, point them at a new research topic, and
get back a structured paper draft with a paper trail of how it was produced.

The rules are organized into five layers:

1. **Research question scaffolding** — what every prior-work survey must
   answer before any writing starts.
2. **Team topology** — who spawns whom, with which model.
3. **Shared memory layout** — the file conventions that let agents
   communicate without re-deriving context.
4. **The harness loop** — rubric-driven evaluation and revision.
5. **Paper-draft contract** — the exact LaTeX skeleton and paragraph rules
   the team must produce.

---

## 1. Research Question Scaffolding

Every project begins with a `HANDOFF.md` that states the topic in one
paragraph. Before any paper writing happens, the team is required to
produce a prior-work survey that explicitly answers four questions:

> - **Q1.** What problem are we working on?
> - **Q2.** Why is it important?
> - **Q3.** Did other people work on similar problems?
> - **Q4.** If so, what is unique in this work, compared with existing works?

Answers are committed to `./docs/plans/` in both **HTML** (for quick
browsing) and **Markdown** (for diffing). Q4 is the gate: until the team
can articulate a defensible answer to Q4, the project does not advance to
the LaTeX phase. This forces the survey to produce a positioning claim,
not just a literature summary.

---

## 2. Team Topology

- **Team lead — Opus.** Reads `HANDOFF.md`, decomposes work, spawns
  teammates, and runs the evaluation loop. Owns the rubric.
- **Teammates — Sonnet.** Each teammate is spawned to handle one scoped
  subtask (a survey slice, a section of the paper, an experiment).

The asymmetry matters: the lead does heavier judgment work (planning,
rubric scoring, deciding when revision is "good enough"), and teammates do
the heavier *generation* work in parallel. Teammates are spawned on
demand, not kept alive — a clean context per subtask is cheaper and keeps
reasoning crisp.

```
HANDOFF.md
   │
   ▼
┌──────────────────────┐
│ team-lead (Opus)     │── reads HANDOFF, plans, evaluates
└─────────┬────────────┘
          │ spawn (per subtask)
          ▼
┌──────────────────────┐   ┌──────────────────────┐   ┌──────────────────────┐
│ teammate-A (Sonnet)  │   │ teammate-B (Sonnet)  │   │ teammate-C (Sonnet)  │
│ survey: Q1/Q2        │   │ survey: Q3/Q4        │   │ figure 1 + intro     │
└──────────┬───────────┘   └──────────┬───────────┘   └──────────┬───────────┘
           │                          │                          │
           └────────── artifacts → ./team/ ──────────────────────┘
```

---

## 3. Shared Memory Layout

Agents communicate through files, not by reading each other's
conversation history. Three directories carry the entire state of the
project:

| Path | Purpose | Writer |
|---|---|---|
| `./HANDOFF.md` | One-paragraph topic description, success criteria | Human |
| `./team/` | Inter-agent messages, scratch notes, status snapshots | Team lead + teammates |
| `./docs/plans/` | Q1–Q4 answers (md + html), surveyed prior work, positioning | Survey teammates |
| `./example.tex` | The growing paper draft | Section teammates, gated by lead |

The `./team/` folder is the live coordination surface: each teammate
writes a `teammate-<id>.md` status file (current task, blockers, last
output path); the lead writes `rubric.md`, `assignments.md`, and
`review-<round>.md` for each evaluation pass. This means a fresh context
window can be brought up to speed by reading `./team/` alone.

---

## 4. The Harness Loop: Rubric-Driven Iteration

The team lead's job is not to write — it is to **harness**. For each
teammate output, the lead:

1. **Builds (or updates) a rubric** — a small set of numeric criteria
   tailored to the deliverable. E.g. for a related-work paragraph:
   *coverage*, *taxonomy clarity*, *positioning sharpness*, *citation
   accuracy*, each on a 1–5 scale, with a one-line rubric definition per
   criterion.
2. **Scores the output** against the rubric and writes a
   `review-<round>.md` file with scores + specific revision requests.
3. **Sends the review back** to the teammate, which revises and resubmits.
4. **Stops** when no criterion drops below a target threshold *and* the
   lead cannot identify a concrete improvement.

Two design choices keep this loop honest:

- **Numeric scores, written down.** Forcing a number per criterion
  prevents the lead from rubber-stamping; trends across rounds are
  visible.
- **Rubric is mutable.** As the user (me) gives feedback to the lead,
  the rubric itself is updated. The rubric is part of the artifact set
  and evolves with the project.

---

## 5. Paper-Draft Contract (`example.tex`)

The final deliverable is a formal LaTeX paper in
`example.tex`. The contract specifies both **structure** and **paragraph
mechanics**.

### 5.1 Section structure (fixed)

```
1. Introduction
2. Related Work
3. Problem Formulation
4. Theoretical Results
5. Experimental Results
Appendix
```

Every section opens with a short overview that names its subsections and
gives a one-line summary of each — so a reader who only reads section
intros still gets the spine of the paper.

### 5.2 Paragraph mechanics (every section)

The team writes in two passes:

- **Pass 1 — Key sentences.** For each section, generate 3–4 *key
  sentences*. These are the thesis statements; together they must form a
  logical chain.
- **Pass 2 — Expansion.** Each key sentence becomes one paragraph,
  wrapped in `\paragraph{key sentence}` so the key sentence acts as the
  paragraph's lead.

This is the single most important rule: it forces argument structure
before prose, and it makes revision local (a flawed paragraph is a
flawed key sentence).

### 5.3 Per-section requirements

**Introduction.**
Three subsections: *Background & motivation* → *Recent related work and
its limits* → *Our work* (main contributions as bullet points). Written
without jargon — a first-time reader should follow it. Includes a
**Figure 1** illustrating what makes our approach different from prior
work; the figure is referenced in prose and walked through as the entry
point to the contribution list.

**Related Work.**
Three categories of prior work. Each category is a short paragraph
that summarizes the line of research and ends by stating what our work
does differently.

**Problem Formulation.**
Fully self-contained. Every symbol and every piece of jargon is
defined — inline or via `\begin{definition}` — *before first use*. A
reader who lands on this section without reading the introduction
should still be able to follow.

**Theoretical Results.**
Lemmas are introduced in the order they are needed for the main
theorem(s). The section narrates the chain: *this lemma gives us X,
which feeds the next lemma, which finally enables the theorem*. Proofs
go to the appendix.

**Experimental Results.**
One paragraph per theoretical result. Each paragraph follows a fixed
template:

1. *Which theoretical result are we trying to validate empirically?*
2. *Setup: what configuration, what is being measured?*
3. *Reported numbers, interpreted in light of the theoretical
   prediction.*

Full experimental details (hyperparameters, dataset stats, hardware) go
to the appendix.

**Appendix.**
One subsection per lemma/theorem proof; one subsection per detailed
experimental setup.

---

## How the Pieces Fit Together

Putting layers 1–5 in motion looks like this:

1. Drop a topic into `HANDOFF.md`.
2. Team lead reads it, drafts an initial rubric, spawns survey teammates.
3. Survey teammates fill `./docs/plans/` (md + html) with Q1–Q4 answers
   until Q4 has a defensible claim. Lead scores, requests revisions,
   iterates.
4. Lead spawns section teammates with the paper contract (Section 5).
   Each teammate first writes key sentences (Pass 1) into `./team/`,
   gets reviewed, then expands into `\paragraph{}`-wrapped prose in
   `example.tex` (Pass 2).
5. Theoretical and experimental teammates work in parallel — the
   experiment teammate consumes the theorem statements as its
   specification.
6. The loop stops when the rubric flatlines and the lead has no new
   revision request.

---

## Why This Works (And Where It Doesn't)

**What this buys:**

- *Reproducibility.* `./docs/plans/`, `./team/`, and the rubric history
  capture how every claim was reached.
- *Parallelism.* Survey, theory, and experiment teammates run
  independently as long as the problem formulation is locked.
- *Quality control without humans-in-every-step.* The rubric is the
  human's leverage — the lead applies it round after round.

**Where it breaks:**

- *Problem-formulation drift.* If the formulation is edited mid-project,
  every downstream artifact has to be re-reviewed against the new
  notation. The lead must aggressively gate changes to
  Problem Formulation.
- *Cost.* Multi-round rubric loops with an Opus lead and Sonnet
  teammates are expensive; rubric thresholds need to be calibrated to
  avoid infinite polishing.
- *Empirical novelty.* The system is strong at structure and survey, but
  the actual *idea* still has to come from the human in `HANDOFF.md`.

---

## What's Next

I am running this workflow on a concrete project right now. A follow-up
exploration will share:

- The actual rubric I converged on for each artifact type
  (survey / intro / related work / theorem / experiment paragraph).
- Failure modes I observed and how I patched the rules.
- The paper draft itself, once the rubric flatlines.

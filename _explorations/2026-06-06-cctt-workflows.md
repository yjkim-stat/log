---
layout: post
title: "Two CCTT Workflows: Theory-Polish and Experiment-Polish for Research Papers"
date: 2026-06-06 18:00:00 +0900
description: >
  A look at two Dynamic Workflows recently added to the
  CC-Research-Team repository — research-phase-polish-thy
  (nine-phase adversarial audit + repair for theory-heavy drafts)
  and research-phase-polish-exps (four-phase claim-preserving polish
  for experimental sections). Both are concrete applications of the
  vendor-shipped orchestrator pattern to the research-paper-writing
  loop.
tags: [agents, claude-code, workflows, research-workflow, multi-agent]
series: Agent Team Architecture
chapter: 7
toc:
  sidebar: left
---

[Chapter 6]({% link _explorations/2026-05-30-dynamic-workflows-ultracode.md %})
introduced Anthropic's Dynamic Workflows / Ultracode features and
argued they're the vendor-shipped version of the orchestrator
pattern the series has been building by hand. This chapter is the
direct follow-up: **two Dynamic Workflows I've added to
[CC-Research-Team](https://github.com/yjkim-stat/CC-Research-Team)**
to operate on a settled research paper draft.

Both workflows live in `.claude/workflows/` and ship with
human-readable docs in `docs/wfs/`. They cover the two halves of
late-stage paper work — *the theory has to hold up*, and *the
prose / figures / evidence has to land* — without restructuring
the argument.

---

## 1. What's in the Repo

```
.claude/workflows/
├── research-phase-polish-thy.js     ← 9-phase audit + repair (theory)
└── research-phase-polish-exps.js    ← 4-phase polish (experiments)

docs/wfs/
├── research-phase-polish-thy.{md,html}
└── research-phase-polish-exps.{md,html}
```

Each workflow is invokable via Claude Code's Dynamic Workflows
runtime. The doc files are the human-readable spec for each — the
same file lives as Markdown for diffing and HTML for browsing.

The two workflows are designed to compose: the theory-polish
workflow is the final correctness gate, and the experiments-polish
workflow is the iterative-improvement loop you run between drafts.

---

## 2. `research-phase-polish-thy` — Adversarial Audit + Repair

### 2.1 What it does

Given a settled theory-heavy LaTeX draft, the workflow runs nine
phases of structured audit and repair. It is deliberately
**adversarial in the audit phase** and **conservative in the
repair phase** — gaps are surfaced harshly, then a separate phase
filters false positives, and only confirmed gaps trigger repairs.

### 2.2 The nine phases

| Phase | Stage | What it does |
|---|---|---|
| **1. Map** | Assessment | Extracts theorems, contributions, quantitative claims, compilation commands |
| **2. Audit** | Assessment | Parallel adversarial checkers run on each proof + a claim-evidence verifier |
| **3. Confirm** | Assessment | Adjudicator filters false positives from the harsh audit |
| **4. Novelty** | Pipeline | Web-searches prior art; checks whether confirmed contributions are genuinely novel |
| **5. Repair** | Pipeline | For each confirmed gap, generates a rigorous proof or an honestly-weakened statement |
| **6. Verify** | Pipeline | Re-audits each repair to ensure it closes the original gap without introducing new ones |
| **7. Apply + Compile** | Application | A single sequential editor applies all verified repairs and recompiles |
| **8. Consistency review** | Validation | Per-section paragraph audits check every claim against the canonical-facts ledger |
| **9. Synthesize** | Validation | Cross-section synthesizer consolidates findings and applies final consistency fixes |

Three things in this phase list are worth pulling out.

### 2.3 The canonical-facts ledger

Before anything else, the workflow parses experiment result files
into a `CANONICAL_FACTS.md` ledger. **Every quantitative claim in
the paper has to match a row in the ledger.** Prose numbers that
don't show up in the ledger are flagged in phase 8.

This is the single most important pattern in the workflow. Without
a ledger, "the paper says X% accuracy" and "the experiment
produced X% accuracy" are two separate claims that nobody
cross-checks. With a ledger, they collapse into one source of
truth, and any drift is a build-time error.

### 2.4 Adversarial audit, then adjudication

Phase 2 runs proof checkers with an explicit "be uncharitable"
prompt. Phase 3 immediately filters that output through an
adjudicator that asks "is this actually a gap, or did the auditor
over-reach?"

Splitting *harshness* from *judgement* into two phases is what
makes the audit usable. A single-pass auditor either misses gaps
(too charitable) or generates noise (too harsh). Two phases let
you tune each end independently.

### 2.5 Sequential editor on `Apply`

Every other phase parallelizes. Phase 7 deliberately doesn't.
A single editor agent applies all verified repairs in sequence,
recompiles after each batch, and only proceeds when the build is
clean.

This is a concession to LaTeX: parallel edits to the same file
cause merge headaches, and many "repairs" are interdependent
(adding a lemma affects the numbering of the next theorem).
Serializing through one agent is the cheap fix.

---

## 3. `research-phase-polish-exps` — Claim-Preserving Polish

### 3.1 What it does

Once the storyline is settled, you don't want to rewrite the
argument — you want to make it land better. This workflow
iterates on prose conciseness, figure quality, evidence surfacing,
and follow-up experiment planning **without strengthening any
claim**.

The hard constraint, written into every phase:

> A proposal passes review only if it *preserves every number,
> citation, cross-reference, and hedge it touches, does not
> strengthen any claim, and does not contradict the fixed
> storyline.*

### 3.2 The four phases

| Phase | What it does |
|---|---|
| **1. Map** | Reads the `.tex`, builds a ledger of load-bearing numbers, citations, hedges, and figures; captures the fixed storyline |
| **2. Diagnose** | Applies seven analytical lenses in parallel (one subagent per lens) to surface candidate improvements |
| **3. Verify** | An adversarial reviewer checks each proposal against the preservation constraints |
| **4. Synthesize** | Consolidates approved proposals into a prioritized dossier and an experiments-to-plan checklist |

### 3.3 The seven lenses

Phase 2 runs seven independent lenses in parallel — each is one
subagent reading the whole draft through one lens:

1. **Succinctness** — remove redundant restatement without deleting
   numbers.
2. **Main-vs-Appendix** — move fine derivations to the appendix
   with pointer sentences.
3. **Figure layout, font, encoding** — visual presentation.
4. **Evidence surfacing** — find existing measured data that
   deserves promotion from appendix to main body.
5. **Experiment planning** — propose new runs that add a *distinct
   message*, not mere reinforcement.
6. **Consistency audit** — flag mismatched quantities or over-claims.
7. **Style parity** — align with gold-standard writing examples in
   `writing_examples/`.

Each lens is a different reviewer perspective. The orchestrator
collects all seven and dedupes overlapping suggestions.

### 3.4 Three forbidden moves

Three things the workflow is explicitly **not allowed to do**:

- **Strengthen a claim.** Proposals can weaken or hold claims
  steady, never strengthen.
- **Reinforce a finding.** New experiments must add *distinct
  evidence*, not pile on more of the same.
- **Drop a hedge.** "Often," "in our setting," "to the best of our
  knowledge" — these can move but can't be silently removed.

The third one is the most subtle. Polish workflows have a
systematic bias toward confident prose, and unhedged sentences read
better. Forbidding hedge removal is a hard floor on that bias.

### 3.5 The rubric escalates

Each round bumps the rubric — eleven aspects and five hard
constraints, with the bar rising every iteration. This is the
[rubric harness pattern]({% link _explorations/2026-05-18-when-and-how-to-use-agents.md %})
specialized to paper polishing. The rubric isn't fixed; it's the
artifact that carries learning across rounds.

---

## 4. Common Design Patterns Across Both Workflows

Both workflows share four patterns that I think generalize beyond
this specific application.

### 4.1 Single source of truth ledgers

- Theory: `CANONICAL_FACTS.md` from experiment result files.
- Experiments: ledger of load-bearing numbers, citations, hedges
  from the existing draft.

The pattern: before any review work happens, derive a *truth
table* and require every downstream claim to match it. This makes
"the paper says X" and "X is true" structurally checkable, not
just plausible-sounding.

### 4.2 Harshness and judgement in separate phases

- Theory: adversarial Audit → Confirm.
- Experiments: parallel Diagnose → adversarial Verify.

Splitting *generation of candidates* from *adjudication of
candidates* gives you two tunable knobs instead of one stuck
trade-off. The auditor can be as harsh as it wants; the
adjudicator can be as charitable as it needs.

### 4.3 Sequential editing under parallel review

Both workflows parallelize review but serialize edits. LaTeX
compilation is the forcing function — file-level merge conflicts
and theorem-numbering dependencies make parallel edits expensive
to reconcile. The orchestrator pays a small wall-clock cost for
a big consistency gain.

### 4.4 Honest weakening as a first-class repair

The repair phase in the theory workflow doesn't always patch a
gap with a stronger proof — sometimes the right move is to
*weaken the claim* to what's actually supportable. This is in
line with the
[`extend-experimental-results` skill]({% link _explorations/2026-05-28-cctt-research-team-repo.md %})'s
"keep the direction, raise only the confidence" principle: the
workflow can downgrade a claim instead of fabricating evidence.

---

## 5. How They Fit the Series

| Chapter | Layer of the stack |
|---|---|
| [1]({% link _explorations/2026-05-06-agent-harness-team.md %}) — concepts | Why multi-agent harnesses |
| [2]({% link _explorations/2026-05-18-when-and-how-to-use-agents.md %}) — decision frame | When to use which shape |
| [3]({% link _explorations/2026-05-18-research-agent-team-rules.md %}) — rules | The team's contract |
| [4]({% link _explorations/2026-05-25-implicit-instruction.md %}) — memory | Across-session learning |
| [5]({% link _explorations/2026-05-28-cctt-research-team-repo.md %}) — repo | CCTT as the implementation |
| [6]({% link _explorations/2026-05-30-dynamic-workflows-ultracode.md %}) — vendor | Anthropic's Dynamic Workflows |
| **7 — this post** — application | Two concrete workflows running inside CCTT |

The progression: the series argued the orchestrator pattern was
right (Ch.1–5), the vendor shipped it (Ch.6), and now CCTT
*uses* the vendor implementation for two specific late-stage
research-paper tasks (Ch.7).

The skills from Chapter 5
([`research-paper-writing`, `extend-experimental-results`,
`iterative-revision-collaboration`]({% link _explorations/2026-05-28-cctt-research-team-repo.md %}))
still hold up — but they were *single-session* tools. The new
workflows are *multi-session-equivalent in one shot*: the
nine-phase theory audit is what an iterative review-and-repair
loop would have produced over many sessions, run as one
orchestrated workflow.

---

## 6. What I Learned About Workflow Design

A few things that surprised me while writing these:

- **The ledger-first move is the single highest-leverage step.**
  Without `CANONICAL_FACTS.md`, every other phase has to
  re-derive ground truth from prose. With it, every downstream
  check collapses to a lookup.
- **Parallel-then-sequential is the natural shape for
  document-level work.** Code might be fully parallelizable
  (one subagent per file), but a single document forces serial
  editing. The workflow's structure should reflect that — fan
  out for review, fan back in for application.
- **Honest weakening needs to be an enumerated verdict.** If the
  repair phase only has "fixed" and "still broken" as outcomes,
  the model will prefer fabricating a fix over weakening a
  claim. Adding "weakened-to-what's-supportable" as a first-class
  verdict changes what the workflow optimizes for.
- **The "be uncharitable" + "filter false positives" split is
  reusable.** I expect to copy this two-phase pattern into other
  workflows beyond research review — code review, security
  review, anything where you want a high recall pass followed by
  a precision filter.

---

## 7. What's Next

- **A third workflow for `intra`-paper structural moves.** The
  current two cover correctness (theory-polish) and presentation
  (experiments-polish). They don't restructure sections, move
  contributions, or reorder a results narrative. That's the next
  natural addition.
- **Workflow re-runnability as artifacts.** Right now each
  invocation is one-shot. Saving the orchestration scripts as
  named, parameterized procedures in the repo — so the same
  workflow can be re-run on the next draft of the same paper —
  is the practical next step.
- **Wiring the workflows into the
  [implicit-instruction loop]({% link _explorations/2026-05-25-implicit-instruction.md %}).**
  The workflows generate enormous amounts of structured feedback.
  Distilling repeated findings ("you keep introducing unhedged
  claims at section openings") into the project's `learnings.md`
  would close the across-session memory gap that Ch.4 flagged
  and Ch.6 noted Dynamic Workflows doesn't yet address.

---

## References

- Repository:
  [github.com/yjkim-stat/CC-Research-Team](https://github.com/yjkim-stat/CC-Research-Team)
  (`.claude/workflows/` and `docs/wfs/`).
- Series chapters:
  [1]({% link _explorations/2026-05-06-agent-harness-team.md %}),
  [2]({% link _explorations/2026-05-18-when-and-how-to-use-agents.md %}),
  [3]({% link _explorations/2026-05-18-research-agent-team-rules.md %}),
  [4]({% link _explorations/2026-05-25-implicit-instruction.md %}),
  [5]({% link _explorations/2026-05-28-cctt-research-team-repo.md %}),
  [6]({% link _explorations/2026-05-30-dynamic-workflows-ultracode.md %}).
- Related skill posts:
  [research-paper-writing]({% link _explorations/2026-05-18-research-paper-writing-skills.md %}).

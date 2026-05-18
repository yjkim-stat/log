---
layout: post
title: "A Packaged Skill for ML Paper Writing: Master-cai/Research-Paper-Writing-Skills"
date: 2026-05-18 14:00:00 +0900
description: >
  A look at Master-cai's Research-Paper-Writing-Skills repository — a Claude
  Code / Codex / Gemini Skill that packages Prof. Peng Sida's paper-writing
  methodology into reusable references for drafting, revising, and
  adversarially reviewing ML/CV/NLP papers.
tags: [agents, claude-code, skills, paper-writing, research-workflow]
toc:
  sidebar: left
---

I keep two kinds of skills around for research work: ones that interrogate
**plans** (`grill-me`, `grill-with-docs` — see
[the earlier post]({% link _explorations/2026-05-18-grill-me-skills.md %}))
and ones that interrogate **writing**. The
[`Master-cai/Research-Paper-Writing-Skills`](https://github.com/Master-cai/Research-Paper-Writing-Skills)
repository is the cleanest entry I've found for the second category.

It packages **Prof. Peng Sida's** open notes on paper writing into a single
Skill that works across Claude Code, Codex, and Gemini. The point isn't
generic LaTeX advice — it's a deliberate, ML/CV/NLP-flavored revision
discipline that an agent can run on your draft.

---

## What's in the Box

The repository's `research-paper-writing/` directory has three layers:

```
research-paper-writing/
├── SKILL.md                 ← the workflow + global rules
├── references/              ← section-by-section playbooks
│   ├── abstract.md
│   ├── introduction.md
│   ├── related-work.md
│   ├── method.md
│   ├── experiments.md
│   ├── conclusion.md
│   ├── paper-review.md      ← adversarial self-review
│   ├── does-my-writing-flow-source.md
│   └── examples/
└── agents/openai.yaml
```

`SKILL.md` is the entry point — it tells the agent *how* to use the
references. The `references/` files are the actual writing playbooks the
agent consults when it gets to a particular section.

Installation is just a copy:

- **Claude Code:** `~/.claude/skills/` (global) or `.claude/skills/`
  (project-local).
- **Codex:** `$CODEX_HOME/skills/`.
- **Gemini:** `~/.gemini/skills/`.

After install you invoke it the way every Skill is invoked — by asking
for it by name in the prompt.

---

## The Five-Step Workflow

`SKILL.md` enforces a sequence that resists the temptation to start
rewriting sentences first:

1. **Clarify the narrative.** Pin down problem → contributions → benefits →
   insights before touching any paragraph.
2. **Apply section-specific guidance.** Pull the appropriate playbook from
   `references/` (e.g. `introduction.md` for the intro).
3. **Rewrite paragraph-by-paragraph.** One paragraph, one message, with
   a topic sentence that names the paragraph's role.
4. **Reverse-outline check.** Read only the topic sentences in order — do
   they form a coherent spine? If not, the structure is wrong, not the
   prose.
5. **Claim–evidence validation.** Every claim, especially in the abstract
   and intro, is matched to a specific experimental result. Unsupported
   claims are either evidenced or weakened.

Two principles run through the whole thing:

- **"One paragraph, one message."** The clarity test asks three things:
  is there a single explicit message? does the opening sentence name the
  paragraph's purpose? does each sentence connect via a logical relation
  (cause / contrast / consequence / refinement)?
- **Figures are core content.** Visual quality is treated as part of the
  argument, not decoration that gets polished last.

---

## The Introduction Playbook (representative example)

`references/introduction.md` is the file that shows what the whole skill
is trying to do. Instead of vague "make the introduction compelling"
advice, it gives a **skeleton with substitutable templates** per part:

The introduction has five parts:

1. Task and application context
2. Prior method limitations and root causes
3. Proposed solution and advantages
4. Additional contributions
5. Experimental validation

And then each part has named *versions* to choose from, e.g.:

| Part | Templates |
|---|---|
| **A. Task setup** | (1) define niche task → applications · (2) lead with applications for a familiar task · (3) general task → specific setting · (4) expose technical challenge first via prior-method failures |
| **B. Challenges** | (1) chain existing methods → show progressive limitations · (2) classical insight → modern gap · (3) for novel tasks, decompose challenges into independent points |
| **C. Method presentation** | (1) single contribution, multiple advantages · (2) two sequential contributions for cascading challenges · (3) new module extending a prior pipeline · (4) observation-driven innovation |

The most useful rule in this file is a warning: **don't hide method
details behind abstract insights** — papers that do this read as
incremental even when the work is genuinely novel.

Each section playbook (`method.md`, `experiments.md`, `related-work.md`,
…) follows the same pattern: a skeleton + a small menu of named templates +
a checklist.

---

## Adversarial Self-Review (`paper-review.md`)

The companion file `paper-review.md` flips the perspective: treat
yourself as a hostile reviewer and probe every weak point before
submission. It defines **five rejection-risk categories**:

| Risk | What a reviewer attacks |
|---|---|
| **Insufficient contribution** | Problem is too common; the technique is already explored |
| **Unclear writing** | Missing technical details; modules introduced without motivation |
| **Weak empirical effect** | Marginal improvements; weak absolute performance |
| **Incomplete evaluation** | Missing ablations, baselines, challenging datasets |
| **Problematic method design** | Unrealistic settings; technical flaws; poor robustness; net negative value |

Authors answer **25 specific questions** across these five categories,
marking each `pass` / `needs revision` / `needs new experiment`. The
loop continues until *no major rejection risk remains*.

The hard constraint: **every major claim, especially in Abstract and
Introduction, must be both technically correct and explicitly supported
by an experimental result.** If unsupported, the claim is either backed
by new evidence or rewritten weaker.

---

## How It Composes With the Other Skills I Use

| Stage | Skill | What it gates |
|---|---|---|
| Plan a piece of work | [`grill-me`](https://github.com/mattpocock/skills/blob/main/skills/productivity/grill-me/SKILL.md) | Unstated assumptions in the plan |
| Plan inside a project's domain | [`grill-with-docs`](https://github.com/mattpocock/skills/blob/main/skills/engineering/grill-with-docs/SKILL.md) | Undocumented decisions and glossary drift |
| **Write the paper** | **`research-paper-writing`** | **Paragraphs without a message; claims without evidence** |

The three sit at different points in the same research pipeline: the
first two enforce discipline *before* the work, the third enforces
discipline *after* the work, while you're turning results into prose.

---

## Why I'm Adopting It

For paper writing I have always relied on ad-hoc heuristics ("does this
paragraph have a topic sentence?", "does the intro motivate the
challenge?"). `research-paper-writing` turns those heuristics into:

- a **playbook** an agent can apply consistently across sections,
- a **reverse-outlining check** that makes structural problems visible
  before they become rewrite cycles,
- a **claim–evidence map** that catches the single most common reason
  papers get desk-rejected — overclaiming in the abstract.

This pairs naturally with the
[research agent team rules]({% link _explorations/2026-05-18-research-agent-team-rules.md %})
I wrote down earlier: the team produces the LaTeX draft; this skill is
the rubric the lead applies when the draft comes back.

---

## References

- Repository:
  [`Master-cai/Research-Paper-Writing-Skills`](https://github.com/Master-cai/Research-Paper-Writing-Skills)
- Original notes credited in the README: **Prof. Peng Sida's open study
  notes** on academic writing.
- License: MIT.

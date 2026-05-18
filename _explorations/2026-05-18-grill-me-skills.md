---
layout: post
title: "Two Skills for Stress-Testing a Plan: grill-me and grill-with-docs"
date: 2026-05-18 12:00:00 +0900
description: >
  A look at two Claude Code Skills from Matt Pocock's skill library —
  grill-me for plan stress-testing, and grill-with-docs for the same
  exercise grounded in a project's glossary and ADRs.
tags: [agents, claude-code, skills, planning, workflow]
series: Skills for Research Workflows
chapter: 1
toc:
  sidebar: left
---

Before any non-trivial change, I want my plan interrogated — not approved.
Two skills from Matt Pocock's open
[`mattpocock/skills`](https://github.com/mattpocock/skills) repository
do exactly that, in two flavors. They are short — under a page each —
but the discipline they encode is hard to recreate from memory in the
middle of a session.

This post walks through:

1. What a Claude Code "Skill" is (one paragraph).
2. The `grill-me` skill — the productivity baseline.
3. The `grill-with-docs` skill — the engineering variant that updates
   project documentation inline.
4. When I reach for each.

---

## What Is a Skill?

A Skill in Claude Code is a small Markdown file with YAML frontmatter
(`name`, `description`) plus a body of instructions. Claude reads the
`description` to decide whether to invoke the skill, and follows the
body when it does. Skills live in a `skills/` directory (or come from a
plugin) and can be triggered explicitly by name or implicitly via
keyword matches.

Both of the skills below are single files. Their power is not the
mechanism — it's the prompt they install.

---

## `grill-me` — Interview-Style Plan Review

[`skills/productivity/grill-me/SKILL.md`](https://github.com/mattpocock/skills/blob/main/skills/productivity/grill-me/SKILL.md)

The entire skill is the following instruction:

> Interview me relentlessly about every aspect of this plan until we reach
> a shared understanding. Walk down each branch of the design tree,
> resolving dependencies between decisions one-by-one. For each question,
> provide your recommended answer.
>
> Ask the questions one at a time.
>
> If a question can be answered by exploring the codebase, explore the
> codebase instead.

Three details do the heavy lifting:

- **"One at a time."** Forces a Socratic loop instead of a wall of
  questions. Each answer becomes context for the next question.
- **"Provide your recommended answer."** Removes the polite ambiguity
  of open questions. Claude has to commit to a default, which gives me
  something to push back on rather than fill in.
- **"Explore the codebase instead."** Routes around lazy questions —
  if the answer is already in the repo, the skill is forbidden from
  asking me.

It's effectively a checklist that prevents three common failure modes
of plan reviews: batching questions, hedging, and asking what the code
already knows.

### When I reach for it

Greenfield design or a fresh refactor plan where there isn't much
documented context yet, and I want to surface the implicit decisions
before any code is written.

---

## `grill-with-docs` — The Same, but Glossary-Aware

[`skills/engineering/grill-with-docs/SKILL.md`](https://github.com/mattpocock/skills/blob/main/skills/engineering/grill-with-docs/SKILL.md)

`grill-with-docs` keeps the interview loop from `grill-me` and layers
on **domain awareness**: it expects the repo to have (or to develop)
a `CONTEXT.md` glossary and a `docs/adr/` folder of Architecture
Decision Records.

### File layout it assumes

```
/
├── CONTEXT.md
├── docs/adr/
│   ├── 0001-event-sourced-orders.md
│   └── 0002-postgres-for-write-model.md
└── src/
```

For repos with multiple bounded contexts, a `CONTEXT-MAP.md` at the
root points to per-context `CONTEXT.md` and `docs/adr/` folders:

```
/
├── CONTEXT-MAP.md
├── docs/adr/                          ← system-wide decisions
└── src/
    ├── ordering/
    │   ├── CONTEXT.md
    │   └── docs/adr/                  ← context-specific decisions
    └── billing/
        ├── CONTEXT.md
        └── docs/adr/
```

The skill is explicit that these files are **created lazily** — only
when there is a real term to write down or a real decision to record.

### Five interview moves it adds

| Move | What it does |
|---|---|
| **Challenge against the glossary** | If a term conflicts with `CONTEXT.md`, call it out: *"Your glossary defines 'cancellation' as X, but you seem to mean Y — which is it?"* |
| **Sharpen fuzzy language** | Propose a precise canonical term: *"You're saying 'account' — do you mean Customer or User?"* |
| **Discuss concrete scenarios** | Invent edge cases to force precise boundaries between concepts. |
| **Cross-reference with code** | If the user's claim contradicts the code, surface the gap immediately. |
| **Update `CONTEXT.md` inline** | Capture resolved terms in the glossary the moment they crystallize — not at the end of the session. |

`CONTEXT.md` is treated as a glossary and *only* a glossary —
the skill explicitly forbids using it as a spec, scratchpad, or
implementation log.

### When an ADR is offered

ADRs are gated by a three-criterion rule. Only when **all three** are
true does the skill propose writing one:

1. **Hard to reverse** — changing your mind later is expensive.
2. **Surprising without context** — a future reader will wonder
   *"why did they do it this way?"*.
3. **The result of a real trade-off** — genuine alternatives existed
   and you picked one for specific reasons.

This is the part I find most useful. Without that gate, ADRs accumulate
into noise; with it, every record earns its keep.

### When I reach for it

A working codebase with an existing domain language. The skill turns
plan review into glossary maintenance and decision archaeology in the
same loop — which means each grilling session leaves the repo with
sharper documentation, not just a sharper plan.

---

## How They Compose

The two skills are layered, not parallel:

- `grill-me` is the **interview engine**.
- `grill-with-docs` is the interview engine **plus** a documentation
  contract.

In practice I use `grill-me` for plans that don't yet live inside a
codebase (architecture sketches, research proposals, paper outlines)
and `grill-with-docs` for plans that touch a project where the language
is starting to matter. The bar for switching is whether I would benefit
from `CONTEXT.md` discipline — once the answer is *yes*, I want the
documentation moves the second skill enforces.

---

## Why I Wrote This Up

Both files are short enough to read in two minutes, but the part that
matters is the *gate* each one installs:

- `grill-me` gates against unstated assumptions.
- `grill-with-docs` gates against undocumented decisions.

Most of what makes plan reviews productive is keeping those gates
closed when you're tired, and prompts like these are a low-cost way to
delegate that vigilance.

---

## References

- [`mattpocock/skills` — repository](https://github.com/mattpocock/skills)
- [`grill-me/SKILL.md`](https://github.com/mattpocock/skills/blob/main/skills/productivity/grill-me/SKILL.md)
- [`grill-with-docs/SKILL.md`](https://github.com/mattpocock/skills/blob/main/skills/engineering/grill-with-docs/SKILL.md)

---
layout: post
title: "Implicit Instruction: Teaching Agents Through Accumulated Experience"
date: 2026-05-25 10:00:00 +0900
description: >
  An exploration of how multi-turn agent sessions can accumulate
  "lessons learned" into evolving instructions that improve the agent
  over time — covering the existing landscape (Reflexion, Voyager,
  MemGPT, ExpeL, production memory tools), the emerging "implicit
  instruction" paradigm (ILWS, Atlas, ACE), and a practical skill
  template for implementing it in Claude Code.
tags: [agents, memory, self-improvement, skills, context-engineering]
series: Agent Team Architecture
chapter: 4
toc:
  sidebar: left
---

Every agent session starts from scratch. The model has no memory of
what worked yesterday, which patterns caused failures, or what the
user corrected three sessions ago. Each session reinvents the wheel.

This post maps out how the field is solving that problem — from
academic frameworks to production tools — and converges on a specific
paradigm I'm calling **implicit instruction**: the idea that an
agent's operating instructions should *evolve based on accumulated
experience*, without explicit human annotation. I'll end with a
practical skill template for running this in Claude Code.

This continues the
[Agent Team Architecture series]({% link _explorations/2026-05-06-agent-harness-team.md %}) —
the previous chapters covered harness design, decision frames, and
agent team rules. This chapter adds the *temporal* dimension: how
does the team get better across sessions?

---

## 1. The Landscape — Six Ways Agents Remember

The accumulated work on agent memory falls into a clean taxonomy.
Understanding where each approach sits helps explain why "implicit
instruction" is a distinct move.

### 1.1 Episodic Reflection

The agent reflects on past trials and stores natural-language
insights for retrieval.

| System | Key idea | Result |
|---|---|---|
| **Reflexion** (Shinn et al., NeurIPS 2023, [arXiv:2303.11366](https://arxiv.org/abs/2303.11366)) | Verbal self-reflection after task failure; reflective text stored in episodic buffer | 91% pass@1 on HumanEval |
| **ExpeL** (Zhao et al., AAAI 2024, [arXiv:2308.10144](https://arxiv.org/abs/2308.10144)) | Trial-and-error → natural language insights + successful examples as in-context demos | No parameter updates; works with closed-source models |
| **MARS** (Jan 2026, [arXiv:2601.11974](https://arxiv.org/abs/2601.11974)) | Principle-based + procedural reflection inspired by educational psychology | Outperforms prior self-evolving systems with less compute |
| **ERL** (Mar 2026, [arXiv:2603.24639](https://arxiv.org/abs/2603.24639)) | Reflects on trajectories → generates transferable **heuristics**; retrieves at test time | +7.8% success rate on GAIA2 |

**The pattern:** store *what went wrong and why* as text → retrieve
relevant reflections next time. Clean, interpretable, but the
reflections are **passive context** — they compete for context-window
space and the model has to decide whether to follow them.

### 1.2 Skill Libraries

The agent produces reusable executable artifacts that compound over
time.

| System | Key idea | Result |
|---|---|---|
| **Voyager** (Wang et al., 2023, [arXiv:2305.16291](https://arxiv.org/abs/2305.16291)) | Ever-growing skill library of executable code in Minecraft; skills are compositional | 3.3× more unique items, 15.3× faster milestones |
| **Hermes Agent** (Nous Research, 2026, [GitHub](https://github.com/NousResearch/hermes-agent)) | After each task → writes a reusable Markdown skill file into SQLite; revises if better approach found | 95K+ GitHub stars |
| **CoEvoSkills** (Apr 2026, [arXiv:2604.01687](https://arxiv.org/abs/2604.01687)) | Self-evolving multi-file skill packages via co-evolutionary verification | 32% → 75% pass rate on SkillsBench |

**The pattern:** store *what to do* as code or structured procedures
→ retrieve and execute next time. Powerful for procedural tasks, but
the skills are *action recipes*, not *behavioral adjustments*.

### 1.3 Tiered Memory Architecture

Inspired by OS virtual memory: the agent manages its own
multi-level store.

| System | Key idea |
|---|---|
| **MemGPT / Letta** (Packer et al., 2023–2026, [letta.com](https://www.letta.com/blog/agent-memory)) | Core memory (always in-context, like RAM) + archival/recall memory (out-of-context, like disk); agent does self-directed read/write |
| **Mem0** ([mem0.ai](https://mem0.ai/blog/state-of-ai-agent-memory-2026)) | Bolt-on extract → store → retrieve API for any agent framework |

**The pattern:** give the agent a *storage API* and let it manage
what to keep, evict, and retrieve. General-purpose, but the stored
items are facts/preferences, not *instructions*.

### 1.4 Context Replay

Replay synthesized past experience directly into the current
context.

| System | Key idea | Result |
|---|---|---|
| **CER** (ACL 2025, [arXiv:2506.06698](https://arxiv.org/abs/2506.06698)) | Dynamic memory buffer of synthesized past experiences | 51% relative improvement on WebArena |
| **ReSpect** (Cornell Tech, 2024, [arXiv:2410.13852](https://arxiv.org/abs/2410.13852)) | Learns from *implicit* feedback (rephrasing, frustration); retrospective retraining | 31% → 82% task completion |
| **Trajectory-Informed Memory** (IBM, Feb 2026, [arXiv:2603.10600](https://arxiv.org/abs/2603.10600)) | Extracts strategy / recovery / optimization tips from trajectories | +28.5pp on complex AppWorld tasks |

**The pattern:** distill past sessions into compact representations
→ inject into future sessions as context. Sits between reflection
and skill libraries.

### 1.5 Production Tool Memory

What shipping products actually do.

| Tool | Memory mechanism |
|---|---|
| **Claude Code** ([docs](https://code.claude.com/docs/en/memory)) | 3-tier: user-written `CLAUDE.md` + auto-memory (self-written notes from corrections) + `/compact` session memory. AutoDream consolidates between sessions. |
| **claude-mem** ([GitHub](https://github.com/thedotmack/claude-mem)) | Hooks into session lifecycle → semantic summaries via Claude → SQLite with FTS5 |
| **Cursor** `.cursor/rules/*.mdc` | Static rules; no built-in session-to-session learning |
| **Windsurf** | Auto-generated memories of preferences/patterns; persistent via MCP |
| **OpenAI Codex** | Preview memory: preferences, style, corrections (cloud sessions) |
| **Memorix** ([npm](https://www.npmjs.com/package/memorix)) | Open-source MCP memory with Git truth |

**The pattern:** capture corrections and preferences as they happen
→ persist across sessions. Practical, but most systems store
*declarative* preferences ("use tabs not spaces"), not *behavioral
instructions* ("when you see pattern X, do Y because Z").

### 1.6 Curated Rules

The manual baseline: humans write and maintain instruction files.

| System | What it is |
|---|---|
| `CLAUDE.md` / `.cursorrules` | Hand-written per-project rules |
| **steipete/agent-rules** ([GitHub](https://github.com/steipete/agent-rules)) | Curated cross-tool rule library (5.5K+ stars) |
| **Learnings.md pattern** ([MindStudio](https://www.mindstudio.ai/blog/self-learning-claude-code-skill-learnings-md)) | Skill-scoped file capturing what worked/failed; read at start, updated at end |

**The pattern:** the human is the reflection engine. Thorough but
expensive.

---

## 2. What's Missing — The Instruction Gap

Looking at the six categories, a structural gap becomes visible:

| Category | What it stores | What it optimizes |
|---|---|---|
| Reflection | "what went wrong" | recall |
| Skills | "what to do" (code) | action |
| Tiered memory | facts / preferences | retrieval |
| Context replay | compressed traces | coverage |
| Tool memory | user corrections | personalization |
| Curated rules | human-written instructions | behavior |

The last row — curated rules — is the only one that directly
modifies the agent's **behavioral instructions**. But it requires
the human to write and maintain the rules.

The implicit-instruction paradigm asks: **what if the agent's
instructions were the memory?** Not "store facts and retrieve
them" but "rewrite the instructions themselves based on what
happened."

---

## 3. Implicit Instruction — the Emerging Paradigm

Three recent papers formalize this idea.

### 3.1 ILWS — Instruction-Level Weight Shaping

Costa (Adobe), Sep 2025.
[arXiv:2509.00251](https://arxiv.org/abs/2509.00251).

After each session, an LLM-driven **Reflection Engine** inspects
the conversation trace, diagnoses successes and failures, and
proposes **typed deltas** over instructions, user preferences, and
tools. Each delta is:

- **Version-controlled** — changes are tracked with diffs.
- **Evaluated** — sliding-window analysis of 1–5 star ratings.
- **Auto-repaired on first failure**, rolled back on repeated
  failure.

System instructions are treated as **axiomatic constraints**, not
suggestive context. Evolving them changes downstream behavior more
reliably than retrieving additional context.

**Result:** 2.4–5.0× throughput improvement, ~80% reduction in
hallucinations vs. a frozen baseline in enterprise support.

The critical insight: **instructions are pseudo-parameters.** They
function like tunable weights, but they are text, so they can be
inspected, versioned, and debugged by a human at any point.

### 3.2 Atlas — Compiled Memory

Rhodes & Kang, Mar 2026.
[arXiv:2603.15666](https://arxiv.org/abs/2603.15666).

Atlas is a **memory kernel** that compiles task experience into the
agent's instruction structure. The core distinction from retrieval:

> Memory is **distillation, not storage.** Delivery is
> **instruction rewriting, not context injection.**

Facts extracted from failures and successes pass through a
**three-step promotion gate**: raw observation → candidate insight
→ promoted instruction sub-bullet. Delivery happens by rewriting
the agent's system prompt — the compiled lessons become part of
the instruction, indistinguishable from hand-written ones.

**Result:** +8.7pp F1 on CUAD contract analysis, +3.16pp on
HotpotQA. Transfers across models — compiled from GPT-4o errors,
applied unchanged to Claude Sonnet: +2.31pp.

The transferability finding is the strongest evidence for the
paradigm: if compiled instructions help a *different model*,
the lessons are genuinely portable behavioral knowledge, not
model-specific patches.

### 3.3 ACE — Agentic Context Engineering

Microsoft Research, Oct 2025.
[arXiv:2510.04618](https://arxiv.org/abs/2510.04618) (ICLR 2026).

ACE treats contexts as **evolving playbooks** that accumulate,
refine, and organize strategies through generation, reflection,
and curation. Prevents "brevity bias" (playbooks that converge to
terse, unhelpful rules) and "context collapse" (playbooks that grow
without bound).

**Result:** +10.6% on agent benchmarks, +8.6% on finance tasks.

The contribution is the *lifecycle management*: how to grow
playbooks without them degenerating.

### 3.4 The Common Structure

All three share a loop:

```
session → trace → reflection engine → instruction delta → review gate → updated instructions
                                                                              │
                                                      next session reads ←────┘
```

The differences are in the *review gate* — ILWS uses star-rating
rollback, Atlas uses a three-step promotion gate, ACE uses
anti-collapse heuristics. The pattern is the same: **the
instruction file is the evolving artifact**, and the agent improves
by changing what it is told, not what it remembers.

---

## 4. A Practical Skill for Implicit Instruction

Here's the file structure I use to run this in Claude Code,
adapted from the patterns above.

### 4.1 Directory layout

```
.claude/skills/implicit-instruction/
├── SKILL.md                ← the skill entry point
├── learnings.md            ← accumulated lessons (the "compiled memory")
├── rubric.md               ← scoring criteria for reflection quality
└── templates/
    ├── reflect.md          ← reflection prompt template
    └── compile.md          ← instruction-rewriting template
```

### 4.2 `SKILL.md` — the entry point

```markdown
# Implicit Instruction — Session Learning Loop

## Purpose
Capture lessons from this session and compile them into
actionable instruction updates. Run at session end, or
when a significant correction or failure has occurred.

## Trigger
Invoke with `/implicit-instruction` or run automatically
as a wrap-up step.

## Workflow

### Step 1 — Trace Review
Read the current session's conversation. Identify:
- Corrections the user made ("no, do X instead of Y")
- Failures that required backtracking
- Patterns that worked well on first attempt
- Preferences expressed implicitly (style, structure, tools)

### Step 2 — Reflection
For each finding, write a structured reflection using
the template in `templates/reflect.md`. Each reflection
must include:
- **Signal**: what happened (quote or paraphrase)
- **Lesson**: the generalizable takeaway
- **Scope**: does this apply to this project, this user,
  or universally?
- **Confidence**: high / medium / low

### Step 3 — Promotion Gate
Before adding a lesson to `learnings.md`, check:
1. Is it already covered? → skip or merge
2. Does it contradict an existing lesson? → flag for
   human review, do not auto-resolve
3. Is it inferrable from common sense? → skip (only
   store non-obvious lessons)
4. Has it appeared in ≥2 sessions? → promote to high
   confidence

### Step 4 — Compile
Rewrite the relevant section of `learnings.md` with the
new lessons integrated. Follow the format in
`templates/compile.md`. Lessons are organized by scope:

- **Project-level** — specific to this codebase
- **User-level** — this user's preferences and style
- **Universal** — generalizable behavioral rules

### Step 5 — Diff Review
Show the user the diff of `learnings.md` before
committing. The user may:
- Accept all changes
- Reject specific lessons
- Promote a lesson's scope (project → universal)
- Demote or remove a stale lesson

## Integration
At session start, read `learnings.md` and treat each
entry as a behavioral instruction with the same weight
as `CLAUDE.md`. Lessons are instructions, not suggestions.

## References
- learnings.md — the compiled memory (this directory)
- rubric.md — how to score reflection quality
- templates/reflect.md — per-finding reflection format
- templates/compile.md — learnings.md rewriting rules
```

### 4.3 `templates/reflect.md`

```markdown
## Reflection Template

For each finding from the session trace, fill in:

### Finding [N]

**Signal:**
> [Quote or paraphrase what happened in the session]

**Lesson:**
[One sentence: the generalizable behavioral rule]

**Scope:** [project | user | universal]

**Confidence:** [high | medium | low]
- High: user explicitly corrected this, or pattern appeared 2+ times
- Medium: inferred from one clear instance
- Low: speculative; needs confirmation in future sessions

**Action:**
[How should the agent behave differently next time?
Write as an imperative instruction, not a description.]
```

### 4.4 `templates/compile.md`

```markdown
## Compilation Rules for learnings.md

When integrating new lessons into learnings.md:

1. **Merge, don't append.** If a new lesson refines an existing one,
   rewrite the existing entry. Do not create duplicates.

2. **Contradictions require human review.** If a new lesson
   contradicts an existing one, add a `⚠️ CONFLICT` marker and
   preserve both until the user resolves it.

3. **Demote stale lessons.** If a lesson has not been relevant in
   the last 5 sessions, move it to an `## Archive` section at the
   bottom. Do not delete — the user may want it back.

4. **Format each lesson as an instruction.**
   Bad:  "The user prefers tabs."
   Good: "Use tabs for indentation in all files in this project."

5. **Group by scope.** Maintain three sections:
   - `## Project` — codebase-specific
   - `## User` — personal preferences and style
   - `## Universal` — cross-project behavioral rules

6. **Version stamp.** Add `<!-- updated: YYYY-MM-DD -->` at the top
   of learnings.md after each compilation pass.
```

### 4.5 `rubric.md`

```markdown
## Reflection Quality Rubric

Score each reflection on these criteria before promotion:

| Criterion | Question | Fail condition |
|---|---|---|
| **Actionable** | Can an agent act on this without further context? | Vague ("be more careful") |
| **Non-obvious** | Would a competent developer not already know this? | Common sense ("test before shipping") |
| **Scoped** | Is the scope correctly assigned? | Project-level lesson marked universal |
| **Non-redundant** | Is this genuinely new relative to existing learnings? | Already covered by existing entry |
| **Stable** | Will this still be true next month? | Temporary workaround for a known bug |

A reflection must pass all five to be promoted. Borderline cases
are kept at low confidence and re-evaluated after the next session.
```

### 4.6 `learnings.md` — example initial state

```markdown
<!-- updated: 2026-05-25 -->

## Project

- When editing `_config.yml`, always check `baseurl` — this is a
  GitHub project page (`/log/`), not a user page.
- Explorations use the `_explorations/` collection, not `_posts/`.
  Front matter requires `layout: post` (not `page`).

## User

- Write in English for all blog content.
- Prefer cross-links between related posts using
  `{% link _posts/... %}` syntax.
- When the user says "정리해" they want a structured, formal post,
  not a casual summary.

## Universal

- Before writing a post about a paper, confirm the paper's core
  claim in at least two independent sources. Do not rely on a single
  summary.
- When a URL returns 403, try WebSearch before giving up. Do not
  ask the user to provide content unless WebSearch also fails.

## Archive

(empty)
```

---

## 5. How This Connects

The implicit-instruction loop sits in between two existing
patterns explored on this site:

- **Below**: the
  [rubric harness]({% link _explorations/2026-05-18-when-and-how-to-use-agents.md %})
  evaluates outputs against criteria and co-evolves the rubric. Implicit
  instruction does the same thing but for the *agent's behavioral
  rules* rather than for a specific deliverable.
- **Above**: the
  [Meta-Harness]({% link _posts/2026-05-18-meta-harness.md %}) optimizes
  the entire harness while holding the model fixed. Implicit instruction
  is the *per-session, manual-scale* version of that optimization —
  the harness (instructions) evolves, the model stays the same.

The relationship to ECHO's insight is also worth noting: ECHO
([trend note]({% link _posts/2026-05-18-echo-terminal-world-models.md %}))
showed that agent rollouts contain far more supervision than the
final reward. Implicit instruction applies the same principle at
the *instruction* level: each session contains behavioral signals
(corrections, preferences, failures) that are currently discarded
but can be compiled into better instructions.

---

## 6. What I'm Watching

- **Automated promotion gates.** ILWS's star-rating rollback and
  Atlas's three-step gate are first attempts. The right gate
  probably looks like the rubric harness from
  [chapter 3]({% link _explorations/2026-05-18-when-and-how-to-use-agents.md %}) —
  multi-criterion, Pareto-driven, co-evolving with the lessons
  themselves.
- **Cross-model transfer of compiled instructions.** Atlas showed
  GPT-4o-compiled instructions help Claude Sonnet. If this holds
  broadly, compiled instructions become a *portable asset*
  independent of the model provider.
- **Conflict resolution.** The hardest unsolved problem: when two
  lessons contradict, who wins? Current systems punt to the human.
  An automated resolution mechanism (confidence decay, recency
  weighting, A/B testing across sessions) is the next piece.
- **Integration with AutoDream.** Claude Code's between-session
  consolidation (AutoDream) is a natural host for the compile
  step. If the skill's compilation output can feed into
  `~/.claude/projects/<project>/memory/`, the loop closes inside
  the existing toolchain.

---

## References

### Implicit Instruction Paradigm
- Costa, R. (2025). *Instruction-Level Weight Shaping.*
  [arXiv:2509.00251](https://arxiv.org/abs/2509.00251).
- Rhodes, J. & Kang, G. (2026). *Atlas: Compiled Memory.*
  [arXiv:2603.15666](https://arxiv.org/abs/2603.15666).
- Microsoft Research (2025). *Agentic Context Engineering.*
  [arXiv:2510.04618](https://arxiv.org/abs/2510.04618) (ICLR 2026).
- Tian, Y. et al. (2024). *Enabling LMs to Implicitly Learn
  Self-Improvement (PIT).*
  [arXiv:2310.00898](https://arxiv.org/abs/2310.00898) (ICLR 2024).
- Hewitt, J. et al. (2024). *Instruction Following without
  Instruction Tuning.*
  [arXiv:2409.14254](https://arxiv.org/abs/2409.14254) ·
  [code](https://github.com/john-hewitt/implicit-ins).

### Episodic Reflection
- Shinn, N. et al. (2023). *Reflexion.*
  [arXiv:2303.11366](https://arxiv.org/abs/2303.11366).
- Zhao, A. et al. (2024). *ExpeL.*
  [arXiv:2308.10144](https://arxiv.org/abs/2308.10144).
- [arXiv:2601.11974](https://arxiv.org/abs/2601.11974) (MARS),
  [arXiv:2603.24639](https://arxiv.org/abs/2603.24639) (ERL).

### Skill Libraries & Memory
- Wang, G. et al. (2023). *Voyager.*
  [arXiv:2305.16291](https://arxiv.org/abs/2305.16291).
- Packer, C. et al. (2023). *MemGPT.*
  [letta.com](https://www.letta.com/blog/agent-memory).
- [arXiv:2604.01687](https://arxiv.org/abs/2604.01687) (CoEvoSkills),
  [arXiv:2506.06698](https://arxiv.org/abs/2506.06698) (CER),
  [arXiv:2603.10600](https://arxiv.org/abs/2603.10600) (Trajectory-Informed Memory).

### Production Tools
- [Claude Code memory docs](https://code.claude.com/docs/en/memory),
  [claude-mem](https://github.com/thedotmack/claude-mem),
  [Hermes Agent](https://github.com/NousResearch/hermes-agent),
  [Learnings.md pattern](https://www.mindstudio.ai/blog/self-learning-claude-code-skill-learnings-md).

### Related on this site
- [Agent Team Architecture series]({% link _explorations/2026-05-06-agent-harness-team.md %}),
  [When and How to Use Agents]({% link _explorations/2026-05-18-when-and-how-to-use-agents.md %}),
  [Meta-Harness review]({% link _posts/2026-05-18-meta-harness.md %}),
  [ECHO trend note]({% link _posts/2026-05-18-echo-terminal-world-models.md %}).

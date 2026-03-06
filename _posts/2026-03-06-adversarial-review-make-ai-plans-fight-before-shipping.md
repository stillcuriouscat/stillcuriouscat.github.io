---
title: "Adversarial Review: Make Your AI Agent's Plans Fight Before Shipping"
date: 2026-03-06
excerpt: "AI agent plans always look great on paper — until they hit reality. I designed a three-role adversarial review method (Planner → Critic → Judge) and codified it into a reusable Claude Code Skill."
tags:
  - adversarial-review
  - claude-code
  - ai-agents
  - planning
  - skills
  - methodology
categories:
  - Engineering
toc: true
toc_sticky: true
---

> AI agent plans always look great on paper — until they hit reality. What if you made agents debate each other first?

## What This Post Covers

After multiple rounds of "beautiful plan, disastrous execution," I designed a three-role adversarial plan review method: **Planner proposes → Critic verifies with real commands and rebuts → Judge decides**, iterating until the Critic gives a GO.

I codified this methodology into a reusable Claude Code Skill (`adversarial-review`). This post covers two things:

1. **The pain**: Why AI-generated plans are unreliable when unchallenged
2. **The methodology**: 6 core design principles of adversarial review, and how to turn it into a Skill

---

## Part 1: The Pain — Why AI Plans Keep Failing

### Common Failure Patterns

After working on several system-level projects with Claude Code, I noticed a pattern: **AI plans are always correct on paper, but break when they meet reality.**

Common failure scenarios:
- AI assumes a system resource is available (file path, permission) — it's actually occupied or doesn't exist
- AI introduces tools or rules that conflict with the system's existing configuration, breaking things that were working fine
- Simple requirements get over-engineered — a problem solvable with one script gets a multi-component architecture

The common thread: **AI designs plans based on assumptions without checking the real environment first.**

### Root Causes

Common problems when AI agents generate plans:

| Problem | Manifestation |
|---------|--------------|
| **Unverified assumptions** | Assumes permissions exist, assumes tools are installed |
| **No alternative search** | Designs from scratch without checking for existing tools |
| **Over-engineering** | Piles complex architecture onto simple requirements |
| **Sunk cost fallacy** | Patches broken plans instead of starting over |

One-line summary: **AI is great at generating plans, but terrible at questioning them.**

---

## Part 2: The Methodology — Three-Role Adversarial Review

### Core Idea

> "I need a planning team, an opposing review team, and a judge. Propose a plan, the opposition counters, propose a revised plan, the opposition counters again, the judge coordinates. Iterate like this for several rounds to reach a better solution. Because this is just too complex, and I've been burned by failed plans before."

This borrows from the security world's Red Team / Blue Team model, with one crucial constraint: **the Critic must verify with real commands — no purely theoretical critiques allowed.**

### Three Roles

```
┌─────────┐     PLAN_R1.md      ┌─────────┐
│ Planner │ ──────────────────→ │  Critic │
│ (Agent) │                     │ (Agent) │
└────┬────┘                     └────┬────┘
     │                               │
     │  ← CRITIQUE_R1.md ───────────┘
     │         (NO-GO)
     │                          ┌─────────┐
     └─── Revised PLAN_R2.md ─→│  Judge  │
                                │  (main) │
                                └─────────┘
                                     │
                              verdict + loop mgmt
```

| Role | Who | What They Do |
|------|-----|-------------|
| **Planner** | Agent (subprocess) | Search for existing tools → probe system → write evidence-based plan |
| **Critic** | Agent (subprocess) | Run real commands to verify plan → search community for known pitfalls → write critique |
| **Judge** | Main session | Read both files → issue verdict → manage iteration → detect user annotations |
| **User** | Me | Final authority: review files, annotate directly in the plan |

### 6 Core Design Principles

#### Design 1: Planner Searches for Existing Wheels First

The traditional approach: AI receives requirements → immediately starts designing architecture.

My approach: AI receives requirements → **first searches for existing open-source projects or CLI tools that can be used or adapted** → evaluates whether to use as dependency / extract core logic / borrow patterns → then designs on top of existing foundations.

It's like building with blocks — check if suitable blocks exist before molding from clay.

Equally important: the Planner must **probe the real system environment**. Designing without probing is like drawing blueprints blindfolded.

#### Design 2: Both Sides Must Verify On-Machine + Search

It's not just the Critic who runs commands — **the Planner must also run commands to probe the current system state.** The difference lies in intent:

| | Planner's Investigation | Critic's Investigation |
|---|---|---|
| **Nature** | Exploratory (discover what exists) | Verificatory (check if claims hold) |
| **Bash** | Probe system state, permissions | Verify each claim in the plan |
| **Search** | Search for existing tools, best practices | Search community issues, known pitfalls |
| **Purpose** | Build a model (generate hypotheses) | Break the model (test hypotheses) |

This is fundamentally the scientific method: hypothesis generation vs hypothesis testing.

#### Design 3: User Annotations in Plan Files = BLOCKER

After the Planner writes a plan, I annotate the markdown file directly. These annotations are BLOCKER-level feedback — the Judge must detect them (via diff) and route them back to the Planner before the Critic even reviews.

Why? Automated Critics can find technical bugs, but they can't find domain-knowledge design flaws. For example:
- The plan depends on an upstream component whose status is still undecided — the Critic doesn't know this
- A tool categorizes data by "physical name," but in practice the same physical name maps to different logical meanings — the Critic doesn't understand the business semantics

Only the user can catch these issues. User annotations take priority over any Critic conclusion.

#### Design 4: Severity Levels + Structured Verdict

The Critic's output isn't free-form text — it's structured:

| Level | Code | Meaning | Impact on Verdict |
|-------|------|---------|-------------------|
| CRITICAL | C1, C2... | Must fix | Any present → cannot GO |
| MEDIUM | M1, M2... | Needs decision | Can CONDITIONAL GO |
| LOW | L1, L2... | Code quality | No impact on Verdict |

Every issue must include: description → trigger scenario → **verification evidence (command output)** → proposed fix → cost/tradeoff.

Three possible verdicts:
- **GO**: Ready to implement
- **CONDITIONAL GO**: With a condition table (condition + effort estimate + priority)
- **NO-GO**: Fundamental design problem, needs rewrite

#### Design 5: e2e-harness Integration — Define "What Success Looks Like" First

A plan can't just say "I'll build X" — it must also say "how to verify X works."

I chained adversarial-review with a previously built e2e-harness skill:
- **e2e-harness** defines "what success looks like" (verification plan)
- **adversarial-review** defines "how to achieve success" (implementation plan)

If the project doesn't have e2e harness artifacts yet (`boundary_map.json`, `e2e_features.json`), the skill reminds you to run `/e2e-harness` first. This turns "verification" from an afterthought into a **plan input**.

#### Design 6: No Implementation Until the Plan Is Finalized

This is the simplest but most important rule.

I've had cases where Claude started writing code immediately after the Critic gave a CONDITIONAL GO. The problem: the plan hadn't been finalized by me yet, and the tradeoffs in the conditions hadn't been discussed.

So the skill has a hard rule: **no automatic coding after GO.** The purpose of planning is to align direction, not to auto-trigger implementation.

---

## Part 3: Codifying It as a Skill

The methodology works, but manually orchestrating agents every time, explaining that the Critic must run commands, defining output formats — it's tedious. So I codified it into a Claude Code Skill.

### File Structure

```
~/.claude/skills/adversarial-review/
├── SKILL.md              # Main workflow + graphviz flowchart (~460 words)
├── planner-prompt.md      # Planner agent prompt template
├── critic-prompt.md       # Critic agent prompt template (IRON RULE)
└── artifact-templates.md  # Plan/Critique output format templates
```

Usage: `/adversarial-review <topic>`

### Key Design: The Critic's IRON RULE

The Critic prompt contains one iron law:

```
Pure theoretical critique is WORTHLESS.
Every issue you raise MUST cite:
  (a) a command you ran + its actual output, OR
  (b) a search result (URL + key finding)

If you cannot verify something, write:
  "UNVERIFIABLE: <reason>"
Do NOT pretend to verify.
```

This eliminates vague objections like "this might be a problem." The Critic either proves there's a problem, or stays silent.

### Toolchain

```
/e2e-harness          →  Define "what success looks like"
      ↓
/adversarial-review   →  Define "how to achieve success"
      ↓
Implementation        →  Only after plan is finalized and user confirms
```

---

## Conclusion: When to Use Adversarial Review

| Scenario | Use It? |
|----------|---------|
| Fixing a typo, adding a log | No |
| New feature involving system services/permissions/multiple components | **Yes** |
| A similar approach has failed before | **Yes** |
| You have gut-level distrust of the AI's plan | **Yes** |
| The plan looks too smooth | **Yes** — the smoother it looks, the more you should doubt it |

Core principle: **AI is great at generating plans, but terrible at questioning them. So let another AI do the questioning — with real evidence, not hand-waving.**

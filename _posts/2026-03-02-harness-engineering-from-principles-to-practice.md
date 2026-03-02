---
title: "Harness Engineering: From Principles to Practice"
date: 2026-03-02
excerpt: "When your AI agent writes 'E2E tests' full of mocks, how do you teach it to verify real environmental state? A synthesis of OpenAI, Anthropic, and Mitchell Hashimoto's ideas — plus lessons from encoding these principles into a Claude Code Skill."
tags:
  - harness-engineering
  - e2e-testing
  - claude-code
  - ai-agents
  - evals
  - testing-methodology
  - skills
categories:
  - Engineering
toc: true
toc_sticky: true
---

> When your AI agent writes "E2E tests" full of mocks, how do you teach it to verify real environmental state?

## What This Post Covers

Harness Engineering emerged in late 2025 as a new discipline: **instead of writing better prompts, build better environments**. OpenAI, Anthropic, and independent voices like Mitchell Hashimoto and Martin Fowler contributed foundational ideas from different angles.

While building end-to-end tests for a Linux voice input tool, I synthesized these ideas into a Claude Code Skill (a reusable AI agent instruction template) and iteratively tested it until it reliably guided agents to do the right thing. This post covers three levels:

1. **The core ideas of Harness Engineering** — what each source contributes, and where they converge
2. **My interpretation** — which ideas matter most for hardware-interactive projects
3. **Practice: encoding principles into a Skill** — how to turn methodology into executable agent instructions, and the traps I found along the way

---

## Part 1: Core Ideas of Harness Engineering

### What Is a Harness

Test harnesses aren't new, but 2025 gave them new meaning in the AI agent context: **not testing your code, but building an environment where AI agents can reliably produce correct results**.

Traditional test harnesses ask "is the code correct?" Agent harnesses ask "can the agent consistently succeed in this environment?"

### Three Key Contributions

**OpenAI — Linter as Teacher**

OpenAI's Codex project shipped ~1 million lines of code with 3 engineers + AI agents. Their core insight: custom linter error messages aren't just errors — they're **teaching moments**. Every linter message includes remediation instructions, so when an agent violates an architectural constraint, it immediately learns how to fix it.

This transforms passive constraints ("you can't do this") into active guidance ("you should do this instead, because...").

**Anthropic — Outcome Over Trace + JSON Checklist**

Anthropic contributed two critical ideas across two engineering blog posts:

- **Outcome vs Trace** (from [Demystifying Evals](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)): Evaluate agents by **final environmental state** (does a reservation exist in the database?), not by process (did the agent call the booking API?). Agents frequently discover valid approaches that evaluators didn't anticipate — penalizing creative solutions damages assessment quality.

- **JSON > Markdown** (from [Effective Harnesses](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)): Track feature checklists in JSON rather than Markdown because agents are less likely to accidentally modify or overwrite JSON files. Agents modify only the `passes` field, preventing accidental deletion of requirements.

- **Session Startup Ritual**: Every new session follows a fixed onboarding sequence — confirm working directory, review git logs, read progress files, run baseline tests — preventing agents from reinventing context.

**Mitchell Hashimoto — Every Failure Is a Design Input**

This may be the most powerful single idea. In traditional engineering, a test failure means the code has a bug. In harness engineering, a test failure equally likely means **the framework itself needs improvement** — add logging, add checkpoints, improve documentation. Failures aren't bugs to fix; they're data that drives framework evolution.

### The Convergence

Synthesizing these sources, a consistent pattern emerges:

| Dimension | Traditional Testing | Harness Engineering |
|-----------|-------------------|-------------------|
| Goal | Verify code correctness | Enable agents to work reliably |
| Failure means | Code has a bug | Framework needs improvement |
| Tool's role | Enforce constraints | Teach + guide |
| Progress tracking | Human-written docs | Machine-readable JSON |
| Evaluation criteria | Was the function called? | Did the environment state change? |

---

## Part 2: My Interpretation

### Why "Outcome Over Trace" Is Existential for Hardware Projects

My project is a Linux global voice input tool: press hotkey, record from microphone, transcribe via ASR model, type text into Kitty terminal. The pipeline involves audio hardware, PulseAudio, X11 window system, terminal emulators — all system-level I/O.

The lesson that taught me this principle the hard way: I asked Claude to write "E2E tests." It mocked xdotool, mocked the terminal — unit tests all green. But when I actually ran it, the terminal received nothing. Mocks had hidden hardware routing issues.

**Outcome Over Trace is a survival-level principle in this context:**

- "xdotool key ctrl+v was called" = Trace (the keystroke may never reach the target window)
- "New text appeared in Kitty terminal scrollback" = Outcome (this is what the user actually cares about)

### Two-Layer Testing: Real E2E Is Irreplaceable

Based on this lesson, I split E2E into two layers:

- **L1 Real E2E**: Real hardware — `paplay` plays audio through speakers, microphone picks it up, `arecord` records, ASR transcribes, `kitty @ send-text` types, `kitty @ get-text` verifies scrollback. Few tests (3-5), failure = blocking.
- **L2 Virtual E2E**: Virtual devices — PulseAudio null sink, mock sockets. Many scenarios (10-20+), optional.

L2 cannot replace L1. Virtual devices may mask hardware routing issues (like PulseAudio source/sink configuration). But L2 provides broader coverage. They're complementary.

### Failure-Driven Framework Evolution

Mitchell Hashimoto's principle isn't motivational — it's a concrete engineering method.

A common failure classification goes: test bug → fix the test, missing instrumentation → add logging, environment issue → update pre_checks, application bug → report to developer. Looks structured, but **the "fix the test" branch is extremely dangerous**.

This is exactly the trap I fell into earlier: the agent wrote mock-based "E2E tests" that all passed green, but the terminal received nothing in practice. A passing test doesn't mean the feature works — fixing a test to make it pass just hides the failure. Agents naturally gravitate toward the path of least resistance, and "fix the test" is much easier than "fix the application," so they'll default to modifying tests even when the bug is in the application.

**The correct approach: always suspect the application first, suspect the test last.** Concretely:

1. Read logs/traces for diagnosis (traces are for **diagnosis**, not pass/fail)
2. **Start by assuming it's an application bug** — manually verify whether environmental state matches expectations
3. Only consider the test itself as faulty when you've confirmed the environmental state is correct but the test still reports failure
4. Any test modification must include justification: "because environmental state X is correct, but the test expected Y, and Y is unreasonable because..."

In other words: **the test is the prosecutor, the application is the defendant. When evidence is insufficient, investigate the application more — don't overturn the prosecution's case.**

---

## Part 3: Practice — Encoding Principles into a Skill

### Why Make It a Skill

Principles in a blog post make people nod. But AI agents don't automatically follow them. I needed a way to turn methodology into **executable instructions** that agents follow every time they set up E2E tests.

Claude Code's Skill mechanism solves this: a Markdown file that auto-loads when the agent encounters a matching scenario, with instructions that directly shape agent behavior.

### Skill Structure

```
~/.claude/skills/e2e-harness/SKILL.md (1312 words)

Core Principles:  P1 Outcome Over Trace / P2 Two-Layer / P3 Failures = Design Inputs
                  P4 JSON Checklist / P7 Structural Verification

Execution Flow:   Phase 1 (Analyze I/O boundaries) → 1.5 (User confirmation)
                  → 2 (Research tools) → 3 (Instrument code)
                  → 4 (Generate tests) → 5 (Run → fail → improve → loop)

Common Mistakes:  6 anti-patterns
Tool Catalog:     13 pre-audited tools
```

### Testing the Skill: Using the Weakest Model as a "Unit Test"

After writing the Skill, I tested it with Haiku (the smallest model in the Claude family). The logic: **if the weakest model can follow the instructions, the instructions are clear enough**.

#### Round 1: Baseline vs Skill-Injected

| Dimension | No Skill (Baseline) | With Skill |
|-----------|-------------------|-----------|
| Methodology | Unstructured, jumps to writing tests | Follows Phase 1→1.5→2→3→4→5 |
| P1 compliance | Not mentioned | Verifies scrollback diff, RMS values, file state |
| L1/L2 distinction | None | Clear separation |
| User confirmation | None | Lists 5 confirmation questions |
| boundary_map.json | None | Complete output |

Clear improvement. But I found a problem —

#### The Trap: Trace-Based Verification in L2 Virtual E2E

The skill-guided agent handled L1 Real E2E perfectly, but in L2 Virtual E2E it wrote:

```python
# Agent's L2 verification code
socket_commands = mock_kitty_socket.get_received_commands()  # <-- Trace!
assert len(socket_commands) > 0  # Checking if mock received calls
```

This violates P1! "Mock received a send-text call" is a trace, not an outcome. The correct approach is for the mock to maintain a text buffer and verify the buffer's content.

#### Why the Agent Made This Mistake

Analysis revealed: while Common Mistakes included "don't do trace-based verification," this was a **negative instruction** ("don't do X"). The agent read it, understood it should avoid this, but still slipped into the pattern during code generation — because at the critical moment of generating L2 test code, there wasn't a strong enough **positive instruction** telling it what to do instead.

#### The Fix: Positive Instructions + Self-Check Trigger Words

I added this paragraph to the Phase 4 L2 section:

> Virtual mocks must produce **readable output state**, not just record calls. A mock Kitty socket must maintain a text buffer that `get-text` reads back — verify that buffer, not the list of received commands. If your verify step uses words like "received", "called", "invoked", or "recorded", you are checking traces — rewrite to check state.

Key design choices:
- **Positive instruction**: "mock must maintain a text buffer, verify the buffer" (what to do)
- **Self-check trigger words**: "if your verify step contains received/called/invoked/recorded, you're checking traces" (how to know you're wrong)

This places a mirror in front of the agent at the exact moment it generates code.

#### Round 2: Pass

After the patch, retesting with Haiku produced fully state-based `verify_output()`:

```python
# Agent's L2 verification code after patch
mock_buffer = mock_kitty_socket.get_text_buffer()  # State!
assert expected_text in mock_buffer  # Verifying buffer content
```

The agent even proactively marked `_call_log` as "diagnosis only, NOT for pass/fail" — indicating the instruction was genuinely understood, not just pattern-matched.

### Methodology Summary: How to Write Effective AI Instructions

Principles distilled from this practice:

| Principle | Description | Example |
|-----------|-------------|---------|
| **Dual constraint: negative + positive** | "Don't do X" isn't enough — also say "Do Y" | "Don't check mock calls" + "Check the mock's output buffer" |
| **Self-check trigger words** | Give agents a quick heuristic for self-correction | "If verify contains received/called, it's a trace" |
| **Test with the weakest model** | If Haiku can follow it, the instruction is clear | Haiku subagent A/B comparison with/without Skill |
| **Place instructions at the critical moment** | Don't just state principles at the top — repeat at code generation point | Emphasize P1 in Phase 4 L2 section, not just Principles |

This is fundamentally the same idea as OpenAI's "Linter as Teacher": **don't just tell agents the rules — give correction instructions at the moment they make mistakes**. Linters do this at compile time; Skills do this in the code generation prompt.

---

## Conclusion

The core of Harness Engineering isn't a specific toolset — it's a mindset shift:

- From "testing code" to "building environments where agents can work reliably"
- From "test failure = bug" to "test failure = framework evolution opportunity"
- From "writing prompts" to "encoding principles into environmental constraints"

For projects involving hardware I/O (audio, terminals, GPUs, networks), this mindset is especially critical — because mocks can fool agents, but they can't fool physics.

---

## References

- [OpenAI: Harness Engineering](https://openai.com/index/harness-engineering/) — Linter as teacher, 3 engineers + Codex = 1M lines
- [Anthropic: Effective Harnesses for Long-Running Agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) — JSON checklist, session startup ritual
- [Anthropic: Demystifying Evals for AI Agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) — Outcome vs Trace, grader taxonomy
- [The Emerging "Harness Engineering" Playbook](https://www.ignorance.ai/p/the-emerging-harness-engineering) — Cross-company synthesis
- [Martin Fowler: Harness Engineering](https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html) — Architecture as guardrails
- Mitchell Hashimoto: "Every failure is a design input"
- James Shore: Testing Without Mocks (Nullable Infrastructure)
- Martin Fowler: Domain Probe Pattern

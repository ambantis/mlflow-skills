# Skill Optimization

This document outlines how to use the test framework as the evaluation signal in an
iterative skill optimization loop.

## The Problem

Skills are documents (`SKILL.md`) that instruct an AI coding agent how to behave.
Improving a skill is inherently empirical: you can't know whether an edit made the agent
behave better without running it and measuring the result. The test framework provides
that measurement, but something still needs to close the loop by turning failure
rationales into skill edits.

## Why Standard Optimizers Don't Apply Directly

DSPy and MLflow's built-in optimizers (SimBA, MemAlign, GEPA) optimize prompts inside
Python programs they can instrument. The skill optimization problem has an extra level
of indirection: `SKILL.md` is a document read by a *separate* Claude Code process, not
a prompt in a Python LLM call. Standard frameworks cannot instrument that call.

MLflow's built-in optimizers are useful for a related but distinct problem: **judge
alignment** — tuning the judge prompts so they better match human feedback. If your
judges are producing inaccurate verdicts, those tools are the right fix. They are not
the right tool for improving the skill content itself.

## The Optimization Loop

The practical approach is a custom iterative loop:

```
┌─────────────────────────────────────────────────────┐
│  1. Run test_skill.py                               │
│     → structured judge results (pass/fail + reason) │
└────────────────────┬────────────────────────────────┘
                     │ all pass? → done
                     ▼
┌─────────────────────────────────────────────────────┐
│  2. Feed to Claude:                                 │
│     - current SKILL.md                              │
│     - judge failures with rationales                │
│     → Claude rewrites SKILL.md                      │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│  3. Repeat until all judges pass                    │
│     or max iterations reached                       │
└─────────────────────────────────────────────────────┘
```

The judge rationales are the optimization signal. Because they describe *specifically*
what the agent did wrong (e.g. "the agent did not read the skill file before writing
code"), Claude has enough context to make targeted edits rather than rewriting the skill
from scratch.

## Role of MLflow

Each iteration should be logged as an MLflow run, capturing:

- The skill content at that iteration
- The judge results (pass/fail + rationale per judge)
- The overall pass rate

This gives you a full history of skill versions and their scores, makes it easy to
compare iterations, and lets you roll back to a previous version if an edit regresses
a judge that was previously passing.

## What You Need

**A target agent repo** (`REPO_URL`) — a real codebase that the agent will work on
during each test run. The quality and variety of this repo affects how well the
optimized skill generalizes. A single small repo may produce a skill that is overfit
to that specific codebase.

**Judges that reflect what you care about** — the optimization loop can only improve
what the judges measure. If a judge is too coarse ("did tracing happen?") the loop
converges quickly but the resulting skill may still produce poor output in edge cases.
Investing in precise judges pays off in skill quality.

**A stopping criterion** — 100% judge pass rate is the natural target, but in practice
you may want a max iteration limit (e.g. 10 rounds) and a minimum pass rate threshold
to avoid over-optimizing against a narrow test case.

## Relationship to DSPy

DSPy could drive step 2 (the rewrite step) if you model skill generation as a DSPy
module with the test pass rate as its metric. This gives you more systematic
optimization (e.g. via MIPROv2) at the cost of additional complexity. It is most
worthwhile when you have a large training set of agent repos to optimize against — at
least 10–20 varied examples. With fewer examples, a simple LLM loop produces comparable
results with less infrastructure.

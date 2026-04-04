# auto-optimize

**AutoResearch for Performance Engineering**

> Measure first. Optimize second. Claude does the rest.

Andrej Karpathy introduced the idea of [autoresearch](https://github.com/karpathy/autoresearch) — closing the loop between hypothesis, experiment, and reflection so that an AI agent can drive an entire research cycle autonomously. **auto-optimize applies that idea to performance engineering.**

You define a numeric goal and a success threshold. The plugin builds regression and benchmark infrastructure, locks a baseline, then runs an autonomous loop: profile → hypothesize → apply → test → measure → reflect → repeat. Every iteration is a git commit. Every decision is reasoned and recorded.

---

## The Problem

Most optimization attempts fail silently:

- You change code, *feel* like it's faster, ship it — but never measured before or after
- You write a quick benchmark once, optimize for it, then lose the script
- You try five approaches, forget what you tried, and repeat the same dead-ends

**auto-optimize enforces the discipline you know you should have but don't.**

---

## How It Works

| Phase | What Happens | Output |
|-------|-------------|--------|
| **0. Gather** | Collects goal, scope, metric direction, and numeric success criteria | `experiment-plan.md` |
| **1. Infra** | Builds Regression Test and Benchmark Test scripts if missing (parallel sub-agents) | `tests/` + `bench/` |
| **1.5 Baseline** | Runs Benchmark Test on unmodified code and locks the result | `baseline/result.txt` |
| **2. Loop** | Profiles → Plans (Opus) → Applies → Tests → Benchmarks → Keep/Revert → Reflects | `iterations/` + `leaderboard.md` |
| **3. Report** | Summarizes all iterations, best config, and recommended next steps | `final-report.md` |

Every iteration is a git commit — including reverts. The full experiment history is always recoverable.

---

## Core Principles

**No measurement, no optimization.**
The loop cannot start until both the Regression Test and Benchmark Test commands are confirmed working. Ambiguous success criteria ("as fast as possible") are rejected — a specific number is required.

**Sub-agents isolate context.**
Each iteration runs in a dedicated sub-agent. Raw profiling output, diffs, and test logs never pollute the main context. No `/clear` needed.

**Failure is data.**
Reverted iterations are never deleted. The leaderboard tracks every attempt — what worked, what didn't, and why.

---

## Installation

```
/plugin install auto-optimize
```

Or from source:

```
/plugin install /path/to/auto-optimize
```

---

## Quick Start

```
/auto-optimize
```

Then describe your optimization goal:

```
"The API p99 latency is 340ms, I want to get it under 200ms"
"Reduce the main bundle size by at least 30%"
"Improve test coverage for the auth module from 45% to 80%"
```

Claude will ask up to 4 questions at a time to collect:

| Question | Example Answer |
|----------|---------------|
| What metric to improve? | `p99 latency (ms), lower is better` |
| Which files are in scope? | `backend/app/services/*.py` |
| Overall success target? | `≤ 200ms` |
| Per-iteration minimum gain? | `at least 10ms improvement to keep a change` |
| Regression Test command? | `pytest tests/` or `none` |
| Benchmark Test command? | `python bench.py` or `none` |

If either command is missing, auto-optimize writes it for you before starting the loop.

---

## Example Leaderboard

After a few iterations, `experiments/exp-001-api-latency/leaderboard.md` looks like:

```markdown
# Experiment Leaderboard: exp-001-api-latency
Last updated: 2026-04-04

| Rank | Iteration              | Result | Delta  | Strategy                    | Status     |
|------|------------------------|--------|--------|-----------------------------|------------|
| 1    | iter-003-query-cache   | 178ms  | -162ms | Cache repeated DB lookups   | ✅ Keep 🎯 |
| 2    | iter-001-index-add     | 295ms  | -45ms  | Add composite index         | ✅ Keep    |
| 3    | baseline               | 340ms  | —      | —                           | Baseline   |
| 4    | iter-002-async-io      | 360ms  | +20ms  | Async conversion (backfired)| ❌ Revert  |
```

---

## Experiment Structure

All experiments are tracked under `experiments/` in your project:

```
experiments/
  exp-001-api-latency/
    experiment-plan.md        # Goal, scope, metric, success criteria
    tests/                    # Regression Test scripts (never modified by optimizer)
    bench/                    # Benchmark Test scripts (never modified by optimizer)
    baseline/
      result.txt              # Initial measurement, locked before loop starts
    iterations/
      iter-001-index-add/
        profile-snapshot.txt  # Profiler output for this iteration
        plan.md               # Step-Back + CoT + Pre-mortem reasoning (Opus)
        result.txt            # Benchmark Test output
        summary.md            # Delta vs baseline, strategy summary
        reflexion.md          # What was learned, hints for next iteration
    leaderboard.md            # Ranked results, updated after every iteration
    final-report.md           # Written after loop ends
```

---

## Requirements

- Claude Code with sub-agent support
- Git (all iterations are committed automatically)
- A project with at least one measurable numeric metric

---

## License

MIT

# auto-optimize

**AutoResearch for Performance Engineering**

> Measure first. Reason deep. Reflect. Repeat.

Andrej Karpathy introduced the idea of [autoresearch](https://github.com/karpathy/autoresearch) — closing the loop between hypothesis, experiment, and reflection so that an AI agent can drive an entire research cycle autonomously. **auto-optimize applies that idea to performance engineering.**

You define a numeric goal and a success threshold. The plugin builds regression and benchmark infrastructure, locks a baseline, then runs an autonomous loop: profile → reason → plan → apply → test → measure → reflect → repeat. Every iteration is a git commit. Every decision is reasoned, recorded, and fed back into the next cycle.

## Installation

```bash
claude plugin marketplace add bluuewhale/auto-optimize
claude plugin install auto-optimize@auto-optimize
```

---

## Quick Start

```
/auto-optimize The API is slow. I want to make it faster.
```

auto-optimize will ask a few clarifying questions — metric, scope, success target, and test commands — then take it from there. If benchmarks or regression tests are missing, it writes them before starting the loop.

---

## Real-World Result: 27% Faster Hash Table

> Full write-up: [HashSmith Part 3 — I Automated My Way to a 27% Faster Hash Table](https://bluuewhale.github.io/posts/i-automated-my-way-to-a-27-percent-faster-hash-table/)

[HashSmith](https://github.com/bluuewhale/hash-smith) is an open-source high-performance hash table for the JVM — a SwissTable-style map built around SWAR probing and 8-byte control word groups. After two rounds of manual optimization, the author handed the profiler to auto-optimize.

**One prompt. ~3 hours. No manual intervention.**

```
/auto-optimize I want to optimize the get/put performance of the SwissMap implementation.
```

| | |
|---|---|
| Experiments run | 5 |
| Optimizations landed | 3 |
| Dropped | 2 |
| Improvement vs baseline | **13–32% across all 8 benchmark scenarios** |

### What the agent found

The agent ran 5 experiments autonomously. Three compounding wins, in order:

1. **Tombstone guard** — the probe loop was carrying tombstone-handling logic on a path where tombstones essentially never exist in production. Splitting into two specialized loop bodies eliminated the dead weight. Put path: **-19% to -45%**.

2. **ILP hoisting on the read path** — `emptyMask` was being computed after the key-equality loop, creating a serial dependency. Moving it adjacent to `eqMask` let the CPU's out-of-order engine pipeline both SWAR operations in the same clock cycle. Get path: **-11% to -36%**.

3. A third, smaller improvement compounded on top of both.

None of these required a single line of code written by the author. The structured reasoning pipeline (Step-Back → CoT → Self-Consistency → Pre-mortem) found the tombstone fast path by asking *what is this loop doing that it doesn't need to do?* — a question that wasn't visible in the disassembly alone.

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
| **1.5 Baseline** | Locks noise-floor-validated baseline measurement and environment snapshot | `baseline/` |
| **2. Loop** | Profile → Disassemble → Reason (Opus) → Apply → Test → Benchmark → Reflect | `iterations/` + `leaderboard.md` |
| **3. Report** | Summarizes all iterations, best config, and recommended next steps | `final-report.md` |

Every iteration is a git commit — including reverts. The full experiment history is always recoverable.

---

## The Intelligence Layer

Most AI coding tools apply changes and hope for the best. auto-optimize's inner loop is built differently — each iteration runs a **structured reasoning pipeline** powered by Claude Opus before a single line of code is touched.

### Multi-technique Reasoning per Iteration

Every iteration delegates planning to a dedicated Opus sub-agent that applies four reasoning techniques in sequence:

| Technique | What It Does |
|-----------|-------------|
| **Step-Back** | Abstracts the bottleneck type first (CPU-bound / I/O-bound / Memory / Network) and derives general solution principles before looking at specifics |
| **Chain-of-Thought** | Enumerates at least 3 candidate approaches with explicit trade-off analysis, then argues for the chosen strategy with an expected improvement estimate |
| **Self-Consistency** | Re-evaluates all candidates independently, as if seeing them for the first time — selects only the strategy that wins both evaluations |
| **Pre-mortem** | Assumes the chosen strategy already failed and writes the post-mortem — identifying the most likely failure mode and the early signal that would confirm it |

This isn't prompt engineering for show. It's a guard against the most common failure mode of autonomous agents: **plausible-sounding decisions that haven't been stress-tested.**

### Deep Profiling + Disassembly Analysis

Before planning, the executor sub-agent re-profiles the current codebase to detect whether the bottleneck has shifted from the previous iteration. When source-level analysis is insufficient — or when previous iterations show diminishing returns — it drops into **disassembly analysis**:

- Native code (C/C++/Rust/Go): checks for missed SIMD vectorization, failed inlining, redundant memory ops
- Python: `dis` module bytecode analysis to spot inefficient interpreter paths
- JVM: `javap` bytecode inspection

Findings are annotated directly into `plan.md` with specific instruction or bytecode offsets, giving the planner sub-agent ground-truth evidence rather than guesses.

### Self-Evolving via Reflexion

After every iteration — whether it succeeded or was reverted — the executor writes a `reflexion.md`:

- What was surprising about this iteration's outcome
- Concrete lessons for the next iteration (not vague impressions)
- Promising unexplored directions spotted during execution

The *next* iteration reads this reflexion before profiling or planning. **The agent gets smarter with each cycle**, accumulating a growing body of experiment-specific knowledge that no general-purpose AI has access to.

This is the difference between an agent that runs N experiments and an agent that runs N *increasingly informed* experiments.

---

## Core Principles

**No measurement, no optimization.**
The loop cannot start until both the Regression Test and Benchmark Test commands are confirmed working. Ambiguous success criteria ("as fast as possible") are rejected — a specific number is required. Baseline measurement includes a noise-floor check: if variance across 5 runs exceeds 5%, the noise source must be identified before locking the baseline.

**Plan before every touch.**
No code is changed until a written plan exists in `plan.md`. The plan is produced by a dedicated Opus planner sub-agent using the full reasoning pipeline above — not inlined into the executor's context.

**Sub-agents isolate context.**
Each iteration runs in a dedicated sub-agent. Raw profiling output, disassembly, diffs, and test logs never pollute the main context. The main loop sees only a structured return value: result, delta, decision, and next hint. No `/clear` needed.

**Failure is data.**
Every attempt is recorded — including the ones that didn't make the cut. The leaderboard tracks what worked, what didn't, and why. The reflexion from a failed iteration is often more valuable than the result from a successful one.

---

## Example Leaderboard

After a few iterations, `experiments/exp-001-api-latency/leaderboard.md` looks like:

```markdown
# Experiment Leaderboard: exp-001-api-latency
Last updated: 2026-04-04

| Rank | Iteration              | Result | Delta  | Strategy                    | Status     |
|------|------------------------|--------|--------|-----------------------------|------------|
| 1    | iter-003-query-cache   | 178ms  | -162ms | Cache repeated DB lookups   | ✅ 🎯      |
| 2    | iter-001-index-add     | 295ms  | -45ms  | Add composite index         | ✅          |
| 3    | baseline               | 340ms  | —      | —                           | Baseline   |
| 4    | iter-002-async-io      | 360ms  | +20ms  | Async conversion (backfired)| ❌          |
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
      result.txt              # Noise-floor-validated baseline (median of 5 runs)
      env-snapshot.txt        # OS, runtime, dependency versions at baseline time
    iterations/
      iter-001-index-add/
        profile-snapshot.txt  # Profiler output for this iteration
        disasm-snapshot.txt   # Disassembly output (when triggered)
        plan.md               # Step-Back + CoT + Self-Consistency + Pre-mortem (Opus)
        result.txt            # Benchmark Test output
        summary.md            # Delta vs baseline, strategy summary
        reflexion.md          # What was learned, concrete hints for next iteration
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

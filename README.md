# auto-optimize

A Claude Code plugin for profiling-driven, autonomous code optimization.

## What it does

Activates when you want to improve a **measurable metric** — API latency, bundle size, memory usage, test coverage, throughput, or any numeric target.

The workflow:
1. Collects your goal, scope, and success criteria (specific numbers required)
2. Builds Guard (regression tests) and Verify (benchmark) infrastructure if missing
3. Measures a baseline
4. Runs an autonomous optimization loop using sub-agents — each iteration profiles, plans, applies, guards, verifies, and reflects
5. Produces a final report with a leaderboard of all attempts

**Core principle:** No optimization without measurement. The loop only starts after Guard and Verify commands are confirmed working.

## Installation

```
/plugin install auto-optimize
```

Or from source:

```
/plugin install /path/to/auto-optimize
```

## Usage

Just describe an optimization goal in natural language:

> "API response time is too slow, optimize it"
> "Reduce the bundle size"
> "Improve test coverage for the auth module"

Claude will invoke the skill and guide you through setup.

## Experiment structure

All experiments are tracked under `experiments/` in your project:

```
experiments/
  exp-001-api-latency/
    experiment-plan.md   # Goal, scope, success criteria
    guard/               # Regression tests (never modified by optimizer)
    bench/               # Benchmark scripts (never modified by optimizer)
    baseline/result.txt  # Initial measurement
    iterations/          # Per-iteration: profile, plan, result, reflexion
    leaderboard.md       # Ranked results table
    final-report.md      # Summary after loop ends
```

Every iteration is committed to git — including reverts — so the full experiment history is preserved.

## License

MIT

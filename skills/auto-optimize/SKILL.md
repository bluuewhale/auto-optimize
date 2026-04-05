---
name: auto-optimize
description: Use when optimizing code for a measurable metric — latency, throughput, memory, test coverage, bundle size, or any numeric target. Handles missing tests and benchmarks before starting the optimization loop.
version: 1.0.0
---

# Code Optimization

## Overview

Profiling-driven code optimization workflow. Establish Regression Test and Benchmark Test infrastructure first, then run an autonomous optimization loop to iteratively improve a measurable metric.

**Core principle:** No optimization without measurement. The loop only starts after both the Regression Test and Benchmark Test commands are confirmed working.

---

## Phase 0: Requirements Gathering (BLOCKING)

**Use AskUserQuestion in batches of up to 4. Never ask one question at a time.**

### Batch 1 — Optimization Goal

| # | Header | Question |
|---|--------|----------|
| 1 | `Goal` | What metric do you want to improve? (e.g. API average response time, test coverage, bundle size, memory usage) |
| 2 | `Scope` | Which files are in scope? (e.g. `src/**/*.ts`, `backend/app/services/*.py`) |
| 3 | `Metric` | What is the numeric metric used to judge improvement? (e.g. ms, %, bytes, error count) — also specify whether higher or lower is better. |
| 4 | `SuccessCriteria` | **Provide specific numeric success criteria.** Two values are required: ① **Target** — the overall goal that ends the experiment (e.g. p99 latency ≤ 200ms, bundle size reduced 30% from current) ② **Iteration threshold** — minimum improvement required to keep a change (e.g. at least 5% improvement, at least 10ms reduction). Vague answers like "as fast as possible" are not accepted. |

> **BLOCKING:** If `SuccessCriteria` is answered with vague language ("as fast as possible", "as small as possible"), re-ask until a concrete number is confirmed before proceeding.

### Batch 2 — Validation Infrastructure

| # | Header | Question |
|---|--------|----------|
| 5 | `Iterations` | How many optimization iterations? (Enter a number to stop after N iterations. Leave blank to run until manually stopped. Stops automatically when the target is reached.) |
| 6 | `RegressionTest` | **Do you have tests that catch functional regressions?** If yes, provide the command. (e.g. `pytest tests/`, `npm test`, `none`) |
| 7 | `BenchmarkTest` | **Do you have a benchmark that measures the performance metric?** If yes, provide the command. (e.g. `python bench.py`, `npm run benchmark`, `none`) |
| 8 | `Existing` | If an `experiments/` directory already exists: **① Continue from where it left off** / **② Start fresh** (confirm whether to preserve existing records) |

---

## Phase 0.5: Session Folder & Plan Creation

**Session folder naming convention:** `exp-NNN-<kebab-case-goal>`
- Examples: `exp-001-api-latency`, `exp-002-bundle-size`
- NNN is automatically determined as (number of existing folders under `experiments/`) + 1

```
experiments/
  exp-001-<goal>/           ← session folder (created now)
    experiment-plan.md      ← experiment charter
    tests/                  ← Regression Test scripts (never modified by optimizer)
    bench/                  ← Benchmark Test scripts (never modified by optimizer)
    baseline/               ← created in Phase 1.5
      result.txt
    iterations/             ← populated during Phase 2 loop
    leaderboard.md          ← updated every iteration
    final-report.md         ← written in Phase 3
```

After creating the session folder, write `experiment-plan.md`:

```markdown
# Experiment Plan

## Experiment ID
exp-NNN-<goal>

## Optimization Goal
{Goal description}

## Scope
{Scope paths}

## Metric
- Name: {metric name}
- Direction: {higher is better / lower is better}

## Success Criteria (BINDING — the loop does not start without these numbers)
- Target: {overall experiment exit condition — e.g. p99 latency ≤ 200ms, bundle size 30% reduction}
- Iteration threshold: {minimum gain to keep a change — e.g. at least 5% improvement, at least 10ms reduction}

## Regression Test Command
{command, or "none"}

## Benchmark Test Command
{command}

## Iterations
{N iterations or until manually stopped}

## Strategies to Explore
- (populated during Phase 2 loop)

## Created
{YYYY-MM-DD}
```

Git commit after writing:
```bash
git add experiments/exp-NNN-<goal>/ && git commit -m "experiment(exp-NNN): create session folder and experiment plan"
```

---

## Phase 1: Infrastructure Setup (only when Regression Test or Benchmark Test is missing)

Regression Test and Benchmark Test authoring are independent — run them as **parallel sub-agents**. The main context receives only the final result (command + success flag), not raw code or test output.

### Parallel Sub-agent Rules

| Situation | How to Run |
|-----------|------------|
| Both Regression Test and Benchmark Test missing | Run sub-agent A + sub-agent B simultaneously |
| Only Regression Test missing | Run sub-agent A alone |
| Only Benchmark Test missing | Run sub-agent B alone |

**Sub-agent A — Write Regression Test** (`oh-my-claudecode:executor`):
- Analyze code within Scope and write tests that verify core behavior
- Save to `experiments/exp-NNN-<goal>/tests/`
- Principle: write only tests that mechanically confirm functional equivalence before and after optimization (these files must never be modified by the optimization loop)
- After dry-run, return: `{ ok: true/false, cmd: "pytest ...", error: "..." }`

```
Regression Test command examples:
  Python: pytest experiments/exp-NNN-<goal>/tests/test_<module>.py -v
  Node:   npm test -- --testPathPattern=tests/<module>
  Go:     go test ./experiments/.../tests/... -run TestXxx
```

**Sub-agent B — Write Benchmark Test** (`oh-my-claudecode:executor`):
- Write a script that repeatedly exercises the critical path within Scope and measures the metric
- Save to `experiments/exp-NNN-<goal>/bench/`
- Principle: output exactly one number to stdout, include warm-up
- After dry-run, return: `{ ok: true/false, cmd: "python bench/bench.py | tail -1", output: "83", error: "..." }`

```
Benchmark Test command examples:
  Python: python experiments/exp-NNN-<goal>/bench/bench.py 2>&1 | tail -1
  Node:   node experiments/exp-NNN-<goal>/bench/bench.js 2>&1 | grep 'avg' | awk '{print $2}'
  wrk:    wrk -t2 -c10 -d5s http://localhost:8000/api/v1/endpoint | grep 'Avg Lat' | awk '{print $2}'
```

**Main context after receiving results:**
- Both sub-agents return `ok: true` → proceed to Phase 1.5
- Either returns `ok: false` → re-run only that sub-agent (pass the error message)

### Infrastructure Checklist

Verify the following from sub-agent return values:

- [ ] Regression Test command: exits 0, results are stable
- [ ] Benchmark Test command: outputs exactly one number to stdout, results are stable
- [ ] Regression Test and Benchmark Test files are excluded from Scope (so the optimizer cannot touch them)

---

## Phase 1.5: Baseline Measurement

**Only run if `experiments/exp-NNN-<goal>/baseline/` does not exist.** If it exists, use it as-is.

Immediately after the Regression Test and Benchmark Test dry-runs pass, measure and lock the pre-optimization metric.

### Noise Floor Protocol (run before locking baseline)

Before accepting a baseline number, verify measurement stability:

```bash
# Run benchmark 5 times and inspect variance
for i in 1 2 3 4 5; do <bench_cmd>; done
```

- If the spread is ≤ 5%: use the **median** of 5 runs as the baseline value
- If the spread is > 5%: investigate noise source (background processes, JIT warm-up, GC) and re-run after mitigation
- Record the chosen aggregation method (median / mean / p95) in `baseline/result.txt` header so every iteration uses the same method

```bash
# Example result.txt format
# Aggregation: median of 5 runs
# Raw: 84 83 85 82 83
83
```

### Environment Snapshot

Record the runtime environment alongside the baseline so result differences can be attributed to code changes, not environment drift:

```bash
# Save environment snapshot
{
  echo "=== Date ==="; date;
  echo "=== OS ==="; uname -a;
  echo "=== Runtime ===";
  # Python
  python --version 2>&1 || true;
  # Node
  node --version 2>&1 || true;
  # Go
  go version 2>&1 || true;
  # Rust
  rustc --version 2>&1 || true;
  echo "=== Key dependencies ===";
  # Python: pip freeze | head -20 (or poetry show)
  # Node: cat package.json | grep '"dependencies"' -A 20
  # Rust: cat Cargo.lock | grep '^name' | head -20
} > experiments/exp-NNN-<goal>/baseline/env-snapshot.txt
```

> If the environment snapshot at a later iteration differs from baseline (e.g. runtime version changed), flag the discrepancy in summary.md before drawing conclusions.

```bash
# Run Benchmark Test and save result
<bench_cmd> > experiments/exp-NNN-<goal>/baseline/result.txt
cat experiments/exp-NNN-<goal>/baseline/result.txt
```

Once a number is saved to `baseline/result.txt`, the baseline is locked. Every iteration computes its delta against this value.

Git commit:
```bash
git add experiments/exp-NNN-<goal>/baseline/ && git commit -m "experiment(exp-NNN/baseline): record baseline measurement and env snapshot"
```

---

## Phase 2: Optimization Loop

**Each iteration is fully delegated to an `oh-my-claudecode:executor` sub-agent.**
The main context receives only the result summary returned by the sub-agent — it never sees raw profiling output, code diffs, or test logs. No `/clear` needed — sub-agent lifecycle provides context isolation.

### Main Context Role (Loop Coordinator)

```
For each iteration:

1. Launch executor sub-agent (pass inputs below)
2. Receive return value:
   {
     iter_id:         "iter-003-cache-layer",
     result:          "45ms",
     delta:           "-38ms",
     decision:        "Keep" | "Drop",
     success_reached: true | false,   ← whether success_target was hit
     next_hint:       "cache TTL tuning may have more headroom"
   }
3. Update leaderboard.md with the return value (written directly in main context)
   - If success_reached == true, add "🎯 Target reached" to that iteration's row
4. Evaluate exit conditions:
   a. Specified iteration count reached → Phase 3
   b. User stops manually → Phase 3
   c. Otherwise → launch next iteration sub-agent (regardless of success_reached)
```

### Sub-agent Inputs (main → executor)

```
exp_path:              experiments/exp-NNN-<goal>/
baseline_value:        {number from baseline/result.txt}
test_cmd:              {Regression Test command}
bench_cmd:             {Benchmark Test command}
scope:                 {Scope paths}
metric_direction:      "lower is better" | "higher is better"
success_target:        {overall exit threshold — e.g. "200" (ms), "70" (%)}
iteration_threshold:   {minimum gain to keep a change — e.g. "5%" or "10ms"}
```

### Executor Sub-agent: 10-Step Harness

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ITERATION N — executor sub-agent
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

STEP 1. Analyze past experiment records
  → Run git log {exp_path} to understand change history
  → Read {exp_path}/leaderboard.md — identify tried strategies and success/failure patterns
  → Cross-reference with experiment-plan.md strategies section
  → If a reflexion.md exists from the previous iteration, read it and extract lessons

STEP 2. Profile current state (re-detect hotspots)
  → Run the language-appropriate profiler against the current codebase
  → Check whether the bottleneck shifted after the previous iteration's optimization
  → Save output to {exp_path}/iterations/iter-NNN-<strategy>/profile-snapshot.txt
  → If a new hotspot is found, update the "Strategies to Explore" section in experiment-plan.md

  [Conditional] Disassembly analysis — trigger when:
    (a) A hot function is identified but the source-level cause is unclear, OR
    (b) Previous iterations show diminishing returns and low-level optimization is the next frontier

  → Select the tool for the target language/runtime:
       Native (C/C++/Rust/Go): objdump -d -M intel <binary> | grep -A 30 <function>
                               cargo-asm <crate::module::fn>  (Rust)
                               go tool objdump -s <pkg.fn> <binary>  (Go)
       Python:                 python -c "import dis, <module>; dis.dis(<module>.<fn>)"
       JVM (Java/Kotlin):      javap -c -p <ClassName>
  → What to look for:
       - SIMD/vectorization missing on numeric loops (check for xmm/ymm/zmm registers)
       - Unexpected function call overhead (inlining not triggered)
       - Redundant memory loads/stores inside tight loops
       - Branch-heavy bytecode that could be restructured
  → Save output to {exp_path}/iterations/iter-NNN-<strategy>/disasm-snapshot.txt
  → Record findings as annotations in plan.md (reference specific instructions or bytecode offsets)

STEP 3. Plan the next iteration [planner sub-agent, model=opus]
  → Delegate to oh-my-claudecode:planner (model=opus)
  → Inputs: profile-snapshot.txt path, leaderboard.md contents, experiment-plan.md, previous reflexion.md
  → The sub-agent applies the following reasoning techniques in order and writes plan.md directly:

       [Step-Back] Establish abstract principles first
         - What type of bottleneck is this? (CPU-bound / I/O-bound / Memory / Network)
         - What are the general solution principles for this bottleneck type?

       [CoT] Reason step by step
         1. What is the current bottleneck? (cite STEP 2 profiling evidence)
         2. List at least 3 possible approaches and analyze trade-offs for each
         3. Why is the chosen strategy the most promising for this iteration?
         4. What is the expected improvement and the reasoning behind the estimate?

       [Self-Consistency] Re-evaluate the 3 approaches independently
         - Re-assess each approach as if seeing it for the first time — does the ranking change?
         - Select the strategy that converges across both evaluations

       [Pre-mortem] Assume the chosen strategy failed — write the post-mortem
         - What is the most likely failure scenario?
         - What early signal would indicate failure? (connect to Regression Test failure criteria)

  → Sub-agent return value: { iter_id: "iter-003-cache-layer", strategy_summary: "one-line description" }
  → Executor only needs the iter_id and strategy — the full reasoning is preserved in plan.md

STEP 4. Apply optimization
  → Select and apply a strategy from the optimization strategy list below
  → Modify only files within Scope (never touch tests/ or bench/)

STEP 5. Run Regression Test
  → Execute {test_cmd}
  → On failure: discard changes via git checkout, record reason in plan.md,
                preserve iter-NNN/ directory, return to STEP 3

STEP 6. Git commit
  → git commit -m "experiment(exp-NNN/iter-NNN): {strategy name}"

STEP 7. Run Benchmark Test
  → {bench_cmd} > {exp_path}/iterations/iter-NNN-<strategy>/result.txt
  → Compare against {baseline_value} → compute delta

STEP 8. Write summary.md
  → Write {exp_path}/iterations/iter-NNN-<strategy>/summary.md
  → Include: measured value, delta_vs_baseline, strategy summary

STEP 9. Keep or Drop (must compare against success criteria)
  Decision logic:
    A. Below iteration_threshold → Drop (even a positive delta is dropped if below minimum)
    B. At or above iteration_threshold → Keep
    C. success_target reached → Keep + set success_reached: true in return value

  → Keep (improved): update leaderboard.md, git commit
  → Drop (regressed): discard optimization changes via git checkout
                      never delete iter-NNN/ result records
  → If success_reached: true, add "🎯 SUCCESS: Target reached" to summary.md

STEP 10. Reflexion + return
  → Write {exp_path}/iterations/iter-NNN-<strategy>/reflexion.md
  → Include:
     - What this iteration taught: what was surprising, why it worked or didn't
     - Concrete lessons for the next iteration (no vague impressions)
     - Any promising unexplored directions spotted during this iteration
  → Git commit: git commit -m "experiment(exp-NNN/iter-NNN): record reflexion"
  → Return to main context:
     {
       iter_id:         "iter-NNN-<strategy>",
       result:          "{measured value}",
       delta:           "{delta}",
       decision:        "Keep" | "Drop",
       next_hint:       "{one-line next promising direction from reflexion}"
     }

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Proceed to Phase 3 when the specified iteration count is reached or the user stops the loop.

---

### Optimization Strategy List

#### A. Algorithms & Data Structures
- Replace O(n²) search/sort with O(n log n)
- Eliminate unnecessary intermediate data structures
- Loop fusion

#### B. I/O & Network
- Replace N+1 queries with batch/join
- Add caching layer (in-memory, Redis)
- Switch to async I/O (async/await)
- Tune connection pool size

#### C. Bundle & Static Assets
- Code splitting / lazy imports
- Tree-shake unused dependencies
- Compress and lazy-load images and fonts

#### D. Memory
- Eliminate unnecessary object copies
- Switch to generators/streams for large data
- Cap cache size

#### E. Parallelism
- CPU-bound: multiprocessing / worker threads
- I/O-bound: asyncio.gather / Promise.all

#### F. Compiler Hints & PGO
- Add branch prediction hints: `likely()`/`unlikely()` (C/C++), `[[likely]]` (C++20)
- Mark hot functions: `__attribute__((hot))` / `#[inline(always)]` (Rust)
- Enable Profile-Guided Optimization (PGO): compile with instrumentation, run representative workload, recompile with profile data
- Enable Link-Time Optimization (LTO): `-flto` (GCC/Clang), `lto = true` (Rust Cargo.toml)
- Force inlining of small frequently-called functions
- Note: verify effect via disassembly analysis (STEP 2) after applying

---

### leaderboard.md Format

```markdown
# Experiment Leaderboard: exp-NNN-<goal>
Last updated: YYYY-MM-DD

| Rank | Iteration | Result | Delta | Strategy | Status |
|------|-----------|--------|-------|----------|--------|
| 1 | iter-003-cache-layer | 45ms | -38ms | Cache query results | ✅ Keep |
| 2 | baseline | 83ms | — | — | Baseline |
| 3 | iter-001-index-add | 80ms | -3ms | Add DB index | ✅ Keep |
| 4 | iter-002-async-io | 90ms | +7ms | Async conversion (no effect) | ❌ Drop |
```

---

## Phase 3: Final Report

After the optimization loop ends, write `experiments/exp-NNN-<goal>/final-report.md`:

1. **Best config summary**: full diff of the highest-performing iteration
2. **Experiment comparison table**: all iteration results based on leaderboard.md
3. **Performance analysis**: improvement over baseline, whether the final target was reached
4. **Insights**: which strategies worked and why
5. **Recommended next steps**: unexplored promising strategies, areas with remaining optimization headroom

Git commit:
```bash
git commit -m "docs(exp-NNN): write final optimization report"
```

---

## Flow Summary

```
Phase 0:   Gather requirements (AskUserQuestion × 2 batches, including Existing question)
    ↓
Phase 0.5: Create session folder + write experiment-plan.md + git commit
    ↓
Phase 1a:  Regression Test missing? → write tests in tests/ → confirm dry-run passes
    ↓
Phase 1b:  Benchmark Test missing? → write benchmark in bench/ → confirm dry-run passes
    ↓
Phase 1.5: Save initial Benchmark Test result to baseline/ + git commit
    ↓
Phase 2:   Optimization loop (executor sub-agent × N iterations)
           [main context] launch executor sub-agent → receive return value → update leaderboard.md
           [executor sub-agent internals]
             STEP 1:  Analyze past records (leaderboard.md + git log)
             STEP 2:  Re-profile current state → re-detect hotspots → [conditional] disassembly analysis
             STEP 3:  planner sub-agent (opus) → write plan.md
             STEP 4:  Apply optimization
             STEP 5:  Run Regression Test (fail → Drop → STEP 3)
             STEP 6:  Git commit
             STEP 7:  Run Benchmark Test → result.txt
             STEP 8:  Write summary.md
             STEP 9:  Keep or Drop
             STEP 10: Write reflexion.md + git commit → return to main
    ↓
Phase 3:   Write final-report.md + git commit
```

## Guardrails

- Do not modify any code before Phase 0 and experiment-plan.md are complete
- If Regression Test or Benchmark Test dry-run fails, fix it and re-confirm — do not start the loop in an unstable state
- `tests/` and `bench/` are never in scope for the optimization loop — exclude them explicitly
- Iteration result directories (`iter-NNN/`) are never deleted, even after a Drop — failed attempts are data
- leaderboard.md is updated by the main context after every executor sub-agent return
- `/clear` is no longer needed — sub-agent lifecycle handles context isolation
- If an executor sub-agent fails or is interrupted, restore state from leaderboard.md and git log, then re-run from that iteration

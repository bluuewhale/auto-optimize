---
name: auto-optimize
description: Use when optimizing code for a measurable metric — latency, throughput, memory, test coverage, bundle size, or any numeric target. Handles missing tests and benchmarks before starting the optimization loop.
version: 1.0.0
---

# Code Optimization

## Overview

정밀 프로파일링 기반의 코드 최적화 워크플로우. 테스트·벤치마크 인프라를 먼저 갖추고, 자율 최적화 루프로 지표를 반복 개선한다.

**핵심 원칙:** 측정 없이 최적화하지 않는다. Guard(회귀 방지)와 Verify(지표 측정) 명령이 실제로 동작하는 것을 확인한 뒤에만 최적화 루프를 시작한다.

---

## Phase 0: 요구사항 수집 (BLOCKING)

**AskUserQuestion을 한 번에 최대 4개씩 묶어서 질문한다. 절대 한 번에 한 개씩 묻지 않는다.**

### Batch 1 — 최적화 목표

| # | 헤더 | 질문 |
|---|------|------|
| 1 | `Goal` | 어떤 지표를 개선하고 싶으신가요? (예: API 평균 응답시간 단축, 테스트 커버리지 향상, 번들 사이즈 감소, 메모리 사용량 절감) |
| 2 | `Scope` | 최적화 대상 파일 범위를 알려주세요. (예: `src/**/*.ts`, `backend/app/services/*.py`) |
| 3 | `Metric` | 개선 여부를 판단할 숫자 지표는 무엇인가요? (예: ms, %, bytes, errors count) — 높을수록 좋은지, 낮을수록 좋은지도 함께 알려주세요. |
| 4 | `SuccessCriteria` | **이 최적화 실험의 성공 기준을 구체적인 수치로 알려주세요.** 두 가지를 명확히 해주세요: ① **목표값** — 실험 전체를 종료할 달성 목표 (예: p99 latency ≤ 200ms, 번들 사이즈 현재 대비 30% 감소) ② **iteration 최소 임계값** — 각 iteration의 변경을 Keep하기 위한 최소 개선 폭 (예: 최소 5% 이상 개선, 최소 10ms 단축). 수치가 없으면 진행하지 않습니다. |

> **BLOCKING:** `SuccessCriteria`가 구체적인 수치 없이 "최대한 빠르게", "가능한 한 작게" 같은 모호한 표현으로 답변되면, 반드시 재질문해 수치를 확정한 뒤에만 다음 단계로 진행한다.

### Batch 2 — 검증 인프라 현황

| # | 헤더 | 질문 |
|---|------|------|
| 5 | `Iterations` | 최적화를 몇 번 반복할까요? (숫자 입력 시 해당 횟수만 실행 후 종료. 비워두면 수동으로 중단할 때까지 무한 실행. 목표값 달성 시 자동 종료됨) |
| 6 | `Guard` | **기능 회귀를 잡아줄 테스트 코드가 있나요?** 있다면 실행 명령을 알려주세요. (예: `pytest tests/`, `npm test`, `없음`) |
| 7 | `Verify` | **성능 지표를 측정하는 벤치마크 코드가 있나요?** 있다면 실행 명령을 알려주세요. (예: `python bench.py`, `npm run benchmark`, `없음`) |
| 8 | `Existing` | `experiments/` 디렉토리가 이미 존재하면: **① 이어서 진행** / **② 새로 시작** (기존 기록 보존 여부도 확인) |

---

## Phase 0.5: 실험 세션 폴더 & 계획서 생성

**실험 세션 폴더 이름 규칙:** `exp-NNN-<kebab-case-goal>`
- 예: `exp-001-api-latency`, `exp-002-bundle-size`
- NNN은 `experiments/` 하위 기존 폴더 수 + 1로 자동 결정

```
experiments/
  exp-001-<goal>/        ← 이번 세션 폴더 (지금 생성)
    experiment-plan.md   ← 실험 헌장
    guard/               ← Guard 스크립트 위치 (최적화 루프에서 수정 금지)
    bench/               ← Verify 스크립트 위치 (최적화 루프에서 수정 금지)
    baseline/            ← Phase 1.5에서 생성
      result.txt
    iterations/          ← Phase 2 루프에서 채워짐
    leaderboard.md       ← 매 iteration 업데이트
    final-report.md      ← Phase 3에서 작성
```

세션 폴더 생성 후 `experiment-plan.md`를 작성한다:

```markdown
# 실험 계획서

## 실험 ID
exp-NNN-<goal>

## 최적화 목표
{Goal 설명}

## 대상 파일 범위
{Scope}

## 지표
- 이름: {metric name}
- 방향: {higher is better / lower is better}

## 성공 기준 (BINDING — 이 수치 없이 루프를 시작하지 않는다)
- 목표값: {실험 전체 종료 기준 — 예: p99 latency ≤ 200ms, 번들 사이즈 30% 감소}
- iteration 최소 임계값: {각 Keep 판단 기준 — 예: 최소 5% 개선, 최소 10ms 단축}

## Guard 명령
{Guard 명령, 없으면 "없음"}

## Verify 명령
{Verify 명령}

## 반복 횟수
{N회 또는 수동 중단}

## 탐색 예정 전략
- (Phase 2 루프에서 채워짐)

## 생성일
{YYYY-MM-DD}
```

작성 후 Git 커밋:
```bash
git add experiments/exp-NNN-<goal>/ && git commit -m "experiment(exp-NNN): 실험 세션 생성 및 계획서 작성"
```

---

## Phase 1: 인프라 설정 (Guard·Verify 없을 때만)

Guard와 Verify 작성은 서로 독립적이므로 **병렬 sub-agent**로 실행한다. main context는 raw 코드·테스트 출력에 오염되지 않고 최종 결과(명령 + 성공 여부)만 수신한다.

### 병렬 sub-agent 실행 규칙

| 상황 | 실행 방법 |
|------|-----------|
| Guard·Verify 둘 다 없음 | sub-agent A + sub-agent B 동시 실행 |
| Guard만 없음 | sub-agent A 단독 실행 |
| Verify만 없음 | sub-agent B 단독 실행 |

**sub-agent A — Guard 작성** (`oh-my-claudecode:executor`):
- Scope 내 코드를 분석하여 핵심 동작을 검증하는 테스트 작성
- `experiments/exp-NNN-<goal>/guard/` 에 저장
- 원칙: 최적화 전후 기능 동일성을 기계적으로 확인할 수 있는 테스트만 작성 (이 파일은 루프에서 절대 수정 금지)
- dry-run 실행 후 반환값: `{ ok: true/false, cmd: "pytest ...", error: "..." }`

```
Guard 명령 확정 예시:
  Python: pytest experiments/exp-NNN-<goal>/guard/test_<module>.py -v
  Node:   npm test -- --testPathPattern=guard/<module>
  Go:     go test ./experiments/.../guard/... -run TestXxx
```

**sub-agent B — Verify 작성** (`oh-my-claudecode:executor`):
- Scope 내 핵심 경로를 반복 실행하며 지표를 측정하는 스크립트 작성
- `experiments/exp-NNN-<goal>/bench/` 에 저장
- 원칙: stdout에 숫자 하나만 출력, warm-up 포함
- dry-run 실행 후 반환값: `{ ok: true/false, cmd: "python bench/bench.py | tail -1", output: "83", error: "..." }`

```
Verify 명령 확정 예시:
  Python: python experiments/exp-NNN-<goal>/bench/bench.py 2>&1 | tail -1
  Node:   node experiments/exp-NNN-<goal>/bench/bench.js 2>&1 | grep 'avg' | awk '{print $2}'
  wrk:    wrk -t2 -c10 -d5s http://localhost:8000/api/v1/endpoint | grep 'Avg Lat' | awk '{print $2}'
```

**main context 수신 후 처리:**
- 두 sub-agent 모두 `ok: true` → Phase 1.5로 진행
- 어느 하나라도 `ok: false` → 해당 sub-agent만 재실행 (에러 메시지 전달)

### 인프라 확인 체크리스트

sub-agent 반환값 기반으로 아래를 확인한다:

- [ ] Guard 명령: exit 0, 결과 안정적
- [ ] Verify 명령: 숫자 하나 stdout 출력, 결과 안정적
- [ ] Guard·Verify 파일이 Scope에서 제외되어 있음 (최적화 루프가 건드리지 못하도록)

---

## Phase 1.5: Baseline 측정

**`experiments/exp-NNN-<goal>/baseline/`가 없을 때만 실행.** 이미 있으면 기존 baseline을 기준으로 사용.

Guard·Verify dry-run이 통과된 직후, 최적화 전 상태의 지표를 측정하고 고정한다.

```bash
# Verify 명령 실행 후 결과를 저장
<verify_command> > experiments/exp-NNN-<goal>/baseline/result.txt
cat experiments/exp-NNN-<goal>/baseline/result.txt
```

`baseline/result.txt`에 숫자가 저장되면 완료. 이후 모든 iteration은 이 값과 delta를 비교한다.

Git 커밋:
```bash
git add experiments/exp-NNN-<goal>/baseline/ && git commit -m "experiment(exp-NNN/baseline): 베이스라인 측정 기록"
```

---

## Phase 2: 최적화 루프

**각 iteration은 `oh-my-claudecode:executor` sub-agent에 완전히 위임한다.**
main context는 프로파일링 raw output·코드 diff·테스트 로그에 오염되지 않고, sub-agent가 반환한 결과 요약만 수신한다. `/clear`가 필요 없다 — sub-agent 생명주기가 컨텍스트 격리를 대신한다.

### main context 역할 (루프 진행자)

```
iteration마다:

1. executor sub-agent 실행 (아래 입력 전달)
2. 반환값 수신:
   {
     iter_id:        "iter-003-cache-layer",
     result:         "45ms",
     delta:          "-38ms",
     decision:       "Keep" | "Revert",
     success_reached: true | false,   ← success_target 달성 여부
     next_hint:      "캐시 TTL 추가 튜닝 가능성 있음"
   }
3. leaderboard.md를 반환값으로 업데이트 (main context에서 직접 작성)
   - success_reached == true이면 leaderboard의 해당 iteration 행에 "🎯 목표 달성" 표시 추가
4. 종료 조건 평가:
   a. 지정 횟수 도달 → Phase 3
   b. 사용자 중단 → Phase 3
   c. 아니면 → 다음 iteration sub-agent 실행 (목표 달성 여부와 무관하게 계속)
```

### sub-agent 입력 (main → executor)

```
exp_path:              experiments/exp-NNN-<goal>/
baseline_value:        {baseline/result.txt 의 숫자}
guard_cmd:             {Guard 명령}
verify_cmd:            {Verify 명령}
scope:                 {Scope 경로}
metric_direction:      "lower is better" | "higher is better"
success_target:        {실험 전체 종료 기준 수치 — 예: "200" (ms), "70" (%)}
iteration_threshold:   {iteration Keep 최소 개선 폭 — 예: "5%" 또는 "10ms"}
```

### executor sub-agent 내부 10단계 하네스

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ITERATION N — executor sub-agent 실행
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

STEP 1. 과거 실험 기록 분석
  → git log {exp_path} 로 변경 이력 파악
  → {exp_path}/leaderboard.md 읽어 시도한 전략·성공/실패 패턴 확인
  → experiment-plan.md의 탐색 예정 전략과 대조
  → 이전 iteration의 reflexion.md가 있으면 함께 읽어 교훈을 파악한다

STEP 2. 현재 상태 프로파일링 (핫스팟 재탐지)
  → 언어에 맞는 프로파일링 명령으로 현재 코드베이스를 다시 실행
  → 이전 iteration의 최적화 후 병목이 이동했는지 확인
  → 결과를 {exp_path}/iterations/iter-NNN-<strategy>/profile-snapshot.txt 에 저장
  → 새로운 핫스팟이 발견되면 experiment-plan.md 탐색 전략 섹션 업데이트

STEP 3. 다음 iteration 계획 수립 [planner sub-agent, model=opus]
  → oh-my-claudecode:planner (model=opus) sub-agent에 위임
  → 입력: profile-snapshot.txt 경로, leaderboard.md 내용, experiment-plan.md, 이전 reflexion.md
  → sub-agent가 아래 추론 기법을 순서대로 적용해 전략을 도출하고 plan.md를 직접 작성:

       [Step-Back] 먼저 추상 원칙을 확립한다
         - 이 병목은 어떤 유형인가? (CPU-bound / I/O-bound / Memory / Network)
         - 이 유형의 병목에 대한 일반적 해법 원칙은 무엇인가?

       [CoT] 단계적으로 추론한다
         1. 현재 병목은 무엇인가? (STEP 2 프로파일링 결과 근거)
         2. 가능한 접근법을 3가지 이상 열거하고 각각의 trade-off를 분석한다
         3. 왜 이 iteration에서 해당 전략이 가장 유망한가?
         4. 예상 개선 폭과 근거는 무엇인가?

       [Self-Consistency] 위 3가지 접근법을 독립적으로 재평가한다
         - 각 접근법을 처음 보는 것처음 보는 것처럼 다시 평가했을 때 순위가 바뀌는가?
         - 두 번의 평가에서 수렴하는 전략을 최종 선택한다

       [Pre-mortem] 선택한 전략이 실패했다고 가정하고 작성한다
         - 실패 원인으로 가장 가능성 높은 시나리오는?
         - 실패를 조기에 감지할 수 있는 시그널은? (Guard 실패 기준과 연결)

  → sub-agent 반환값: { iter_id: "iter-003-cache-layer", strategy_summary: "한줄 설명" }
  → executor sub-agent는 반환된 iter_id와 전략만 파악하면 됨 (추론 과정은 plan.md에 보존)

STEP 4. 최적화 적용
  → 아래 최적화 전략 목록에서 선택 적용
  → Scope 내 파일만 수정 (guard/, bench/ 파일 절대 수정 금지)

STEP 5. Guard 실행
  → {guard_cmd} 실행
  → 실패 시: git checkout으로 변경 사항 되돌리기, 이유를 plan.md에 기록,
             iter-NNN/ 디렉토리는 보존, STEP 3으로 돌아가기

STEP 6. Git 커밋
  → git commit -m "experiment(exp-NNN/iter-NNN): {전략 이름}"

STEP 7. Verify 실행
  → {verify_cmd} > {exp_path}/iterations/iter-NNN-<strategy>/result.txt
  → {baseline_value}와 비교 → delta 계산

STEP 8. summary.md 작성
  → {exp_path}/iterations/iter-NNN-<strategy>/summary.md 작성
  → 포함 내용: 측정값, delta_vs_baseline, 전략 요약

STEP 9. Keep or Revert (성공 기준 대조 필수)
  판단 로직:
    A. iteration_threshold 미달 → Revert (개선이 있어도 최소 임계값 미달이면 되돌림)
    B. iteration_threshold 이상 개선 → Keep
    C. success_target 달성 → Keep + success_reached: true 플래그를 반환값에 포함

  → Keep  (개선): leaderboard.md 업데이트, Git 커밋
  → Revert(퇴보): git checkout으로 최적화 변경 사항 되돌리기
                  단, iter-NNN/ 결과 기록은 절대 삭제하지 않음
  → success_reached: true일 경우 summary.md에 "🎯 SUCCESS: 목표값 달성" 명시

STEP 10. Reflexion + 반환
  → {exp_path}/iterations/iter-NNN-<strategy>/reflexion.md 작성
  → 포함 내용:
     - 이번 iteration에서 배운 것: 예상과 달랐던 점, 왜 효과가 있었는가/없었는가
     - 다음 iteration에 반영할 구체적 교훈 (추상적 감상 금지)
     - 탐색하지 않은 유망한 방향이 보였다면 기록
  → Git 커밋: git commit -m "experiment(exp-NNN/iter-NNN): reflexion 기록"
  → main context에 반환:
     {
       iter_id:   "iter-NNN-<strategy>",
       result:    "{측정값}",
       delta:     "{delta}",
       decision:  "Keep" | "Revert",
       next_hint: "{reflexion에서 발견한 다음 유망 방향 한줄}"
     }

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

지정 횟수 도달 또는 사용자 중단 시 → Phase 3으로 진행.

---

### 최적화 전략 목록

#### A. 알고리즘 & 자료구조
- O(n²) → O(n log n) 탐색/정렬 교체
- 불필요한 중간 자료구조 제거
- 루프 합치기 (loop fusion)

#### B. I/O & 네트워크
- N+1 쿼리 → batch/join으로 교체
- 캐싱 레이어 추가 (in-memory, Redis)
- 비동기 I/O 전환 (async/await)
- 커넥션 풀 크기 조정

#### C. 번들 & 정적 자산
- 코드 스플리팅 / lazy import
- Tree-shaking 불필요 의존성 제거
- 이미지·폰트 압축 및 lazy loading

#### D. 메모리
- 불필요한 객체 복사 제거
- 제너레이터/스트림으로 전환 (대용량 데이터)
- 캐시 크기 상한 설정

#### E. 병렬화
- CPU-bound: 멀티프로세싱 / 워커 스레드
- I/O-bound: asyncio.gather / Promise.all

---

### leaderboard.md 형식

```markdown
# 실험 리더보드: exp-NNN-<goal>
마지막 업데이트: YYYY-MM-DD

| 순위 | Iteration | 측정값 | delta | 전략 요약 | 상태 |
|------|-----------|--------|-------|-----------|------|
| 1 | iter-003-cache-layer | 45ms | -38ms | 쿼리 결과 캐싱 추가 | ✅ Keep |
| 2 | baseline | 83ms | — | — | 기준 |
| 3 | iter-001-index-add | 80ms | -3ms | DB 인덱스 추가 | ✅ Keep |
| 4 | iter-002-async-io | 90ms | +7ms | async 전환 (효과 없음) | ❌ Revert |
```

---

## Phase 3: 최종 리포트

최적화 루프 종료 후 `experiments/exp-NNN-<goal>/final-report.md` 작성:

1. **Best config 요약**: 최고 성과 iteration의 변경 사항 전체 출력
2. **실험 비교표**: leaderboard.md 기반 전체 iteration 결과 비교
3. **성능 분석**: baseline 대비 개선량, 최종 지표 달성 여부
4. **인사이트**: 효과적이었던 전략과 그 이유
5. **권장 다음 단계**: 미탐색 유망 전략, 추가 최적화 가능 영역

Git 커밋:
```bash
git commit -m "docs(exp-NNN): 최종 최적화 리포트 작성"
```

---

## 흐름 요약

```
Phase 0:   요구사항 수집 (AskUserQuestion × 2 배치, Existing 질문 포함)
    ↓
Phase 0.5: 실험 세션 폴더 생성 + experiment-plan.md 작성 + Git 커밋
    ↓
Phase 1a:  Guard 없음? → guard/ 에 테스트 코드 작성 → dry-run 확인
    ↓
Phase 1b:  Verify 없음? → bench/ 에 벤치마크 코드 작성 → dry-run 확인
    ↓
Phase 1.5: baseline/ 에 초기 Verify 결과 저장 + Git 커밋
    ↓
Phase 2:   최적화 루프 (executor sub-agent × N iterations)
           [main context] executor sub-agent 실행 → 반환값 수신 → leaderboard.md 업데이트
           [executor sub-agent 내부]
             STEP 1: 과거 기록 분석 (leaderboard.md + git log)
             STEP 2: 현재 상태 재프로파일링 → 핫스팟 재탐지
             STEP 3: planner sub-agent (opus) → plan.md 작성
             STEP 4: 최적화 적용
             STEP 5: Guard 실행 (실패 시 Revert → STEP 3)
             STEP 6: Git 커밋
             STEP 7: Verify 실행 → result.txt
             STEP 8: summary.md 작성
             STEP 9: Keep or Revert
             STEP 10: Reflexion → reflexion.md + Git 커밋 → main에 반환
    ↓
Phase 3:   final-report.md 작성 + Git 커밋
```

## 주의사항

- Phase 0 + experiment-plan.md 작성 완료 전에는 어떤 코드도 수정하지 않는다
- Guard·Verify dry-run이 실패하면 고친 뒤 재확인 — 불안정한 상태로 루프를 시작하지 않는다
- Guard·Verify 파일(guard/, bench/)은 최적화 루프에서 절대 수정 대상이 아니다 — Scope에서 명시적으로 제외
- 실험 결과(iter-NNN/)는 Revert 후에도 절대 삭제하지 않는다 — 실패 기록도 자산이다
- leaderboard.md는 매 iteration executor sub-agent 반환 후 main context에서 업데이트한다
- `/clear`는 더 이상 사용하지 않는다 — sub-agent 생명주기가 컨텍스트 격리를 담당한다
- executor sub-agent가 실패·중단된 경우, leaderboard.md와 git log로 상태를 복원한 뒤 해당 iteration부터 재실행한다

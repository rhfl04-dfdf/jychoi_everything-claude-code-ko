---
name: autonomous-loops
description: "Claude Code 자율 루프를 위한 패턴 및 아키텍처 — 단순 순차 파이프라인부터 RFC 기반 멀티 에이전트 DAG 시스템까지."
origin: ECC
---

# Autonomous Loops 스킬

> 호환성 노트 (v1.8.0): `autonomous-loops`는 한 릴리스 동안 유지됩니다.
> 표준 스킬 이름은 이제 `continuous-agent-loop`입니다. 새로운 루프 가이드는
> 거기에 작성해야 하며, 이 스킬은 기존 워크플로우를 중단하지 않기 위해 계속 사용할 수 있습니다.

Claude Code를 루프에서 자율적으로 실행하기 위한 패턴, 아키텍처 및 참고 구현체. 단순 `claude -p` 파이프라인부터 완전한 RFC 기반 멀티 에이전트 DAG 오케스트레이션까지 모든 것을 다룹니다.

## 사용 시점

- 사람의 개입 없이 실행되는 자율 개발 워크플로우 설정
- 문제에 맞는 올바른 루프 아키텍처 선택 (단순 vs 복잡)
- CI/CD 스타일의 지속적 개발 파이프라인 구축
- 병합 조정을 통한 병렬 에이전트 실행
- 루프 반복 간 컨텍스트 지속성 구현
- 자율 워크플로우에 품질 게이트 및 클린업 패스 추가

## 루프 패턴 스펙트럼

가장 단순한 것부터 가장 정교한 것 순서:

| 패턴 | 복잡도 | 최적 용도 |
|---------|-----------|----------|
| [순차 파이프라인](#1-순차-파이프라인-claude--p) | 낮음 | 일일 개발 단계, 스크립트화된 워크플로우 |
| [NanoClaw REPL](#2-nanoclaw-repl) | 낮음 | 인터랙티브 지속 세션 |
| [Infinite Agentic Loop](#3-infinite-agentic-loop) | 중간 | 병렬 콘텐츠 생성, 사양 기반 작업 |
| [Continuous Claude PR Loop](#4-continuous-claude-pr-loop) | 중간 | CI 게이트를 포함한 여러 날에 걸친 반복 프로젝트 |
| [De-Sloppify 패턴](#5-the-de-sloppify-패턴) | 추가 기능 | 구현 단계 후 품질 클린업 |
| [Ralphinho / RFC 기반 DAG](#6-ralphinho--rfc-기반-dag-오케스트레이션) | 높음 | 대규모 기능, 병합 큐가 있는 멀티 유닛 병렬 작업 |

---

## 1. 순차 파이프라인 (`claude -p`)

**가장 단순한 루프.** 일일 개발을 일련의 비인터랙티브 `claude -p` 호출 순서로 분리하세요. 각 호출은 명확한 프롬프트가 있는 집중된 단계입니다.

### 핵심 인사이트

> 이와 같은 루프를 구성할 수 없다면, 그것은 인터랙티브 모드에서도 LLM이 코드를 수정하도록 이끌 수 없다는 의미입니다.

`claude -p` 플래그는 Claude Code를 프롬프트로 비인터랙티브하게 실행하고, 완료되면 종료합니다. 파이프라인을 만들기 위해 호출을 연결하세요:

```bash
#!/bin/bash
# daily-dev.sh — Sequential pipeline for a feature branch

set -e

# Step 1: Implement the feature
claude -p "Read the spec in docs/auth-spec.md. Implement OAuth2 login in src/auth/. Write tests first (TDD). Do NOT create any new documentation files."

# Step 2: De-sloppify (cleanup pass)
claude -p "Review all files changed by the previous commit. Remove any unnecessary type tests, overly defensive checks, or testing of language features (e.g., testing that TypeScript generics work). Keep real business logic tests. Run the test suite after cleanup."

# Step 3: Verify
claude -p "Run the full build, lint, type check, and test suite. Fix any failures. Do not add new features."

# Step 4: Commit
claude -p "Create a conventional commit for all staged changes. Use 'feat: add OAuth2 login flow' as the message."
```

### 핵심 설계 원칙

1. **각 단계는 격리됨** — `claude -p` 호출당 새로운 컨텍스트 창은 단계 간 컨텍스트 오염이 없음을 의미합니다.
2. **순서가 중요함** — 단계는 순차적으로 실행됩니다. 각 단계는 이전 단계가 남긴 파일시스템 상태를 기반으로 합니다.
3. **부정 지시는 위험함** — "타입 시스템을 테스트하지 말라"고 하지 마세요. 대신 별도의 클린업 단계를 추가하세요 ([De-Sloppify 패턴](#5-the-de-sloppify-패턴) 참고).
4. **종료 코드가 전파됨** — `set -e`는 실패 시 파이프라인을 중지합니다.

### 변형

**모델 라우팅 포함:**
```bash
# Research with Opus (deep reasoning)
claude -p --model opus "Analyze the codebase architecture and write a plan for adding caching..."

# Implement with Sonnet (fast, capable)
claude -p "Implement the caching layer according to the plan in docs/caching-plan.md..."

# Review with Opus (thorough)
claude -p --model opus "Review all changes for security issues, race conditions, and edge cases..."
```

**환경 컨텍스트 포함:**
```bash
# Pass context via files, not prompt length
echo "Focus areas: auth module, API rate limiting" > .claude-context.md
claude -p "Read .claude-context.md for priorities. Work through them in order."
rm .claude-context.md
```

**`--allowedTools` 제한 포함:**
```bash
# Read-only analysis pass
claude -p --allowedTools "Read,Grep,Glob" "Audit this codebase for security vulnerabilities..."

# Write-only implementation pass
claude -p --allowedTools "Read,Write,Edit,Bash" "Implement the fixes from security-audit.md..."
```

---

## 2. NanoClaw REPL

**ECC의 내장 지속 루프.** 전체 대화 기록으로 `claude -p`를 동기적으로 호출하는 세션 인식 REPL입니다.

```bash
# Start the default session
node scripts/claw.js

# Named session with skill context
CLAW_SESSION=my-project CLAW_SKILLS=tdd-workflow,security-review node scripts/claw.js
```

### 작동 방식

1. `~/.claude/claw/{session}.md`에서 대화 기록 로드
2. 각 사용자 메시지는 전체 기록을 컨텍스트로 하여 `claude -p`에 전송됨
3. 응답이 세션 파일에 추가됨 (데이터베이스로서의 Markdown)
4. 세션이 재시작 후에도 지속됨

### NanoClaw vs 순차 파이프라인 비교

| 사용 사례 | NanoClaw | 순차 파이프라인 |
|----------|----------|-------------------|
| 인터랙티브 탐색 | 예 | 아니오 |
| 스크립트화된 자동화 | 아니오 | 예 |
| 세션 지속성 | 내장 | 수동 |
| 컨텍스트 누적 | 턴마다 증가 | 각 단계마다 새로 시작 |
| CI/CD 통합 | 불량 | 우수 |

자세한 내용은 `/claw` 명령 문서를 참고하세요.

---

## 3. Infinite Agentic Loop

**두 프롬프트 시스템**으로 사양 기반 생성을 위한 병렬 서브 에이전트를 오케스트레이션합니다. disler (@disler)가 개발했습니다.

### 아키텍처: 두 프롬프트 시스템

```
PROMPT 1 (Orchestrator)              PROMPT 2 (Sub-Agents)
┌─────────────────────┐             ┌──────────────────────┐
│ Parse spec file      │             │ Receive full context  │
│ Scan output dir      │  deploys   │ Read assigned number  │
│ Plan iteration       │────────────│ Follow spec exactly   │
│ Assign creative dirs │  N agents  │ Generate unique output │
│ Manage waves         │             │ Save to output dir    │
└─────────────────────┘             └──────────────────────┘
```

### 패턴

1. **사양 분석** — 오케스트레이터가 생성할 내용을 정의하는 사양 파일(Markdown)을 읽음
2. **디렉토리 정찰** — 기존 출력을 스캔하여 가장 높은 반복 번호 찾기
3. **병렬 배포** — N개의 서브 에이전트 실행, 각각 다음을 포함:
   - 전체 사양
   - 고유한 창의적 방향
   - 특정 반복 번호 (충돌 없음)
   - 기존 반복의 스냅샷 (고유성을 위해)
4. **웨이브 관리** — 무한 모드의 경우 컨텍스트 소진될 때까지 3-5개 에이전트의 웨이브 배포

### Claude Code 명령을 통한 구현

`.claude/commands/infinite.md` 생성:

```markdown
Parse the following arguments from $ARGUMENTS:
1. spec_file — path to the specification markdown
2. output_dir — where iterations are saved
3. count — integer 1-N or "infinite"

PHASE 1: Read and deeply understand the specification.
PHASE 2: List output_dir, find highest iteration number. Start at N+1.
PHASE 3: Plan creative directions — each agent gets a DIFFERENT theme/approach.
PHASE 4: Deploy sub-agents in parallel (Task tool). Each receives:
  - Full spec text
  - Current directory snapshot
  - Their assigned iteration number
  - Their unique creative direction
PHASE 5 (infinite mode): Loop in waves of 3-5 until context is low.
```

**호출:**
```bash
/project:infinite specs/component-spec.md src/ 5
/project:infinite specs/component-spec.md src/ infinite
```

### 배치 전략

| 수량 | 전략 |
|-------|----------|
| 1-5 | 모든 에이전트 동시 실행 |
| 6-20 | 5개씩 배치 실행 |
| infinite | 3-5개 웨이브, 점진적 정교화 |

### 핵심 인사이트: 할당을 통한 고유성

에이전트가 스스로 차별화하도록 의존하지 마세요. 오케스트레이터가 각 에이전트에게 특정 창의적 방향과 반복 번호를 **할당**합니다. 이는 병렬 에이전트 간의 중복 개념을 방지합니다.

---

## 4. Continuous Claude PR Loop

**프로덕션급 셸 스크립트**로 Claude Code를 지속적인 루프에서 실행하고, PR을 생성하고, CI를 기다리고, 자동으로 병합합니다. AnandChowdhary (@AnandChowdhary)가 만들었습니다.

### 핵심 루프

```
┌─────────────────────────────────────────────────────┐
│  CONTINUOUS CLAUDE ITERATION                        │
│                                                     │
│  1. Create branch (continuous-claude/iteration-N)   │
│  2. Run claude -p with enhanced prompt              │
│  3. (Optional) Reviewer pass — separate claude -p   │
│  4. Commit changes (claude generates message)       │
│  5. Push + create PR (gh pr create)                 │
│  6. Wait for CI checks (poll gh pr checks)          │
│  7. CI failure? → Auto-fix pass (claude -p)         │
│  8. Merge PR (squash/merge/rebase)                  │
│  9. Return to main → repeat                         │
│                                                     │
│  Limit by: --max-runs N | --max-cost $X             │
│            --max-duration 2h | completion signal     │
└─────────────────────────────────────────────────────┘
```

### 설치

```bash
curl -fsSL https://raw.githubusercontent.com/AnandChowdhary/continuous-claude/HEAD/install.sh | bash
```

### 사용법

```bash
# Basic: 10 iterations
continuous-claude --prompt "Add unit tests for all untested functions" --max-runs 10

# Cost-limited
continuous-claude --prompt "Fix all linter errors" --max-cost 5.00

# Time-boxed
continuous-claude --prompt "Improve test coverage" --max-duration 8h

# With code review pass
continuous-claude \
  --prompt "Add authentication feature" \
  --max-runs 10 \
  --review-prompt "Run npm test && npm run lint, fix any failures"

# Parallel via worktrees
continuous-claude --prompt "Add tests" --max-runs 5 --worktree tests-worker &
continuous-claude --prompt "Refactor code" --max-runs 5 --worktree refactor-worker &
wait
```

### 반복 간 컨텍스트: SHARED_TASK_NOTES.md

핵심 혁신: `SHARED_TASK_NOTES.md` 파일이 반복 간에 지속됩니다:

```markdown
## Progress
- [x] Added tests for auth module (iteration 1)
- [x] Fixed edge case in token refresh (iteration 2)
- [ ] Still need: rate limiting tests, error boundary tests

## Next Steps
- Focus on rate limiting module next
- The mock setup in tests/helpers.ts can be reused
```

Claude는 반복 시작 시 이 파일을 읽고 반복 종료 시 업데이트합니다. 이는 독립적인 `claude -p` 호출 간의 컨텍스트 간격을 연결합니다.

### CI 실패 복구

PR 검사 실패 시 Continuous Claude는 자동으로:
1. `gh run list`를 통해 실패한 실행 ID 가져오기
2. CI 수정 컨텍스트로 새 `claude -p` 실행
3. Claude가 `gh run view`를 통해 로그 검사, 코드 수정, 커밋, 푸시
4. 검사를 다시 기다림 (`--ci-retry-max` 시도 횟수까지)

### 완료 신호

Claude는 매직 문구를 출력하여 "완료"를 신호할 수 있습니다:

```bash
continuous-claude \
  --prompt "Fix all bugs in the issue tracker" \
  --completion-signal "CONTINUOUS_CLAUDE_PROJECT_COMPLETE" \
  --completion-threshold 3  # Stops after 3 consecutive signals
```

3번 연속 완료를 신호하는 반복이 루프를 중지하여 완료된 작업에 대한 낭비 실행을 방지합니다.

### 주요 설정

| 플래그 | 목적 |
|------|---------|
| `--max-runs N` | N번 성공적인 반복 후 중지 |
| `--max-cost $X` | $X 소비 후 중지 |
| `--max-duration 2h` | 시간 경과 후 중지 |
| `--merge-strategy squash` | squash, merge, 또는 rebase |
| `--worktree <name>` | git 워크트리를 통한 병렬 실행 |
| `--disable-commits` | 드라이런 모드 (git 작업 없음) |
| `--review-prompt "..."` | 반복당 리뷰어 패스 추가 |
| `--ci-retry-max N` | CI 실패 자동 수정 (기본: 1) |

---

## 5. The De-Sloppify 패턴

**어떤 루프에도 적용 가능한 추가 기능 패턴.** 각 구현 단계 후에 전용 클린업/리팩토링 단계를 추가하세요.

### 문제점

LLM에게 TDD로 구현하도록 요청하면 "테스트 작성"을 너무 문자 그대로 받아들입니다:
- TypeScript의 타입 시스템이 작동하는지 확인하는 테스트 (`typeof x === 'string'` 테스트)
- 타입 시스템이 이미 보장하는 것에 대한 과도하게 방어적인 런타임 검사
- 비즈니스 로직이 아닌 프레임워크 동작 테스트
- 실제 코드를 가리는 과도한 에러 처리

### 부정 지시가 안 되는 이유

구현 프롬프트에 "타입 시스템을 테스트하지 말라" 또는 "불필요한 검사 추가 금지"를 추가하면 다운스트림 효과가 있습니다:
- 모델이 모든 테스트에 대해 주저하게 됨
- 합법적인 엣지 케이스 테스트를 건너뜀
- 품질이 예측 불가능하게 저하됨

### 해결책: 별도 패스

구현자를 제약하는 대신 철저하게 진행하게 하세요. 그런 다음 집중된 클린업 에이전트를 추가하세요:

```bash
# Step 1: Implement (let it be thorough)
claude -p "Implement the feature with full TDD. Be thorough with tests."

# Step 2: De-sloppify (separate context, focused cleanup)
claude -p "Review all changes in the working tree. Remove:
- Tests that verify language/framework behavior rather than business logic
- Redundant type checks that the type system already enforces
- Over-defensive error handling for impossible states
- Console.log statements
- Commented-out code

Keep all business logic tests. Run the test suite after cleanup to ensure nothing breaks."
```

### 루프 컨텍스트에서

```bash
for feature in "${features[@]}"; do
  # Implement
  claude -p "Implement $feature with TDD."

  # De-sloppify
  claude -p "Cleanup pass: review changes, remove test/code slop, run tests."

  # Verify
  claude -p "Run build + lint + tests. Fix any failures."

  # Commit
  claude -p "Commit with message: feat: add $feature"
done
```

### 핵심 인사이트

> 다운스트림 품질 효과가 있는 부정 지시를 추가하는 대신, 별도의 de-sloppify 패스를 추가하세요. 두 개의 집중된 에이전트가 하나의 제약된 에이전트보다 뛰어납니다.

---

## 6. Ralphinho / RFC 기반 DAG 오케스트레이션

**가장 정교한 패턴.** RFC 기반 멀티 에이전트 파이프라인으로 사양을 의존성 DAG로 분해하고, 각 유닛을 계층화된 품질 파이프라인을 통해 실행하고, 에이전트 기반 병합 큐를 통해 처리합니다. enitrat (@enitrat)이 만들었습니다.

### 아키텍처 개요

```
RFC/PRD Document
       │
       ▼
  DECOMPOSITION (AI)
  Break RFC into work units with dependency DAG
       │
       ▼
┌──────────────────────────────────────────────────────┐
│  RALPH LOOP (up to 3 passes)                         │
│                                                      │
│  For each DAG layer (sequential, by dependency):     │
│                                                      │
│  ┌── Quality Pipelines (parallel per unit) ───────┐  │
│  │  Each unit in its own worktree:                │  │
│  │  Research → Plan → Implement → Test → Review   │  │
│  │  (depth varies by complexity tier)             │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
│  ┌── Merge Queue ─────────────────────────────────┐  │
│  │  Rebase onto main → Run tests → Land or evict │  │
│  │  Evicted units re-enter with conflict context  │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### RFC 분해

AI가 RFC를 읽고 작업 유닛을 생성합니다:

```typescript
interface WorkUnit {
  id: string;              // kebab-case identifier
  name: string;            // Human-readable name
  rfcSections: string[];   // Which RFC sections this addresses
  description: string;     // Detailed description
  deps: string[];          // Dependencies (other unit IDs)
  acceptance: string[];    // Concrete acceptance criteria
  tier: "trivial" | "small" | "medium" | "large";
}
```

**분해 규칙:**
- 더 적고 응집력 있는 유닛 선호 (병합 위험 최소화)
- 유닛 간 파일 중복 최소화 (충돌 방지)
- 테스트는 구현과 함께 유지 (절대 "X 구현" + "X 테스트" 분리 금지)
- 실제 코드 의존성이 존재하는 경우에만 의존성 설정

의존성 DAG가 실행 순서를 결정합니다:
```
Layer 0: [unit-a, unit-b]     ← no deps, run in parallel
Layer 1: [unit-c]             ← depends on unit-a
Layer 2: [unit-d, unit-e]     ← depend on unit-c
```

### 복잡도 티어

다른 티어는 다른 파이프라인 깊이를 갖습니다:

| 티어 | 파이프라인 단계 |
|------|----------------|
| **trivial** | 구현 → 테스트 |
| **small** | 구현 → 테스트 → code-review |
| **medium** | 리서치 → 계획 → 구현 → 테스트 → PRD-review + code-review → review-fix |
| **large** | 리서치 → 계획 → 구현 → 테스트 → PRD-review + code-review → review-fix → final-review |

이는 단순한 변경에 비용이 많이 드는 작업을 하지 않으면서 아키텍처 변경에는 철저한 검토를 보장합니다.

### 별도 컨텍스트 창 (저자 편향 제거)

각 단계는 자체 컨텍스트 창으로 자체 에이전트 프로세스에서 실행됩니다:

| 단계 | 모델 | 목적 |
|-------|-------|---------|
| 리서치 | Sonnet | 코드베이스 + RFC 읽기, 컨텍스트 문서 생성 |
| 계획 | Opus | 구현 단계 설계 |
| 구현 | Codex | 계획에 따라 코드 작성 |
| 테스트 | Sonnet | 빌드 + 테스트 스위트 실행 |
| PRD 리뷰 | Sonnet | 사양 준수 확인 |
| 코드 리뷰 | Opus | 품질 + 보안 확인 |
| 리뷰 수정 | Codex | 리뷰 이슈 처리 |
| 최종 리뷰 | Opus | 품질 게이트 (대형 티어만) |

**중요 설계:** 리뷰어는 자신이 리뷰하는 코드를 작성하지 않았습니다. 이는 자기 리뷰에서 가장 흔한 이슈 누락 원인인 저자 편향을 제거합니다.

### 제거 기능이 있는 병합 큐

품질 파이프라인 완료 후 유닛이 병합 큐에 진입합니다:

```
Unit branch
    │
    ├─ Rebase onto main
    │   └─ Conflict? → EVICT (capture conflict context)
    │
    ├─ Run build + tests
    │   └─ Fail? → EVICT (capture test output)
    │
    └─ Pass → Fast-forward main, push, delete branch
```

**파일 중복 인텔리전스:**
- 중복되지 않는 유닛은 투기적으로 병렬 병합
- 중복 유닛은 매번 리베이스하면서 하나씩 병합

**제거 복구:**
제거 시 전체 컨텍스트(충돌 파일, diff, 테스트 출력)가 캡처되어 다음 Ralph 패스에서 구현자에게 다시 제공됩니다:

```markdown
## MERGE CONFLICT — RESOLVE BEFORE NEXT LANDING

Your previous implementation conflicted with another unit that landed first.
Restructure your changes to avoid the conflicting files/lines below.

{full eviction context with diffs}
```

### 단계 간 데이터 흐름

```
research.contextFilePath ──────────────────→ plan
plan.implementationSteps ──────────────────→ implement
implement.{filesCreated, whatWasDone} ─────→ test, reviews
test.failingSummary ───────────────────────→ reviews, implement (next pass)
reviews.{feedback, issues} ────────────────→ review-fix → implement (next pass)
final-review.reasoning ────────────────────→ implement (next pass)
evictionContext ───────────────────────────→ implement (after merge conflict)
```

### 워크트리 격리

모든 유닛은 격리된 워크트리에서 실행됩니다 (git이 아닌 jj/Jujutsu 사용):
```
/tmp/workflow-wt-{unit-id}/
```

같은 유닛의 파이프라인 단계는 **공유** 워크트리에서 실행되어 리서치 → 계획 → 구현 → 테스트 → 리뷰 간에 상태(컨텍스트 파일, 계획 파일, 코드 변경)를 보존합니다.

### 핵심 설계 원칙

1. **결정적 실행** — 사전 분해가 병렬성과 순서를 고정
2. **레버리지 포인트에서 인간 리뷰** — 작업 계획이 가장 높은 레버리지 개입 포인트
3. **관심사 분리** — 별도 컨텍스트 창으로 별도 에이전트의 각 단계
4. **컨텍스트를 활용한 충돌 복구** — 전체 제거 컨텍스트로 맹목적인 재시도가 아닌 지능적인 재실행 가능
5. **티어 기반 깊이** — 사소한 변경은 리서치/리뷰 건너뜀; 대형 변경은 최대 검토
6. **재개 가능한 워크플로우** — 전체 상태가 SQLite에 지속됨; 어느 지점에서든 재개 가능

### Ralphinho vs 더 단순한 패턴 사용 시기

| 신호 | Ralphinho 사용 | 더 단순한 패턴 사용 |
|--------|--------------|-------------------|
| 상호 의존적인 여러 작업 유닛 | 예 | 아니오 |
| 병렬 구현 필요 | 예 | 아니오 |
| 병합 충돌 가능성 높음 | 예 | 아니오 (순차적이면 괜찮음) |
| 단일 파일 변경 | 아니오 | 예 (순차 파이프라인) |
| 여러 날에 걸친 프로젝트 | 예 | 아마도 (continuous-claude) |
| 사양/RFC 이미 작성됨 | 예 | 아마도 |
| 한 가지에 대한 빠른 반복 | 아니오 | 예 (NanoClaw 또는 파이프라인) |

---

## 올바른 패턴 선택

### 의사결정 매트릭스

```
Is the task a single focused change?
├─ Yes → Sequential Pipeline or NanoClaw
└─ No → Is there a written spec/RFC?
         ├─ Yes → Do you need parallel implementation?
         │        ├─ Yes → Ralphinho (DAG orchestration)
         │        └─ No → Continuous Claude (iterative PR loop)
         └─ No → Do you need many variations of the same thing?
                  ├─ Yes → Infinite Agentic Loop (spec-driven generation)
                  └─ No → Sequential Pipeline with de-sloppify
```

### 패턴 조합

이 패턴들은 잘 조합됩니다:

1. **순차 파이프라인 + De-Sloppify** — 가장 일반적인 조합. 모든 구현 단계는 클린업 패스를 받습니다.

2. **Continuous Claude + De-Sloppify** — 각 반복에 de-sloppify 지시문이 있는 `--review-prompt` 추가.

3. **모든 루프 + 검증** — 커밋 전 게이트로 ECC의 `/verify` 명령 또는 `verification-loop` 스킬 사용.

4. **더 단순한 루프에서 Ralphinho의 계층화된 접근 방식** — 순차 파이프라인에서도 단순 작업은 Haiku로, 복잡한 작업은 Opus로 라우팅할 수 있습니다:
   ```bash
   # Simple formatting fix
   claude -p --model haiku "Fix the import ordering in src/utils.ts"

   # Complex architectural change
   claude -p --model opus "Refactor the auth module to use the strategy pattern"
   ```

---

## 안티패턴

### 흔한 실수

1. **종료 조건 없는 무한 루프** — 항상 max-runs, max-cost, max-duration, 또는 완료 신호를 가지세요.

2. **반복 간 컨텍스트 브릿지 없음** — 각 `claude -p` 호출은 새로 시작합니다. `SHARED_TASK_NOTES.md` 또는 파일시스템 상태를 사용하여 컨텍스트를 연결하세요.

3. **같은 실패 재시도** — 반복이 실패하면 단순히 재시도하지 마세요. 에러 컨텍스트를 캡처하고 다음 시도에 제공하세요.

4. **클린업 패스 대신 부정 지시** — "X를 하지 마라"고 하지 마세요. X를 제거하는 별도 패스를 추가하세요.

5. **하나의 컨텍스트 창에 모든 에이전트** — 복잡한 워크플로우의 경우 다른 에이전트 프로세스로 관심사를 분리하세요. 리뷰어는 저자가 되어서는 안 됩니다.

6. **병렬 작업에서 파일 중복 무시** — 두 병렬 에이전트가 같은 파일을 편집할 수 있다면 병합 전략이 필요합니다 (순차 병합, 리베이스, 또는 충돌 해결).

---

## 참고

| 프로젝트 | 저자 | 링크 |
|---------|--------|------|
| Ralphinho | enitrat | credit: @enitrat |
| Infinite Agentic Loop | disler | credit: @disler |
| Continuous Claude | AnandChowdhary | credit: @AnandChowdhary |
| NanoClaw | ECC | 이 레포의 `/claw` 명령 |
| Verification Loop | ECC | 이 레포의 `skills/verification-loop/` |

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
# daily-dev.sh — 기능 브랜치를 위한 순차 파이프라인

set -e

# 1단계: 기능 구현
claude -p "docs/auth-spec.md의 사양을 읽으세요. src/auth/에 OAuth2 로그인을 구현하세요. 먼저 테스트를 작성하세요 (TDD). 새 문서 파일을 만들지 마세요."

# 2단계: De-sloppify (클린업 패스)
claude -p "이전 커밋으로 변경된 모든 파일을 검토하세요. 불필요한 타입 테스트, 과도하게 방어적인 검사, 또는 언어 기능 테스트(예: TypeScript 제네릭이 작동하는지 테스트)를 제거하세요. 실제 비즈니스 로직 테스트는 유지하세요. 클린업 후 테스트 스위트를 실행하세요."

# 3단계: 검증
claude -p "전체 빌드, lint, 타입 체크, 테스트 스위트를 실행하세요. 실패한 것은 수정하세요. 새 기능을 추가하지 마세요."

# 4단계: 커밋
claude -p "스테이징된 모든 변경에 대한 conventional commit을 생성하세요. 'feat: add OAuth2 login flow'를 메시지로 사용하세요."
```

### 핵심 설계 원칙

1. **각 단계는 격리됨** — `claude -p` 호출당 새로운 컨텍스트 창은 단계 간 컨텍스트 오염이 없음을 의미합니다.
2. **순서가 중요함** — 단계는 순차적으로 실행됩니다. 각 단계는 이전 단계가 남긴 파일시스템 상태를 기반으로 합니다.
3. **부정 지시는 위험함** — "타입 시스템을 테스트하지 말라"고 하지 마세요. 대신 별도의 클린업 단계를 추가하세요 ([De-Sloppify 패턴](#5-the-de-sloppify-패턴) 참고).
4. **종료 코드가 전파됨** — `set -e`는 실패 시 파이프라인을 중지합니다.

### 변형

**모델 라우팅 포함:**
```bash
# Opus로 리서치 (심층 추론)
claude -p --model opus "코드베이스 아키텍처를 분석하고 캐싱 추가 계획을 작성하세요..."

# Sonnet으로 구현 (빠르고 유능함)
claude -p "docs/caching-plan.md의 계획에 따라 캐싱 레이어를 구현하세요..."

# Opus로 리뷰 (철저함)
claude -p --model opus "보안 이슈, 경쟁 조건, 엣지 케이스에 대해 모든 변경사항을 검토하세요..."
```

**환경 컨텍스트 포함:**
```bash
# 프롬프트 길이가 아닌 파일을 통해 컨텍스트 전달
echo "Focus areas: auth module, API rate limiting" > .claude-context.md
claude -p ".claude-context.md를 읽어 우선순위를 확인하세요. 순서대로 처리하세요."
rm .claude-context.md
```

**`--allowedTools` 제한 포함:**
```bash
# 읽기 전용 분석 패스
claude -p --allowedTools "Read,Grep,Glob" "이 코드베이스를 보안 취약점에 대해 감사하세요..."

# 쓰기 전용 구현 패스
claude -p --allowedTools "Read,Write,Edit,Bash" "security-audit.md의 수정사항을 구현하세요..."
```

---

## 2. NanoClaw REPL

**ECC의 내장 지속 루프.** 전체 대화 기록으로 `claude -p`를 동기적으로 호출하는 세션 인식 REPL입니다.

```bash
# 기본 세션 시작
node scripts/claw.js

# 스킬 컨텍스트가 있는 명명된 세션
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
PROMPT 1 (오케스트레이터)           PROMPT 2 (서브 에이전트)
┌─────────────────────┐             ┌──────────────────────┐
│ 사양 파일 파싱        │             │ 전체 컨텍스트 수신     │
│ 출력 디렉토리 스캔    │  배포       │ 할당된 번호 읽기      │
│ 반복 계획 수립        │────────────│ 사양 정확히 따름      │
│ 창의적 방향 할당      │  N 에이전트 │ 고유 출력 생성        │
│ 웨이브 관리           │             │ 출력 디렉토리에 저장  │
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
$ARGUMENTS에서 다음 인수를 파싱하세요:
1. spec_file — 사양 마크다운의 경로
2. output_dir — 반복이 저장되는 곳
3. count — 정수 1-N 또는 "infinite"

PHASE 1: 사양을 읽고 깊이 이해하세요.
PHASE 2: output_dir를 나열하고, 가장 높은 반복 번호 찾기. N+1에서 시작.
PHASE 3: 창의적 방향 계획 — 각 에이전트는 다른 테마/접근 방식을 받음.
PHASE 4: 서브 에이전트를 병렬로 배포 (Task 도구). 각각 다음을 받음:
  - 전체 사양 텍스트
  - 현재 디렉토리 스냅샷
  - 할당된 반복 번호
  - 고유한 창의적 방향
PHASE 5 (infinite 모드): 컨텍스트가 낮을 때까지 3-5개의 웨이브로 루프.
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
│  CONTINUOUS CLAUDE 반복                              │
│                                                     │
│  1. 브랜치 생성 (continuous-claude/iteration-N)     │
│  2. 향상된 프롬프트로 claude -p 실행                 │
│  3. (선택적) 리뷰어 패스 — 별도 claude -p           │
│  4. 변경사항 커밋 (claude가 메시지 생성)             │
│  5. 푸시 + PR 생성 (gh pr create)                   │
│  6. CI 검사 대기 (gh pr checks 폴링)                │
│  7. CI 실패? → 자동 수정 패스 (claude -p)           │
│  8. PR 병합 (squash/merge/rebase)                   │
│  9. main으로 돌아가서 반복                          │
│                                                     │
│  제한: --max-runs N | --max-cost $X                 │
│        --max-duration 2h | 완료 신호                │
└─────────────────────────────────────────────────────┘
```

### 설치

```bash
curl -fsSL https://raw.githubusercontent.com/AnandChowdhary/continuous-claude/HEAD/install.sh | bash
```

### 사용법

```bash
# 기본: 10회 반복
continuous-claude --prompt "테스트되지 않은 모든 함수에 단위 테스트 추가" --max-runs 10

# 비용 제한
continuous-claude --prompt "모든 린터 오류 수정" --max-cost 5.00

# 시간 제한
continuous-claude --prompt "테스트 커버리지 개선" --max-duration 8h

# 코드 리뷰 패스 포함
continuous-claude \
  --prompt "인증 기능 추가" \
  --max-runs 10 \
  --review-prompt "npm test && npm run lint 실행, 실패 수정"

# 워크트리를 통한 병렬 실행
continuous-claude --prompt "테스트 추가" --max-runs 5 --worktree tests-worker &
continuous-claude --prompt "코드 리팩토링" --max-runs 5 --worktree refactor-worker &
wait
```

### 반복 간 컨텍스트: SHARED_TASK_NOTES.md

핵심 혁신: `SHARED_TASK_NOTES.md` 파일이 반복 간에 지속됩니다:

```markdown
## 진행 상황
- [x] auth 모듈에 테스트 추가 (반복 1)
- [x] 토큰 갱신의 엣지 케이스 수정 (반복 2)
- [ ] 아직 필요한 것: 요청 제한 테스트, 에러 경계 테스트

## 다음 단계
- 다음에는 요청 제한 모듈에 집중
- tests/helpers.ts의 mock 설정 재사용 가능
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
  --prompt "이슈 트래커의 모든 버그 수정" \
  --completion-signal "CONTINUOUS_CLAUDE_PROJECT_COMPLETE" \
  --completion-threshold 3  # 3번 연속 신호 후 중지
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
# 1단계: 구현 (철저하게 진행하도록 허용)
claude -p "전체 TDD로 기능을 구현하세요. 테스트에 대해 철저하게 하세요."

# 2단계: De-sloppify (별도 컨텍스트, 집중된 클린업)
claude -p "작업 트리의 모든 변경사항을 검토하세요. 다음을 제거하세요:
- 비즈니스 로직이 아닌 언어/프레임워크 동작을 검증하는 테스트
- 타입 시스템이 이미 강제하는 중복 타입 검사
- 불가능한 상태에 대한 과도하게 방어적인 에러 처리
- Console.log 문
- 주석 처리된 코드

모든 비즈니스 로직 테스트는 유지하세요. 클린업 후 테스트 스위트를 실행하여 아무것도 깨지지 않는지 확인하세요."
```

### 루프 컨텍스트에서

```bash
for feature in "${features[@]}"; do
  # 구현
  claude -p "$feature를 TDD로 구현하세요."

  # De-sloppify
  claude -p "클린업 패스: 변경사항 검토, 테스트/코드 슬롭 제거, 테스트 실행."

  # 검증
  claude -p "빌드 + lint + 테스트 실행. 실패 수정."

  # 커밋
  claude -p "메시지: feat: add $feature로 커밋하세요"
done
```

### 핵심 인사이트

> 다운스트림 품질 효과가 있는 부정 지시를 추가하는 대신, 별도의 de-sloppify 패스를 추가하세요. 두 개의 집중된 에이전트가 하나의 제약된 에이전트보다 뛰어납니다.

---

## 6. Ralphinho / RFC 기반 DAG 오케스트레이션

**가장 정교한 패턴.** RFC 기반 멀티 에이전트 파이프라인으로 사양을 의존성 DAG로 분해하고, 각 유닛을 계층화된 품질 파이프라인을 통해 실행하고, 에이전트 기반 병합 큐를 통해 처리합니다. enitrat (@enitrat)이 만들었습니다.

### 아키텍처 개요

```
RFC/PRD 문서
       │
       ▼
  분해 (AI)
  RFC를 의존성 DAG가 있는 작업 유닛으로 분리
       │
       ▼
┌──────────────────────────────────────────────────────┐
│  RALPH 루프 (최대 3번 패스)                           │
│                                                      │
│  각 DAG 레이어에 대해 (의존성에 따라 순차적으로):     │
│                                                      │
│  ┌── 품질 파이프라인 (유닛당 병렬) ───────────────┐  │
│  │  각 유닛은 자체 워크트리에서:                  │  │
│  │  리서치 → 계획 → 구현 → 테스트 → 리뷰         │  │
│  │  (복잡도 티어에 따라 깊이 변동)                │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
│  ┌── 병합 큐 ──────────────────────────────────┐  │
│  │  main에 리베이스 → 테스트 실행 → 병합 또는 제거│  │
│  │  제거된 유닛은 충돌 컨텍스트와 함께 재진입     │  │
│  └────────────────────────────────────────────────┘  │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### RFC 분해

AI가 RFC를 읽고 작업 유닛을 생성합니다:

```typescript
interface WorkUnit {
  id: string;              // kebab-case 식별자
  name: string;            // 사람이 읽기 쉬운 이름
  rfcSections: string[];   // 어떤 RFC 섹션을 다루는지
  description: string;     // 상세 설명
  deps: string[];          // 의존성 (다른 유닛 ID)
  acceptance: string[];    // 구체적인 인수 기준
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
레이어 0: [unit-a, unit-b]     ← 의존성 없음, 병렬 실행
레이어 1: [unit-c]             ← unit-a에 의존
레이어 2: [unit-d, unit-e]     ← unit-c에 의존
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
유닛 브랜치
    │
    ├─ main에 리베이스
    │   └─ 충돌? → 제거 (충돌 컨텍스트 캡처)
    │
    ├─ 빌드 + 테스트 실행
    │   └─ 실패? → 제거 (테스트 출력 캡처)
    │
    └─ 통과 → main 빠른 전진, 푸시, 브랜치 삭제
```

**파일 중복 인텔리전스:**
- 중복되지 않는 유닛은 투기적으로 병렬 병합
- 중복 유닛은 매번 리베이스하면서 하나씩 병합

**제거 복구:**
제거 시 전체 컨텍스트(충돌 파일, diff, 테스트 출력)가 캡처되어 다음 Ralph 패스에서 구현자에게 다시 제공됩니다:

```markdown
## 병합 충돌 — 다음 병합 전 해결 필요

이전 구현이 먼저 병합된 다른 유닛과 충돌했습니다.
아래 충돌하는 파일/줄을 피하도록 변경사항을 재구성하세요.

{전체 제거 컨텍스트 및 diff}
```

### 단계 간 데이터 흐름

```
research.contextFilePath ──────────────────→ plan
plan.implementationSteps ──────────────────→ implement
implement.{filesCreated, whatWasDone} ─────→ test, reviews
test.failingSummary ───────────────────────→ reviews, implement (다음 패스)
reviews.{feedback, issues} ────────────────→ review-fix → implement (다음 패스)
final-review.reasoning ────────────────────→ implement (다음 패스)
evictionContext ───────────────────────────→ implement (병합 충돌 후)
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
작업이 단일 집중 변경인가?
├─ 예 → 순차 파이프라인 또는 NanoClaw
└─ 아니오 → 작성된 사양/RFC가 있는가?
             ├─ 예 → 병렬 구현이 필요한가?
             │        ├─ 예 → Ralphinho (DAG 오케스트레이션)
             │        └─ 아니오 → Continuous Claude (반복 PR 루프)
             └─ 아니오 → 같은 것의 많은 변형이 필요한가?
                          ├─ 예 → Infinite Agentic Loop (사양 기반 생성)
                          └─ 아니오 → de-sloppify를 포함한 순차 파이프라인
```

### 패턴 조합

이 패턴들은 잘 조합됩니다:

1. **순차 파이프라인 + De-Sloppify** — 가장 일반적인 조합. 모든 구현 단계는 클린업 패스를 받습니다.

2. **Continuous Claude + De-Sloppify** — 각 반복에 de-sloppify 지시문이 있는 `--review-prompt` 추가.

3. **모든 루프 + 검증** — 커밋 전 게이트로 ECC의 `/verify` 명령 또는 `verification-loop` 스킬 사용.

4. **더 단순한 루프에서 Ralphinho의 계층화된 접근 방식** — 순차 파이프라인에서도 단순 작업은 Haiku로, 복잡한 작업은 Opus로 라우팅할 수 있습니다:
   ```bash
   # 단순 포매팅 수정
   claude -p --model haiku "src/utils.ts의 임포트 순서 수정"

   # 복잡한 아키텍처 변경
   claude -p --model opus "auth 모듈을 strategy 패턴으로 리팩토링"
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

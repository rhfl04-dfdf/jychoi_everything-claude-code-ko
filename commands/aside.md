---
description: 현재 작업을 중단하거나 컨텍스트를 잃지 않고 빠른 사이드 질문에 답변합니다. 답변 후 자동으로 작업을 재개합니다.
---

# Aside 커맨드

작업 도중 질문을 하고 즉각적이고 집중된 답변을 받은 후 — 멈췄던 곳에서 바로 이어서 계속합니다. 현재 작업, 파일, 컨텍스트는 절대 수정되지 않습니다.

## 사용 시점

- Claude가 작업 중인데 모멘텀을 잃지 않고 뭔가 궁금한 것이 생겼을 때
- Claude가 현재 편집 중인 코드에 대한 빠른 설명이 필요할 때
- 작업을 방해하지 않고 결정에 대한 의견이나 확인이 필요할 때
- Claude가 진행하기 전에 에러, 개념, 패턴을 이해해야 할 때
- 새 세션을 시작하지 않고 현재 작업과 무관한 것을 묻고 싶을 때

## 사용법

```
/aside <your question>
/aside what does this function actually return?
/aside is this pattern thread-safe?
/aside why are we using X instead of Y here?
/aside what's the difference between foo() and bar()?
/aside should we be worried about the N+1 query we just added?
```

## 프로세스

### 1단계: 현재 작업 상태 고정

무엇이든 답변하기 전에 다음을 기억해 둡니다:
- 현재 활성 작업이 무엇인지? (어떤 파일, 기능, 또는 문제를 작업 중이었는지)
- `/aside`가 호출된 순간 어떤 단계가 진행 중이었는지?
- 다음에 무엇이 일어날 예정이었는지?

aside 중에는 파일을 건드리거나, 편집하거나, 생성하거나, 삭제하지 않습니다.

### 2단계: 질문에 직접 답변

완전하고 유용하면서도 가장 간결한 형태로 질문에 답변합니다.

- 추론이 아닌 답변을 먼저 제시합니다
- 간결하게 유지합니다 — 전체 설명이 필요한 경우 작업 후 더 깊이 들어가겠다고 제안합니다
- 현재 작업 중인 파일이나 코드에 관한 질문인 경우 정확하게 참조합니다 (관련이 있다면 파일 경로와 줄 번호)
- 답변을 위해 파일을 읽어야 한다면 읽습니다 — 단, 읽기만 하고 쓰지 않습니다

응답 형식:

```
ASIDE: [restate the question briefly]

[Your answer here]

— Back to task: [one-line description of what was being done]
```

### 3단계: 메인 작업 재개

답변을 전달한 후 즉시 멈췄던 정확한 지점부터 활성 작업을 이어갑니다. aside 답변이 차단 요소나 현재 접근 방식을 재고해야 할 이유를 드러낸 경우가 아니라면 재개 허락을 구하지 않습니다 (엣지 케이스 참조).

---

## 엣지 케이스

**질문이 없는 경우 (`/aside` 뒤에 아무것도 없을 때):**
다음과 같이 응답합니다:
```
ASIDE: no question provided

What would you like to know? (ask your question and I'll answer without losing the current task context)

— Back to task: [one-line description of what was being done]
```

**질문이 현재 작업의 잠재적 문제를 드러낼 때:**
재개하기 전에 명확하게 플래그를 표시합니다:
```
ASIDE: [answer]

⚠️ Note: This answer suggests [issue] with the current approach. Want to address this before continuing, or proceed as planned?
```
재개하기 전에 사용자의 결정을 기다립니다.

**질문이 실제로 작업 변경 요청인 경우 (사이드 질문이 아닐 때):**
질문이 무엇을 만들고 있는지 변경을 의미하는 경우 (예: `/aside 실은 Redis를 사용하면 어떨까요`), 다음과 같이 명확히 합니다:
```
ASIDE: That sounds like a direction change, not just a side question.
Do you want to:
  (a) Answer this as information only and keep the current plan
  (b) Pause the current task and change approach
```
사용자의 답변을 기다립니다 — 가정하지 않습니다.

**현재 열려 있는 파일이나 코드에 관한 질문인 경우:**
라이브 컨텍스트에서 답변합니다. 세션 초반에 파일을 읽었다면 직접 참조합니다. 그렇지 않다면 지금 읽어서 (읽기 전용으로) file:line 참조와 함께 답변합니다.

**`/aside`가 호출될 때 활성 작업이 없는 경우:**
응답 형태를 일관되게 유지하기 위해 표준 래퍼를 사용합니다:
```
ASIDE: [restate the question briefly]

[Your answer here]

— Back to task: no active task to resume
```

**긴 답변이 필요한 질문인 경우:**
핵심 답변을 간결하게 제공한 후 다음을 제안합니다:
```
That's the short version. Want a deeper explanation after we finish [current task]?
```

**연속적인 여러 `/aside` 질문:**
각각 순서대로 답변합니다. 마지막 답변 후 메인 작업을 재개합니다. 연속된 aside 체인 전반에 걸쳐 작업 상태를 잃지 않습니다.

**Aside 답변이 코드 변경이 필요함을 시사하는 경우:**
aside 중에는 변경을 하지 않고 필요한 변경 사항을 메모합니다:
```
ASIDE: [answer]

📝 Worth fixing: [what should be changed]. I'll flag this after the current task unless you want to address it now.
```

**질문이 모호하거나 너무 막연한 경우:**
단 하나의 명확화 질문을 합니다 — 답변에 필요한 정보를 얻기 위한 가장 짧은 질문. 여러 질문을 하지 않습니다.

---

## 출력 예시

```
User: /aside what does fetchWithRetry() actually do?

ASIDE: what does fetchWithRetry() do?

fetchWithRetry() (src/api/retry.ts:12) attempts the request up to 3 times with
exponential backoff (250ms → 500ms → 1s). It only retries on 5xx responses and
network errors — 4xx errors are treated as final and not retried.

— Back to task: refactoring the auth middleware in src/middleware/auth.ts
```

```
User: /aside is the approach we're taking thread-safe?

ASIDE: is the current approach thread-safe?

No — the shared cache object in src/cache/store.ts:34 is mutated without locking.
Under concurrent requests this is a race condition. It's low risk in a single-process
Node.js server but would be a real problem with worker threads or clustering.

⚠️ Note: This could affect the feature we're building. Want to address this now or continue and fix it in a follow-up?
```

---

## 참고 사항

- aside 중에는 절대 파일을 수정하지 않습니다 — 읽기 전용 접근만 허용
- aside는 대화 일시 중지이지 새 작업이 아닙니다 — 원래 작업은 항상 재개되어야 합니다
- 답변을 집중적으로 유지합니다: 목표는 강의를 제공하는 것이 아니라 사용자의 막힘을 빠르게 해소하는 것입니다
- aside가 더 큰 논의로 이어진다면, aside가 차단 요소를 드러내지 않는 한 현재 작업을 먼저 마칩니다
- Aside는 작업 결과와 명시적으로 관련되지 않는 한 세션 파일에 저장되지 않습니다

---
description: 현재 session 상태를 날짜가 포함된 ~/.claude/sessions/ 폴더의 파일로 저장하여, 향후 session에서 모든 context를 유지한 채 작업을 재개할 수 있도록 합니다.
---

# Save Session 명령

이 session에서 발생한 모든 내용(빌드된 것, 성공한 것, 실패한 것, 남은 작업 등)을 캡처하여 날짜가 기입된 파일에 기록함으로써, 다음 session이 중단된 지점에서 정확히 다시 시작할 수 있게 합니다.

## 사용 시점

- Claude Code를 종료하기 전 작업 session을 마칠 때
- context 제한에 도달하기 전 (이 명령을 먼저 실행한 후 새로운 session을 시작하세요)
- 기억하고 싶은 복잡한 문제를 해결한 후
- 향후 session으로 context를 전달해야 할 때 언제든지

## 프로세스

### 단계 1: context 수집

파일을 작성하기 전에 다음 사항을 수집합니다:

- 이 session 동안 수정된 모든 파일을 읽습니다 (git diff를 사용하거나 대화 내용을 복기하세요)
- 논의된 사항, 시도한 내용, 결정된 사항을 검토합니다
- 발생한 오류와 해결 방법(또는 미해결 상태)을 기록합니다
- 관련이 있는 경우 현재 테스트/빌드 상태를 확인합니다

### 단계 2: sessions 폴더가 없으면 생성

사용자의 Claude 홈 디렉토리에 표준 sessions 폴더를 생성합니다:

```bash
mkdir -p ~/.claude/sessions
```

### 단계 3: session 파일 작성

오늘의 실제 날짜와 `session-manager.js`의 `SESSION_FILENAME_REGEX` 규칙을 만족하는 short-id를 사용하여 `~/.claude/sessions/YYYY-MM-DD-<short-id>-session.tmp`를 생성합니다:

- 허용되는 문자: 소문자 `a-z`, 숫자 `0-9`, 하이픈 `-`
- 최소 길이: 8자
- 대문자, 언더스코어(`_`), 공백 사용 불가

유효한 예시: `abc123de`, `a1b2c3d4`, `frontend-worktree-1`
유효하지 않은 예시: `ABC123de` (대문자), `short` (8자 미만), `test_id1` (언더스코어)

전체 유효 파일명 예시: `2024-01-15-abc123de-session.tmp`

레거시 파일명 형식인 `YYYY-MM-DD-session.tmp`도 여전히 유효하지만, 동일한 날짜의 충돌을 피하기 위해 새로운 session 파일은 short-id 형식을 사용하는 것이 좋습니다.

### 단계 4: 아래의 모든 섹션으로 파일 내용 채우기

모든 섹션을 정직하게 작성하세요. 섹션을 건너뛰지 마세요. 실제로 내용이 없는 섹션이라면 "Nothing yet" 또는 "N/A"라고 작성하세요. 불완전한 파일은 정직하게 비워둔 섹션보다 더 좋지 않습니다.

### 단계 5: 사용자에게 파일 보여주기

작성 후 전체 내용을 표시하고 다음과 같이 질문하세요:

```
Session saved to [actual resolved path to the session file]

Does this look accurate? Anything to correct or add before we close?
```

확인을 기다립니다. 요청이 있으면 수정합니다.

---

## Session 파일 형식

```markdown
# Session: YYYY-MM-DD

**Started:** [approximate time if known]
**Last Updated:** [current time]
**Project:** [project name or path]
**Topic:** [one-line summary of what this session was about]

---

## What We Are Building

[1-3 paragraphs describing the feature, bug fix, or task. Include enough
context that someone with zero memory of this session can understand the goal.
Include: what it does, why it's needed, how it fits into the larger system.]

---

## What WORKED (with evidence)

[List only things that are confirmed working. For each item include WHY you
know it works — test passed, ran in browser, Postman returned 200, etc.
Without evidence, move it to "Not Tried Yet" instead.]

- **[thing that works]** — confirmed by: [specific evidence]
- **[thing that works]** — confirmed by: [specific evidence]

If nothing is confirmed working yet: "Nothing confirmed working yet — all approaches still in progress or untested."

---

## What Did NOT Work (and why)

[This is the most important section. List every approach tried that failed.
For each failure write the EXACT reason so the next session doesn't retry it.
Be specific: "threw X error because Y" is useful. "didn't work" is not.]

- **[approach tried]** — failed because: [exact reason / error message]
- **[approach tried]** — failed because: [exact reason / error message]

If nothing failed: "No failed approaches yet."

---

## What Has NOT Been Tried Yet

[Approaches that seem promising but haven't been attempted. Ideas from the
conversation. Alternative solutions worth exploring. Be specific enough that
the next session knows exactly what to try.]

- [approach / idea]
- [approach / idea]

If nothing is queued: "No specific untried approaches identified."

---

## Current State of Files

[Every file touched this session. Be precise about what state each file is in.]

| File              | Status         | Notes                      |
| ----------------- | -------------- | -------------------------- |
| `path/to/file.ts` | ✅ Complete    | [what it does]             |
| `path/to/file.ts` | 🔄 In Progress | [what's done, what's left] |
| `path/to/file.ts` | ❌ Broken      | [what's wrong]             |
| `path/to/file.ts` | 🗒️ Not Started | [planned but not touched]  |

If no files were touched: "No files modified this session."

---

## Decisions Made

[Architecture choices, tradeoffs accepted, approaches chosen and why.
These prevent the next session from relitigating settled decisions.]

- **[decision]** — reason: [why this was chosen over alternatives]

If no significant decisions: "No major decisions made this session."

---

## Blockers & Open Questions

[Anything unresolved that the next session needs to address or investigate.
Questions that came up but weren't answered. External dependencies waiting on.]

- [blocker / open question]

If none: "No active blockers."

---

## Exact Next Step

[If known: The single most important thing to do when resuming. Be precise
enough that resuming requires zero thinking about where to start.]

[If not known: "Next step not determined — review 'What Has NOT Been Tried Yet'
and 'Blockers' sections to decide on direction before starting."]

---

## Environment & Setup Notes

[Only fill this if relevant — commands needed to run the project, env vars
required, services that need to be running, etc. Skip if standard setup.]

[If none: omit this section entirely.]
```

---

## 예시 출력

```markdown
# Session: 2024-01-15

**Started:** ~2pm
**Last Updated:** 5:30pm
**Project:** my-app
**Topic:** Building JWT authentication with httpOnly cookies

---

## What We Are Building

User authentication system for the Next.js app. Users register with email/password,
receive a JWT stored in an httpOnly cookie (not localStorage), and protected routes
check for a valid token via middleware. The goal is session persistence across browser
refreshes without exposing the token to JavaScript.

---

## What WORKED (with evidence)

- **`/api/auth/register` endpoint** — confirmed by: Postman POST returns 200 with user
  object, row visible in Supabase dashboard, bcrypt hash stored correctly
- **JWT generation in `lib/auth.ts`** — confirmed by: unit test passes
  (`npm test -- auth.test.ts`), decoded token at jwt.io shows correct payload
- **Password hashing** — confirmed by: `bcrypt.compare()` returns true in test

---

## What Did NOT Work (and why)

- **Next-Auth library** — failed because: conflicts with our custom Prisma adapter,
  threw "Cannot use adapter with credentials provider in this configuration" on every
  request. Not worth debugging — too opinionated for our setup.
- **Storing JWT in localStorage** — failed because: SSR renders happen before
  localStorage is available, caused React hydration mismatch error on every page load.
  This approach is fundamentally incompatible with Next.js SSR.

---

## What Has NOT Been Tried Yet

- Store JWT as httpOnly cookie in the login route response (most likely solution)
- Use `cookies()` from `next/headers` to read token in server components
- Write middleware.ts to protect routes by checking cookie existence

---

## Current State of Files

| File                             | Status         | Notes                                           |
| -------------------------------- | -------------- | ----------------------------------------------- |
| `app/api/auth/register/route.ts` | ✅ Complete    | Works, tested                                   |
| `app/api/auth/login/route.ts`    | 🔄 In Progress | Token generates but not setting cookie yet      |
| `lib/auth.ts`                    | ✅ Complete    | JWT helpers, all tested                         |
| `middleware.ts`                  | 🗒️ Not Started | Route protection, needs cookie read logic first |
| `app/login/page.tsx`             | 🗒️ Not Started | UI not started                                  |

---

## Decisions Made

- **httpOnly cookie over localStorage** — reason: prevents XSS token theft, works with SSR
- **Custom auth over Next-Auth** — reason: Next-Auth conflicts with our Prisma setup, not worth the fight

---

## Blockers & Open Questions

- Does `cookies().set()` work inside a Route Handler or only in Server Actions? Need to verify.

---

## Exact Next Step

In `app/api/auth/login/route.ts`, after generating the JWT, set it as an httpOnly
cookie using `cookies().set('token', jwt, { httpOnly: true, secure: true, sameSite: 'strict' })`.
Then test with Postman — the response should include a `Set-Cookie` header.
```

---

## 참고 사항

- 각 session은 고유한 파일을 가집니다 — 이전 session 파일에 내용을 추가하지 마세요.
- "What Did NOT Work" 섹션이 가장 중요합니다 — 이 내용이 없으면 향후 session에서 실패한 접근 방식을 맹목적으로 다시 시도하게 될 수 있습니다.
- 사용자가 session 중간에 저장을 요청하는 경우(종료 시점뿐만 아니라), 현재까지 파악된 내용을 저장하고 진행 중인 항목을 명확하게 표시하세요.
- 이 파일은 다음 session 시작 시 `/resume-session`을 통해 Claude가 읽도록 고안되었습니다.
- 표준 글로벌 session 저장소인 `~/.claude/sessions/`를 사용하세요.
- 새로운 session 파일에는 short-id 파일명 형식(`YYYY-MM-DD-<short-id>-session.tmp`)을 사용하는 것이 좋습니다.

---
description: ~/.claude/sessions/에서 가장 최근 세션 파일을 로드하고 마지막 세션이 종료된 지점부터 전체 context와 함께 작업을 재개합니다.
---

# Resume Session 명령

작업을 시작하기 전에 마지막으로 저장된 세션 상태를 로드하고 완전히 파악합니다.
이 명령은 `/save-session`의 대응 쌍입니다.

## 사용 시기

- 이전 날짜의 작업을 계속하기 위해 새 세션을 시작할 때
- context 제한으로 인해 새 세션을 시작한 후
- 다른 소스에서 세션 파일을 전달받았을 때 (파일 경로만 제공)
- 세션 파일이 있고 진행하기 전에 Claude가 이를 완전히 흡수하기를 원할 때마다

## Usage

```
/resume-session                                                      # loads most recent file in ~/.claude/sessions/
/resume-session 2024-01-15                                           # loads most recent session for that date
/resume-session ~/.claude/sessions/2024-01-15-session.tmp           # loads a specific legacy-format file
/resume-session ~/.claude/sessions/2024-01-15-abc123de-session.tmp  # loads a current short-id session file
```

## 프로세스

### 1단계: 세션 파일 찾기

인수가 제공되지 않은 경우:

1. `~/.claude/sessions/` 확인
2. 가장 최근에 수정된 `*-session.tmp` 파일 선택
3. 폴더가 존재하지 않거나 일치하는 파일이 없으면 사용자에게 다음과 같이 알립니다:
   ```
   No session files found in ~/.claude/sessions/
   Run /save-session at the end of a session to create one.
   ```
   그 다음 중단합니다.

인수가 제공된 경우:

- 날짜 형식(`YYYY-MM-DD`)인 경우, `~/.claude/sessions/`에서 `YYYY-MM-DD-session.tmp`(레거시 형식) 또는 `YYYY-MM-DD-<shortid>-session.tmp`(현재 형식)와 일치하는 파일을 검색하고 해당 날짜의 가장 최근에 수정된 변체를 로드합니다.
- 파일 경로인 경우, 해당 파일을 직접 읽습니다.
- 찾을 수 없는 경우 명확하게 보고하고 중단합니다.

### 2단계: 전체 세션 파일 읽기

파일 전체를 읽습니다. 아직 요약하지 마세요.

### 3단계: 이해 확인

다음과 같은 정확한 형식의 구조화된 브리핑으로 응답합니다:

```
SESSION LOADED: [actual resolved path to the file]
════════════════════════════════════════════════

PROJECT: [project name / topic from file]

WHAT WE'RE BUILDING:
[2-3 sentence summary in your own words]

CURRENT STATE:
✅ Working: [count] items confirmed
🔄 In Progress: [list files that are in progress]
🗒️ Not Started: [list planned but untouched]

WHAT NOT TO RETRY:
[list every failed approach with its reason — this is critical]

OPEN QUESTIONS / BLOCKERS:
[list any blockers or unanswered questions]

NEXT STEP:
[exact next step if defined in the file]
[if not defined: "No next step defined — recommend reviewing 'What Has NOT Been Tried Yet' together before starting"]

════════════════════════════════════════════════
Ready to continue. What would you like to do?
```

### 4단계: 사용자 대기

자동으로 작업을 시작하지 마십시오. 어떤 파일도 만지지 마십시오. 사용자가 다음에 무엇을 할지 말할 때까지 기다리십시오.

세션 파일에 다음 단계가 명확하게 정의되어 있고 사용자가 "계속(continue)" 또는 "예(yes)" 등을 말하면 해당 다음 단계를 정확히 진행합니다.

다음 단계가 정의되지 않은 경우 사용자에게 어디서부터 시작할지 묻고, 선택적으로 "아직 시도하지 않은 것(What Has NOT Been Tried Yet)" 섹션에서 접근 방식을 제안합니다.

---

## 예외 상황 (Edge Cases)

**같은 날짜에 여러 세션이 있는 경우** (`2024-01-15-session.tmp`, `2024-01-15-abc123de-session.tmp`):
레거시 no-id 형식이든 현재 short-id 형식이든 상관없이 해당 날짜와 일치하는 가장 최근에 수정된 파일을 로드합니다.

**세션 파일이 더 이상 존재하지 않는 파일을 참조하는 경우:**
브리핑 중에 이를 언급합니다 — "⚠️ `path/to/file.ts` referenced in session but not found on disk."

**세션 파일이 7일보다 더 오래된 경우:**
차이를 언급합니다 — "⚠️ This session is from N days ago (threshold: 7 days). Things may have changed." — 그 다음 정상적으로 진행합니다.

**사용자가 파일 경로를 직접 제공하는 경우 (예: 팀원으로부터 전달받음):**
이를 읽고 동일한 브리핑 프로세스를 따릅니다 — 소스에 관계없이 형식은 동일합니다.

**세션 파일이 비어 있거나 손상된 경우:**
보고: "Session file found but appears empty or unreadable. You may need to create a new one with /save-session."

---

## 출력 예시

```
SESSION LOADED: /Users/you/.claude/sessions/2024-01-15-abc123de-session.tmp
════════════════════════════════════════════════

PROJECT: my-app — JWT Authentication

WHAT WE'RE BUILDING:
User authentication with JWT tokens stored in httpOnly cookies.
Register and login endpoints are partially done. Route protection
via middleware hasn't been started yet.

CURRENT STATE:
✅ Working: 3 items (register endpoint, JWT generation, password hashing)
🔄 In Progress: app/api/auth/login/route.ts (token works, cookie not set yet)
🗒️ Not Started: middleware.ts, app/login/page.tsx

WHAT NOT TO RETRY:
❌ Next-Auth — conflicts with custom Prisma adapter, threw adapter error on every request
❌ localStorage for JWT — causes SSR hydration mismatch, incompatible with Next.js

OPEN QUESTIONS / BLOCKERS:
- Does cookies().set() work inside a Route Handler or only Server Actions?

NEXT STEP:
In app/api/auth/login/route.ts — set the JWT as an httpOnly cookie using
cookies().set('token', jwt, { httpOnly: true, secure: true, sameSite: 'strict' })
then test with Postman for a Set-Cookie header in the response.

════════════════════════════════════════════════
Ready to continue. What would you like to do?
```

---

## 참고 사항

- 세션 파일을 로드할 때 절대 수정하지 마십시오 — 이는 읽기 전용 기록입니다.
- 브리핑 형식은 고정되어 있습니다 — 섹션이 비어 있더라도 건너뛰지 마십시오.
- "다시 시도하지 말아야 할 것(What Not To Retry)"은 "None"이라고 표시되더라도 항상 보여야 합니다 — 놓치기에는 너무 중요합니다.
- 재개한 후, 사용자는 새 날짜의 파일을 생성하기 위해 새 세션의 끝에서 `/save-session`을 다시 실행하기를 원할 수 있습니다.

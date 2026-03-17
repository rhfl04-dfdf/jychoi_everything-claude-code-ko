# Execute - 멀티 모델 협업 실행

멀티 모델 협업 실행 - 계획에서 프로토타입 획득 → Claude가 리팩토링 및 구현 → 멀티 모델 감사 및 전달.

$ARGUMENTS

---

## 핵심 프로토콜

- **언어 프로토콜**: 도구/모델과 상호작용 시 **영어** 사용, 사용자와는 사용자의 언어로 소통
- **코드 주권**: 외부 모델은 **파일 시스템 쓰기 권한 없음**, 모든 수정은 Claude가 담당
- **Dirty Prototype 리팩토링**: Codex/Gemini의 Unified Diff를 "dirty prototype"으로 취급하고, 프로덕션 수준의 코드로 리팩토링 필수
- **손절 메커니즘**: 현재 단계 출력이 검증될 때까지 다음 단계로 진행하지 않음
- **사전 조건**: `/ccg:plan` 출력에 사용자가 명시적으로 "Y"로 응답한 후에만 실행 (없는 경우, 먼저 확인 필수)

---

## 멀티 모델 호출 명세

**호출 구문** (병렬: `run_in_background: true` 사용):

```
# Resume session call (recommended) - Implementation Prototype
Bash({
  command: "~/.claude/bin/codeagent-wrapper {{LITE_MODE_FLAG}}--backend <codex|gemini> {{GEMINI_MODEL_FLAG}}resume <SESSION_ID> - \"$PWD\" <<'EOF'
ROLE_FILE: <role prompt path>
<TASK>
Requirement: <task description>
Context: <plan content + target files>
</TASK>
OUTPUT: Unified Diff Patch ONLY. Strictly prohibit any actual modifications.
EOF",
  run_in_background: true,
  timeout: 3600000,
  description: "Brief description"
})

# New session call - Implementation Prototype
Bash({
  command: "~/.claude/bin/codeagent-wrapper {{LITE_MODE_FLAG}}--backend <codex|gemini> {{GEMINI_MODEL_FLAG}}- \"$PWD\" <<'EOF'
ROLE_FILE: <role prompt path>
<TASK>
Requirement: <task description>
Context: <plan content + target files>
</TASK>
OUTPUT: Unified Diff Patch ONLY. Strictly prohibit any actual modifications.
EOF",
  run_in_background: true,
  timeout: 3600000,
  description: "Brief description"
})
```

**감사 호출 구문** (Code Review / Audit):

```
Bash({
  command: "~/.claude/bin/codeagent-wrapper {{LITE_MODE_FLAG}}--backend <codex|gemini> {{GEMINI_MODEL_FLAG}}resume <SESSION_ID> - \"$PWD\" <<'EOF'
ROLE_FILE: <role prompt path>
<TASK>
Scope: Audit the final code changes.
Inputs:
- The applied patch (git diff / final unified diff)
- The touched files (relevant excerpts if needed)
Constraints:
- Do NOT modify any files.
- Do NOT output tool commands that assume filesystem access.
</TASK>
OUTPUT:
1) A prioritized list of issues (severity, file, rationale)
2) Concrete fixes; if code changes are needed, include a Unified Diff Patch in a fenced code block.
EOF",
  run_in_background: true,
  timeout: 3600000,
  description: "Brief description"
})
```

**모델 파라미터 참고사항**:
- `{{GEMINI_MODEL_FLAG}}`: `--backend gemini` 사용 시 `--gemini-model gemini-3-pro-preview`로 대체 (뒤에 공백 주의); codex의 경우 빈 문자열 사용

**Role Prompts**:

| Phase | Codex | Gemini |
|-------|-------|--------|
| Implementation | `~/.claude/.ccg/prompts/codex/architect.md` | `~/.claude/.ccg/prompts/gemini/frontend.md` |
| Review | `~/.claude/.ccg/prompts/codex/reviewer.md` | `~/.claude/.ccg/prompts/gemini/reviewer.md` |

**세션 재사용**: `/ccg:plan`이 SESSION_ID를 제공한 경우, `resume <SESSION_ID>`로 컨텍스트를 재사용합니다.

**백그라운드 작업 대기** (최대 타임아웃 600000ms = 10분):

```
TaskOutput({ task_id: "<task_id>", block: true, timeout: 600000 })
```

**중요사항**:
- `timeout: 600000` 반드시 지정해야 합니다. 그렇지 않으면 기본값 30초로 인해 조기 타임아웃 발생
- 10분 후에도 완료되지 않으면 `TaskOutput`으로 계속 폴링하고, **절대 프로세스를 종료하지 말 것**
- 타임아웃으로 인해 대기를 건너뛴 경우, **반드시 `AskUserQuestion`을 호출하여 사용자에게 계속 기다릴지 또는 작업을 종료할지 물어볼 것**

---

## 실행 워크플로우

**실행 작업**: $ARGUMENTS

### Phase 0: 계획 읽기

`[Mode: Prepare]`

1. **입력 유형 식별**:
   - 계획 파일 경로 (예: `.claude/plan/xxx.md`)
   - 직접 작업 설명

2. **계획 내용 읽기**:
   - 계획 파일 경로가 제공된 경우 읽고 파싱
   - 추출 항목: 작업 유형, 구현 단계, 핵심 파일, SESSION_ID

3. **실행 전 확인**:
   - 입력이 "직접 작업 설명"이거나 계획에 `SESSION_ID` / 핵심 파일이 없는 경우: 먼저 사용자 확인
   - 사용자가 계획에 "Y"로 응답한 것을 확인할 수 없는 경우: 진행 전 다시 확인 필수

4. **작업 유형 라우팅**:

   | 작업 유형 | 감지 | 라우팅 |
   |-----------|------|--------|
   | **Frontend** | 페이지, 컴포넌트, UI, 스타일, 레이아웃 | Gemini |
   | **Backend** | API, 인터페이스, 데이터베이스, 로직, 알고리즘 | Codex |
   | **Fullstack** | 프론트엔드와 백엔드 모두 포함 | Codex ∥ Gemini 병렬 |

---

### Phase 1: 빠른 컨텍스트 검색

`[Mode: Retrieval]`

**ace-tool MCP가 사용 가능한 경우**, 빠른 컨텍스트 검색에 활용:

계획의 "Key Files" 목록을 기반으로 `mcp__ace-tool__search_context` 호출:

```
mcp__ace-tool__search_context({
  query: "<semantic query based on plan content, including key files, modules, function names>",
  project_root_path: "$PWD"
})
```

**검색 전략**:
- 계획의 "Key Files" 테이블에서 대상 경로 추출
- 진입점 파일, 의존성 모듈, 관련 타입 정의를 포함하는 시맨틱 쿼리 작성
- 결과가 불충분한 경우, 1-2회 재귀 검색 추가

**ace-tool MCP가 사용 불가한 경우**, Claude Code 내장 도구를 대체로 사용:
1. **Glob**: 계획의 "Key Files" 테이블에서 대상 파일 탐색 (예: `Glob("src/components/**/*.tsx")`)
2. **Grep**: 코드베이스 전반에서 핵심 심볼, 함수명, 타입 정의 검색
3. **Read**: 탐색된 파일을 읽어 완전한 컨텍스트 수집
4. **Task (Explore 에이전트)**: 광범위한 탐색을 위해 `subagent_type: "Explore"`로 `Task` 사용

**검색 후**:
- 검색된 코드 조각 정리
- 구현을 위한 완전한 컨텍스트 확인
- Phase 3으로 진행

---

### Phase 3: 프로토타입 획득

`[Mode: Prototype]`

**작업 유형에 따라 라우팅**:

#### Route A: Frontend/UI/Styles → Gemini

**제한**: 컨텍스트 < 32k 토큰

1. Gemini 호출 (`~/.claude/.ccg/prompts/gemini/frontend.md` 사용)
2. 입력: 계획 내용 + 검색된 컨텍스트 + 대상 파일
3. OUTPUT: `Unified Diff Patch ONLY. Strictly prohibit any actual modifications.`
4. **Gemini는 프론트엔드 디자인 권위자이며, CSS/React/Vue 프로토타입이 최종 시각적 기준**
5. **경고**: Gemini의 백엔드 로직 제안은 무시
6. 계획에 `GEMINI_SESSION`이 포함된 경우: `resume <GEMINI_SESSION>` 우선 사용

#### Route B: Backend/Logic/Algorithms → Codex

1. Codex 호출 (`~/.claude/.ccg/prompts/codex/architect.md` 사용)
2. 입력: 계획 내용 + 검색된 컨텍스트 + 대상 파일
3. OUTPUT: `Unified Diff Patch ONLY. Strictly prohibit any actual modifications.`
4. **Codex는 백엔드 로직 권위자이며, 논리적 추론과 디버그 능력 활용**
5. 계획에 `CODEX_SESSION`이 포함된 경우: `resume <CODEX_SESSION>` 우선 사용

#### Route C: Fullstack → 병렬 호출

1. **병렬 호출** (`run_in_background: true`):
   - Gemini: 프론트엔드 부분 처리
   - Codex: 백엔드 부분 처리
2. `TaskOutput`으로 두 모델의 완전한 결과 대기
3. 각각 계획의 `SESSION_ID`를 `resume`에 사용 (없는 경우 새 세션 생성)

**위 `Multi-Model Call Specification`의 `IMPORTANT` 지침을 따를 것**

---

### Phase 4: 코드 구현

`[Mode: Implement]`

**Claude가 Code Sovereign으로서 다음 단계를 실행합니다**:

1. **Diff 읽기**: Codex/Gemini가 반환한 Unified Diff Patch 파싱

2. **멘탈 샌드박스**:
   - 대상 파일에 Diff를 적용하는 시뮬레이션
   - 논리적 일관성 확인
   - 잠재적 충돌이나 부작용 파악

3. **리팩토링 및 정리**:
   - "dirty prototype"을 **가독성, 유지보수성, 엔터프라이즈 수준의 코드**로 리팩토링
   - 중복 코드 제거
   - 프로젝트의 기존 코드 표준 준수 확인
   - **불필요한 경우 주석/문서 생성 금지**, 코드가 자체적으로 설명되어야 함

4. **최소 범위**:
   - 변경은 요구사항 범위에만 제한
   - 부작용에 대한 **의무적 검토**
   - 대상화된 수정 실행

5. **변경 적용**:
   - Edit/Write 도구로 실제 수정 실행
   - **필요한 코드만 수정**, 사용자의 다른 기존 기능에 절대 영향 주지 않음

6. **자체 검증** (강력 권장):
   - 프로젝트의 기존 lint / typecheck / 테스트 실행 (최소 관련 범위 우선)
   - 실패 시: 먼저 회귀를 수정한 후 Phase 5로 진행

---

### Phase 5: 감사 및 전달

`[Mode: Audit]`

#### 5.1 자동 감사

**변경이 적용된 후, 즉시 병렬로** Codex와 Gemini에 Code Review를 요청해야 합니다:

1. **Codex Review** (`run_in_background: true`):
   - ROLE_FILE: `~/.claude/.ccg/prompts/codex/reviewer.md`
   - 입력: 변경된 Diff + 대상 파일
   - 초점: 보안, 성능, 에러 처리, 논리적 정확성

2. **Gemini Review** (`run_in_background: true`):
   - ROLE_FILE: `~/.claude/.ccg/prompts/gemini/reviewer.md`
   - 입력: 변경된 Diff + 대상 파일
   - 초점: 접근성, 디자인 일관성, 사용자 경험

`TaskOutput`으로 두 모델의 완전한 리뷰 결과 대기. 컨텍스트 일관성을 위해 Phase 3 세션 재사용 (`resume <SESSION_ID>`) 우선.

#### 5.2 통합 및 수정

1. Codex + Gemini 리뷰 피드백 종합
2. 신뢰 규칙에 따라 가중치 부여: 백엔드는 Codex 따름, 프론트엔드는 Gemini 따름
3. 필요한 수정 실행
4. 필요에 따라 Phase 5.1 반복 (위험이 수용 가능한 수준이 될 때까지)

#### 5.3 전달 확인

감사 통과 후, 사용자에게 보고:

```markdown
## Execution Complete

### Change Summary
| File | Operation | Description |
|------|-----------|-------------|
| path/to/file.ts | Modified | Description |

### Audit Results
- Codex: <Passed/Found N issues>
- Gemini: <Passed/Found N issues>

### Recommendations
1. [ ] <Suggested test steps>
2. [ ] <Suggested verification steps>
```

---

## 핵심 규칙

1. **코드 주권** – 모든 파일 수정은 Claude가 담당, 외부 모델은 쓰기 권한 없음
2. **Dirty Prototype 리팩토링** – Codex/Gemini 출력은 초안으로 취급하며, 반드시 리팩토링
3. **신뢰 규칙** – 백엔드는 Codex 따름, 프론트엔드는 Gemini 따름
4. **최소 변경** – 필요한 코드만 수정, 부작용 없음
5. **의무적 감사** – 변경 후 반드시 멀티 모델 Code Review 수행

---

## 사용법

```bash
# Execute plan file
/ccg:execute .claude/plan/feature-name.md

# Execute task directly (for plans already discussed in context)
/ccg:execute implement user authentication based on previous plan
```

---

## /ccg:plan과의 관계

1. `/ccg:plan`이 계획 + SESSION_ID 생성
2. 사용자가 "Y"로 확인
3. `/ccg:execute`가 계획을 읽고, SESSION_ID를 재사용하여 구현 실행

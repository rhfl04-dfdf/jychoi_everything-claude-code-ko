# Plan - 멀티 모델 협업 계획 수립

멀티 모델 협업 계획 수립 - 컨텍스트 검색 + 이중 모델 분석 → 단계별 구현 계획 생성.

$ARGUMENTS

---

## 핵심 프로토콜

- **언어 프로토콜**: 도구/모델과 상호작용 시 **영어** 사용, 사용자와는 사용자의 언어로 소통
- **병렬 처리 필수**: Codex/Gemini 호출은 반드시 `run_in_background: true` 사용 (단일 모델 호출 포함, 메인 스레드 블로킹 방지)
- **코드 주권**: 외부 모델은 **파일 시스템 쓰기 권한 없음**, 모든 수정은 Claude가 담당
- **손절 메커니즘**: 현재 단계 출력이 검증될 때까지 다음 단계로 진행하지 않음
- **계획만 수행**: 이 커맨드는 컨텍스트 읽기와 `.claude/plan/*` 계획 파일 작성은 허용하지만, **프로덕션 코드는 절대 수정하지 않음**

---

## 멀티 모델 호출 명세

**호출 구문** (병렬: `run_in_background: true` 사용):

```
Bash({
  command: "~/.claude/bin/codeagent-wrapper {{LITE_MODE_FLAG}}--backend <codex|gemini> {{GEMINI_MODEL_FLAG}}- \"$PWD\" <<'EOF'
ROLE_FILE: <role prompt path>
<TASK>
Requirement: <enhanced requirement>
Context: <retrieved project context>
</TASK>
OUTPUT: Step-by-step implementation plan with pseudo-code. DO NOT modify any files.
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
| Analysis | `~/.claude/.ccg/prompts/codex/analyzer.md` | `~/.claude/.ccg/prompts/gemini/analyzer.md` |
| Planning | `~/.claude/.ccg/prompts/codex/architect.md` | `~/.claude/.ccg/prompts/gemini/architect.md` |

**세션 재사용**: 각 호출은 `SESSION_ID: xxx`를 반환합니다 (일반적으로 wrapper가 출력). 이후 `/ccg:execute` 사용을 위해 **반드시 저장**해야 합니다.

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

**계획 작업**: $ARGUMENTS

### Phase 1: 전체 컨텍스트 검색

`[Mode: Research]`

#### 1.1 프롬프트 향상 (반드시 먼저 실행)

**ace-tool MCP가 사용 가능한 경우**, `mcp__ace-tool__enhance_prompt` 도구 호출:

```
mcp__ace-tool__enhance_prompt({
  prompt: "$ARGUMENTS",
  conversation_history: "<최근 5-10개 대화 턴>",
  project_root_path: "$PWD"
})
```

향상된 프롬프트를 기다린 후, **이후 모든 단계에서 원래 $ARGUMENTS를 향상된 결과로 대체**합니다.

**ace-tool MCP가 사용 불가한 경우**: 이 단계를 건너뛰고 모든 이후 단계에서 원래 `$ARGUMENTS`를 그대로 사용합니다.

#### 1.2 컨텍스트 검색

**ace-tool MCP가 사용 가능한 경우**, `mcp__ace-tool__search_context` 도구 호출:

```
mcp__ace-tool__search_context({
  query: "<향상된 요구사항 기반 시맨틱 쿼리>",
  project_root_path: "$PWD"
})
```

- 자연어 (Where/What/How)를 사용하여 시맨틱 쿼리 작성
- **가정에 기반한 답변은 절대 금지**

**ace-tool MCP가 사용 불가한 경우**, Claude Code 내장 도구를 대체로 사용:
1. **Glob**: 패턴으로 관련 파일 탐색 (예: `Glob("**/*.ts")`, `Glob("src/**/*.py")`)
2. **Grep**: 핵심 심볼, 함수명, 클래스 정의 검색 (예: `Grep("className|functionName")`)
3. **Read**: 탐색된 파일을 읽어 완전한 컨텍스트 수집
4. **Task (Explore 에이전트)**: 심층 탐색을 위해 `subagent_type: "Explore"`로 `Task` 사용하여 코드베이스 전반 검색

#### 1.3 완성도 확인

- 관련 클래스, 함수, 변수의 **완전한 정의와 서명** 획득 필수
- 컨텍스트가 불충분한 경우, **재귀 검색** 실행
- 출력 우선순위: 진입 파일 + 줄 번호 + 핵심 심볼명; 모호성 해소에 필요한 경우에만 최소한의 코드 조각 추가

#### 1.4 요구사항 정렬

- 요구사항에 여전히 모호함이 있는 경우, **반드시** 사용자를 위한 안내 질문 출력
- 요구사항 경계가 명확해질 때까지 (누락 없음, 중복 없음)

### Phase 2: 멀티 모델 협업 분석

`[Mode: Analysis]`

#### 2.1 입력 배분

**병렬로** Codex와 Gemini 호출 (`run_in_background: true`):

**원래 요구사항** (사전 설정된 의견 없이)을 두 모델에 배분:

1. **Codex 백엔드 분석**:
   - ROLE_FILE: `~/.claude/.ccg/prompts/codex/analyzer.md`
   - 초점: 기술적 타당성, 아키텍처 영향, 성능 고려사항, 잠재적 위험
   - OUTPUT: 다각도 솔루션 + 장단점 분석

2. **Gemini 프론트엔드 분석**:
   - ROLE_FILE: `~/.claude/.ccg/prompts/gemini/analyzer.md`
   - 초점: UI/UX 영향, 사용자 경험, 시각적 디자인
   - OUTPUT: 다각도 솔루션 + 장단점 분석

`TaskOutput`으로 두 모델의 완전한 결과 대기. **SESSION_ID 저장** (`CODEX_SESSION` 및 `GEMINI_SESSION`).

#### 2.2 상호 검증

관점을 통합하고 최적화를 위해 반복:

1. **합의점 파악** (강한 신호)
2. **차이점 파악** (가중치 부여 필요)
3. **강점 보완**: 백엔드 로직은 Codex 따름, 프론트엔드 디자인은 Gemini 따름
4. **논리적 추론**: 솔루션의 논리적 간격 제거

#### 2.3 (선택 사항이지만 권장) 이중 모델 계획 초안

Claude의 종합 계획에서 누락 위험을 줄이기 위해, 두 모델이 "계획 초안"을 병렬로 출력할 수 있습니다 (여전히 **파일 수정 불허**):

1. **Codex 계획 초안** (백엔드 권위):
   - ROLE_FILE: `~/.claude/.ccg/prompts/codex/architect.md`
   - OUTPUT: 단계별 계획 + 의사 코드 (초점: 데이터 흐름/엣지 케이스/에러 처리/테스트 전략)

2. **Gemini 계획 초안** (프론트엔드 권위):
   - ROLE_FILE: `~/.claude/.ccg/prompts/gemini/architect.md`
   - OUTPUT: 단계별 계획 + 의사 코드 (초점: 정보 아키텍처/인터랙션/접근성/시각적 일관성)

`TaskOutput`으로 두 모델의 완전한 결과 대기, 제안의 주요 차이점 기록.

#### 2.4 구현 계획 생성 (Claude 최종 버전)

두 분석을 종합하여 **단계별 구현 계획** 생성:

```markdown
## 구현 계획: <작업명>

### 작업 유형
- [ ] Frontend (→ Gemini)
- [ ] Backend (→ Codex)
- [ ] Fullstack (→ 병렬)

### 기술적 솔루션
<Codex + Gemini 분석에서 종합된 최적 솔루션>

### 구현 단계
1. <1단계> - 예상 산출물
2. <2단계> - 예상 산출물
...

### 핵심 파일
| 파일 | 작업 | 설명 |
|------|------|------|
| path/to/file.ts:L10-L50 | 수정 | 설명 |

### 위험 및 완화
| 위험 | 완화 방안 |
|------|----------|

### SESSION_ID (/ccg:execute 사용을 위해)
- CODEX_SESSION: <session_id>
- GEMINI_SESSION: <session_id>
```

### Phase 2 종료: 계획 전달 (실행 아님)

**`/ccg:plan`의 책임은 여기서 끝납니다. 반드시 다음 작업을 실행해야 합니다**:

1. 사용자에게 완전한 구현 계획 제시 (의사 코드 포함)
2. 계획을 `.claude/plan/<feature-name>.md`에 저장 (요구사항에서 기능 이름 추출, 예: `user-auth`, `payment-module`)
3. **굵은 글씨**로 프롬프트 출력 (반드시 실제 저장된 파일 경로 사용):

   ---
   **계획이 생성되어 `.claude/plan/actual-feature-name.md`에 저장되었습니다**

   **위의 계획을 검토해 주세요. 다음을 할 수 있습니다:**
   - **계획 수정**: 조정이 필요한 내용을 말씀해 주시면 계획을 업데이트하겠습니다
   - **계획 실행**: 다음 커맨드를 새 세션에 복사하여 붙여넣으세요

   ```
   /ccg:execute .claude/plan/actual-feature-name.md
   ```
   ---

   **참고**: 위의 `actual-feature-name.md`는 반드시 실제 저장된 파일명으로 대체해야 합니다!

4. **즉시 현재 응답 종료** (여기서 중단. 추가 도구 호출 없음.)

**절대 금지**:
- 사용자에게 "Y/N"을 묻고 자동 실행 (실행은 `/ccg:execute`의 책임)
- 프로덕션 코드에 대한 어떤 쓰기 작업
- `/ccg:execute` 또는 구현 작업의 자동 호출
- 사용자가 명시적으로 수정을 요청하지 않은 경우 모델 호출 계속 실행

---

## 계획 저장

계획 수립 완료 후, 계획 저장 위치:

- **첫 번째 계획**: `.claude/plan/<feature-name>.md`
- **반복 버전**: `.claude/plan/<feature-name>-v2.md`, `.claude/plan/<feature-name>-v3.md`...

계획 파일 작성은 사용자에게 계획을 제시하기 전에 완료해야 합니다.

---

## 계획 수정 흐름

사용자가 계획 수정을 요청하는 경우:

1. 사용자 피드백을 바탕으로 계획 내용 조정
2. `.claude/plan/<feature-name>.md` 파일 업데이트
3. 수정된 계획 다시 제시
4. 사용자에게 검토 또는 재실행 안내

---

## 다음 단계

사용자 승인 후, **수동으로** 실행:

```bash
/ccg:execute .claude/plan/<feature-name>.md
```

---

## 핵심 규칙

1. **계획만 수행, 구현 없음** – 이 커맨드는 코드 변경을 실행하지 않음
2. **Y/N 프롬프트 없음** – 계획만 제시하고, 다음 단계는 사용자가 결정
3. **신뢰 규칙** – 백엔드는 Codex 따름, 프론트엔드는 Gemini 따름
4. 외부 모델은 **파일 시스템 쓰기 권한 없음**
5. **SESSION_ID 인계** – 계획 끝에 반드시 `CODEX_SESSION` / `GEMINI_SESSION` 포함 (`/ccg:execute resume <SESSION_ID>` 사용을 위해)

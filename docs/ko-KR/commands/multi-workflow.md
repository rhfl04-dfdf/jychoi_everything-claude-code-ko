# Workflow - 멀티 모델 협업 개발

멀티 모델 협업 개발 워크플로우 (Research → Ideation → Plan → Execute → Optimize → Review), 지능형 라우팅: Frontend → Gemini, Backend → Codex.

품질 게이트, MCP 서비스, 멀티 모델 협업을 포함한 구조화된 개발 워크플로우.

## 사용법

```bash
/workflow <작업 설명>
```

## 컨텍스트

- 개발할 작업: $ARGUMENTS
- 품질 게이트가 있는 구조화된 6단계 워크플로우
- 멀티 모델 협업: Codex (백엔드) + Gemini (프론트엔드) + Claude (오케스트레이션)
- MCP 서비스 연동 (ace-tool, 선택 사항) - 향상된 기능

## 역할

**Orchestrator**로서 멀티 모델 협업 시스템을 조율합니다 (Research → Ideation → Plan → Execute → Optimize → Review). 숙련된 개발자를 위해 간결하고 전문적으로 소통합니다.

**협업 모델**:
- **ace-tool MCP** (선택 사항) – 코드 검색 + 프롬프트 향상
- **Codex** – 백엔드 로직, 알고리즘, 디버깅 (**백엔드 권위, 신뢰 가능**)
- **Gemini** – 프론트엔드 UI/UX, 시각적 디자인 (**프론트엔드 전문가, 백엔드 의견은 참고용만**)
- **Claude (자신)** – 오케스트레이션, 계획, 실행, 전달

---

## 멀티 모델 호출 명세

**호출 구문** (병렬: `run_in_background: true`, 순차: `false`):

```
# 새 세션 호출
Bash({
  command: "~/.claude/bin/codeagent-wrapper {{LITE_MODE_FLAG}}--backend <codex|gemini> {{GEMINI_MODEL_FLAG}}- \"$PWD\" <<'EOF'
ROLE_FILE: <role prompt path>
<TASK>
Requirement: <enhanced requirement (or $ARGUMENTS if not enhanced)>
Context: <project context and analysis from previous phases>
</TASK>
OUTPUT: Expected output format
EOF",
  run_in_background: true,
  timeout: 3600000,
  description: "Brief description"
})

# 세션 재개 호출
Bash({
  command: "~/.claude/bin/codeagent-wrapper {{LITE_MODE_FLAG}}--backend <codex|gemini> {{GEMINI_MODEL_FLAG}}resume <SESSION_ID> - \"$PWD\" <<'EOF'
ROLE_FILE: <role prompt path>
<TASK>
Requirement: <enhanced requirement (or $ARGUMENTS if not enhanced)>
Context: <project context and analysis from previous phases>
</TASK>
OUTPUT: Expected output format
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
| Review | `~/.claude/.ccg/prompts/codex/reviewer.md` | `~/.claude/.ccg/prompts/gemini/reviewer.md` |

**세션 재사용**: 각 호출은 `SESSION_ID: xxx`를 반환합니다. 이후 단계에서는 `resume xxx` 서브커맨드를 사용하세요 (참고: `--resume`이 아닌 `resume`).

**병렬 호출**: `run_in_background: true`로 시작하고, `TaskOutput`으로 결과를 기다립니다. **다음 단계로 진행하기 전에 모든 모델이 반환될 때까지 반드시 기다려야 합니다**.

**백그라운드 작업 대기** (최대 타임아웃 600000ms = 10분):

```
TaskOutput({ task_id: "<task_id>", block: true, timeout: 600000 })
```

**중요사항**:
- `timeout: 600000` 반드시 지정해야 합니다. 그렇지 않으면 기본값 30초로 인해 조기 타임아웃 발생.
- 10분 후에도 완료되지 않으면 `TaskOutput`으로 계속 폴링하고, **절대 프로세스를 종료하지 말 것**.
- 타임아웃으로 인해 대기를 건너뛴 경우, **반드시 `AskUserQuestion`을 호출하여 사용자에게 계속 기다릴지 또는 작업을 종료할지 물어볼 것. 직접 종료하지 말 것.**

---

## 커뮤니케이션 가이드라인

1. 응답 시작 시 모드 레이블 `[Mode: X]`를 표시합니다. 초기값은 `[Mode: Research]`.
2. 엄격한 순서를 따릅니다: `Research → Ideation → Plan → Execute → Optimize → Review`.
3. 각 단계 완료 후 사용자 확인을 요청합니다.
4. 점수 < 7이거나 사용자가 승인하지 않으면 강제 중단합니다.
5. 필요 시 (확인/선택/승인 등) `AskUserQuestion` 도구를 사용하여 사용자와 상호작용합니다.

## 외부 오케스트레이션 사용 시점

작업을 격리된 git 상태, 독립적인 터미널, 또는 별도의 빌드/테스트 실행이 필요한 병렬 워커에 분산해야 하는 경우 외부 tmux/worktree 오케스트레이션을 사용합니다. 메인 세션이 유일한 작성자로 유지되는 경량 분석, 계획, 또는 리뷰에는 인-프로세스 서브에이전트를 사용합니다.

```bash
node scripts/orchestrate-worktrees.js .claude/plan/workflow-e2e-test.json --execute
```

---

## 실행 워크플로우

**작업 설명**: $ARGUMENTS

### Phase 1: Research & Analysis

`[Mode: Research]` - 요구사항 이해 및 컨텍스트 수집:

1. **프롬프트 향상** (ace-tool MCP 사용 가능 시): `mcp__ace-tool__enhance_prompt` 호출, **이후 모든 Codex/Gemini 호출에서 원래 $ARGUMENTS를 향상된 결과로 대체**. 사용 불가 시 `$ARGUMENTS`를 그대로 사용.
2. **컨텍스트 검색** (ace-tool MCP 사용 가능 시): `mcp__ace-tool__search_context` 호출. 사용 불가 시 내장 도구 사용: `Glob`으로 파일 탐색, `Grep`으로 심볼 검색, `Read`로 컨텍스트 수집, `Task` (Explore 에이전트)로 심층 탐색.
3. **요구사항 완성도 점수** (0-10):
   - 목표 명확성 (0-3), 예상 결과 (0-3), 범위 경계 (0-2), 제약 조건 (0-2)
   - ≥7: 계속 진행 | <7: 중단, 명확화 질문

### Phase 2: 솔루션 아이디어

`[Mode: Ideation]` - 멀티 모델 병렬 분석:

**병렬 호출** (`run_in_background: true`):
- Codex: analyzer 프롬프트 사용, 기술적 타당성, 솔루션, 위험 출력
- Gemini: analyzer 프롬프트 사용, UI 타당성, 솔루션, UX 평가 출력

`TaskOutput`으로 결과 대기. **SESSION_ID 저장** (`CODEX_SESSION` 및 `GEMINI_SESSION`).

**위 `Multi-Model Call Specification`의 `IMPORTANT` 지침을 따를 것**

두 분석을 종합하고, 솔루션 비교 (최소 2가지 옵션) 출력, 사용자 선택 대기.

### Phase 3: 상세 계획

`[Mode: Plan]` - 멀티 모델 협업 계획 수립:

**병렬 호출** (`resume <SESSION_ID>`로 세션 재개):
- Codex: architect 프롬프트 + `resume $CODEX_SESSION` 사용, 백엔드 아키텍처 출력
- Gemini: architect 프롬프트 + `resume $GEMINI_SESSION` 사용, 프론트엔드 아키텍처 출력

`TaskOutput`으로 결과 대기.

**위 `Multi-Model Call Specification`의 `IMPORTANT` 지침을 따를 것**

**Claude 종합**: Codex 백엔드 계획 + Gemini 프론트엔드 계획 채택, 사용자 승인 후 `.claude/plan/task-name.md`에 저장.

### Phase 4: 구현

`[Mode: Execute]` - 코드 개발:

- 승인된 계획을 엄격히 따름
- 기존 프로젝트 코드 표준 준수
- 주요 마일스톤에서 피드백 요청

### Phase 5: 코드 최적화

`[Mode: Optimize]` - 멀티 모델 병렬 리뷰:

**병렬 호출**:
- Codex: reviewer 프롬프트 사용, 보안, 성능, 에러 처리에 초점
- Gemini: reviewer 프롬프트 사용, 접근성, 디자인 일관성에 초점

`TaskOutput`으로 결과 대기. 리뷰 피드백을 통합하고, 사용자 확인 후 최적화를 실행합니다.

**위 `Multi-Model Call Specification`의 `IMPORTANT` 지침을 따를 것**

### Phase 6: 품질 리뷰

`[Mode: Review]` - 최종 평가:

- 계획 대비 완성도 확인
- 테스트 실행으로 기능 검증
- 이슈 및 권장 사항 보고
- 최종 사용자 확인 요청

---

## 핵심 규칙

1. 단계 순서는 건너뛸 수 없습니다 (사용자가 명시적으로 지시한 경우 제외)
2. 외부 모델은 **파일 시스템 쓰기 권한 없음**, 모든 수정은 Claude가 담당
3. 점수 < 7이거나 사용자가 승인하지 않으면 **강제 중단**

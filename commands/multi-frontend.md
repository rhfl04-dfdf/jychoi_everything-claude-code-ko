# Frontend - 프론트엔드 중심 개발

프론트엔드 중심 워크플로우 (Research → Ideation → Plan → Execute → Optimize → Review), Gemini 주도.

## 사용법

```bash
/frontend <UI task description>
```

## 컨텍스트

- 프론트엔드 작업: $ARGUMENTS
- Gemini 주도, Codex는 보조 참고용
- 적용 범위: 컴포넌트 설계, 반응형 레이아웃, UI 애니메이션, 스타일 최적화

## 역할

**Frontend Orchestrator**로서 UI/UX 작업을 위한 멀티 모델 협업을 조율합니다 (Research → Ideation → Plan → Execute → Optimize → Review).

**협업 모델**:
- **Gemini** – 프론트엔드 UI/UX (**프론트엔드 권위, 신뢰 가능**)
- **Codex** – 백엔드 관점 (**프론트엔드 의견은 참고용만**)
- **Claude (자신)** – 오케스트레이션, 계획, 실행, 전달

---

## 멀티 모델 호출 명세

**호출 구문**:

```
# New session call
Bash({
  command: "~/.claude/bin/codeagent-wrapper {{LITE_MODE_FLAG}}--backend gemini --gemini-model gemini-3-pro-preview - \"$PWD\" <<'EOF'
ROLE_FILE: <role prompt path>
<TASK>
Requirement: <enhanced requirement (or $ARGUMENTS if not enhanced)>
Context: <project context and analysis from previous phases>
</TASK>
OUTPUT: Expected output format
EOF",
  run_in_background: false,
  timeout: 3600000,
  description: "Brief description"
})

# Resume session call
Bash({
  command: "~/.claude/bin/codeagent-wrapper {{LITE_MODE_FLAG}}--backend gemini --gemini-model gemini-3-pro-preview resume <SESSION_ID> - \"$PWD\" <<'EOF'
ROLE_FILE: <role prompt path>
<TASK>
Requirement: <enhanced requirement (or $ARGUMENTS if not enhanced)>
Context: <project context and analysis from previous phases>
</TASK>
OUTPUT: Expected output format
EOF",
  run_in_background: false,
  timeout: 3600000,
  description: "Brief description"
})
```

**Role Prompts**:

| Phase | Gemini |
|-------|--------|
| Analysis | `~/.claude/.ccg/prompts/gemini/analyzer.md` |
| Planning | `~/.claude/.ccg/prompts/gemini/architect.md` |
| Review | `~/.claude/.ccg/prompts/gemini/reviewer.md` |

**세션 재사용**: 각 호출은 `SESSION_ID: xxx`를 반환합니다. 이후 단계에서는 `resume xxx`를 사용하세요. Phase 2에서 `GEMINI_SESSION`을 저장하고, Phase 3과 5에서 `resume`을 사용하세요.

---

## 커뮤니케이션 가이드라인

1. 응답 시작 시 모드 레이블 `[Mode: X]`를 표시합니다. 초기값은 `[Mode: Research]`
2. 엄격한 순서를 따릅니다: `Research → Ideation → Plan → Execute → Optimize → Review`
3. 필요 시 (확인/선택/승인 등) `AskUserQuestion` 도구를 사용하여 사용자와 상호작용합니다

---

## 핵심 워크플로우

### Phase 0: 프롬프트 향상 (선택 사항)

`[Mode: Prepare]` - ace-tool MCP가 사용 가능하면 `mcp__ace-tool__enhance_prompt`를 호출하고, **이후 Gemini 호출에서 원래 $ARGUMENTS를 향상된 결과로 대체**합니다. 사용 불가 시 `$ARGUMENTS`를 그대로 사용합니다.

### Phase 1: Research

`[Mode: Research]` - 요구사항 이해 및 컨텍스트 수집

1. **코드 검색** (ace-tool MCP 사용 가능 시): `mcp__ace-tool__search_context`를 호출하여 기존 컴포넌트, 스타일, 디자인 시스템을 검색합니다. 사용 불가 시 내장 도구 사용: `Glob`으로 파일 탐색, `Grep`으로 컴포넌트/스타일 검색, `Read`로 컨텍스트 수집, `Task` (Explore 에이전트)로 심층 탐색.
2. 요구사항 완성도 점수 (0-10): >=7이면 계속 진행, <7이면 중단 후 보완

### Phase 2: Ideation

`[Mode: Ideation]` - Gemini 주도 분석

**반드시 Gemini를 호출해야 합니다** (위의 호출 명세 참고):
- ROLE_FILE: `~/.claude/.ccg/prompts/gemini/analyzer.md`
- Requirement: 향상된 요구사항 (향상되지 않은 경우 $ARGUMENTS)
- Context: Phase 1의 프로젝트 컨텍스트
- OUTPUT: UI 타당성 분석, 권장 솔루션 (최소 2가지), UX 평가

**SESSION_ID 저장** (`GEMINI_SESSION`) - 이후 단계 재사용을 위해.

솔루션 (최소 2가지) 출력 후, 사용자 선택을 기다립니다.

### Phase 3: Planning

`[Mode: Plan]` - Gemini 주도 계획 수립

**반드시 Gemini를 호출해야 합니다** (`resume <GEMINI_SESSION>`으로 세션 재사용):
- ROLE_FILE: `~/.claude/.ccg/prompts/gemini/architect.md`
- Requirement: 사용자가 선택한 솔루션
- Context: Phase 2의 분석 결과
- OUTPUT: 컴포넌트 구조, UI 흐름, 스타일링 접근 방식

Claude가 계획을 종합하고, 사용자 승인 후 `.claude/plan/task-name.md`에 저장합니다.

### Phase 4: 구현

`[Mode: Execute]` - 코드 개발

- 승인된 계획을 엄격히 따름
- 기존 프로젝트 디자인 시스템 및 코드 표준 준수
- 반응형 디자인, 접근성 보장

### Phase 5: 최적화

`[Mode: Optimize]` - Gemini 주도 리뷰

**반드시 Gemini를 호출해야 합니다** (위의 호출 명세 참고):
- ROLE_FILE: `~/.claude/.ccg/prompts/gemini/reviewer.md`
- Requirement: 다음 프론트엔드 코드 변경 사항 리뷰
- Context: git diff 또는 코드 내용
- OUTPUT: 접근성, 반응형 디자인, 성능, 디자인 일관성 이슈 목록

리뷰 피드백을 통합하고, 사용자 확인 후 최적화를 실행합니다.

### Phase 6: 품질 리뷰

`[Mode: Review]` - 최종 평가

- 계획 대비 완성도 확인
- 반응형 디자인 및 접근성 검증
- 이슈 및 권장 사항 보고

---

## 핵심 규칙

1. **Gemini 프론트엔드 의견은 신뢰 가능합니다**
2. **Codex 프론트엔드 의견은 참고용만입니다**
3. 외부 모델은 **파일 시스템 쓰기 권한 없음**
4. Claude가 모든 코드 작성 및 파일 작업을 담당합니다

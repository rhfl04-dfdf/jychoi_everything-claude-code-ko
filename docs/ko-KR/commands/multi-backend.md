# Backend - 백엔드 중심 개발

백엔드 중심 워크플로우 (Research → Ideation → Plan → Execute → Optimize → Review), Codex 주도.

## 사용법

```bash
/backend <백엔드 작업 설명>
```

## 컨텍스트

- 백엔드 작업: $ARGUMENTS
- Codex 주도, Gemini는 보조 참고용
- 적용 범위: API 설계, 알고리즘 구현, 데이터베이스 최적화, 비즈니스 로직

## 역할

**Backend Orchestrator**로서 서버 사이드 작업을 위한 멀티 모델 협업을 조율합니다 (Research → Ideation → Plan → Execute → Optimize → Review).

**협업 모델**:
- **Codex** – 백엔드 로직, 알고리즘 (**백엔드 권위, 신뢰 가능**)
- **Gemini** – 프론트엔드 관점 (**백엔드 의견은 참고용만**)
- **Claude (자신)** – 오케스트레이션, 계획, 실행, 전달

---

## 멀티 모델 호출 명세

**호출 구문**:

```
# 새 세션 호출
Bash({
  command: "~/.claude/bin/codeagent-wrapper {{LITE_MODE_FLAG}}--backend codex - \"$PWD\" <<'EOF'
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

# 세션 재개 호출
Bash({
  command: "~/.claude/bin/codeagent-wrapper {{LITE_MODE_FLAG}}--backend codex resume <SESSION_ID> - \"$PWD\" <<'EOF'
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

| Phase | Codex |
|-------|-------|
| Analysis | `~/.claude/.ccg/prompts/codex/analyzer.md` |
| Planning | `~/.claude/.ccg/prompts/codex/architect.md` |
| Review | `~/.claude/.ccg/prompts/codex/reviewer.md` |

**세션 재사용**: 각 호출은 `SESSION_ID: xxx`를 반환합니다. 이후 단계에서는 `resume xxx`를 사용하세요. Phase 2에서 `CODEX_SESSION`을 저장하고, Phase 3과 5에서 `resume`을 사용하세요.

---

## 커뮤니케이션 가이드라인

1. 응답 시작 시 모드 레이블 `[Mode: X]`를 표시합니다. 초기값은 `[Mode: Research]`
2. 엄격한 순서를 따릅니다: `Research → Ideation → Plan → Execute → Optimize → Review`
3. 필요 시 (확인/선택/승인 등) `AskUserQuestion` 도구를 사용하여 사용자와 상호작용합니다

---

## 핵심 워크플로우

### Phase 0: 프롬프트 향상 (선택 사항)

`[Mode: Prepare]` - ace-tool MCP가 사용 가능하면 `mcp__ace-tool__enhance_prompt`를 호출하고, **이후 Codex 호출에서 원래 $ARGUMENTS를 향상된 결과로 대체**합니다. 사용 불가 시 `$ARGUMENTS`를 그대로 사용합니다.

### Phase 1: Research

`[Mode: Research]` - 요구사항 이해 및 컨텍스트 수집

1. **코드 검색** (ace-tool MCP 사용 가능 시): `mcp__ace-tool__search_context`를 호출하여 기존 API, 데이터 모델, 서비스 아키텍처를 검색합니다. 사용 불가 시 내장 도구 사용: `Glob`으로 파일 탐색, `Grep`으로 심볼/API 검색, `Read`로 컨텍스트 수집, `Task` (Explore 에이전트)로 심층 탐색.
2. 요구사항 완성도 점수 (0-10): >=7이면 계속 진행, <7이면 중단 후 보완

### Phase 2: Ideation

`[Mode: Ideation]` - Codex 주도 분석

**반드시 Codex를 호출해야 합니다** (위의 호출 명세 참고):
- ROLE_FILE: `~/.claude/.ccg/prompts/codex/analyzer.md`
- Requirement: 향상된 요구사항 (향상되지 않은 경우 $ARGUMENTS)
- Context: Phase 1의 프로젝트 컨텍스트
- OUTPUT: 기술적 타당성 분석, 권장 솔루션 (최소 2가지), 위험 평가

**SESSION_ID 저장** (`CODEX_SESSION`) - 이후 단계 재사용을 위해.

솔루션 (최소 2가지) 출력 후, 사용자 선택을 기다립니다.

### Phase 3: Planning

`[Mode: Plan]` - Codex 주도 계획 수립

**반드시 Codex를 호출해야 합니다** (`resume <CODEX_SESSION>`으로 세션 재사용):
- ROLE_FILE: `~/.claude/.ccg/prompts/codex/architect.md`
- Requirement: 사용자가 선택한 솔루션
- Context: Phase 2의 분석 결과
- OUTPUT: 파일 구조, 함수/클래스 설계, 의존성 관계

Claude가 계획을 종합하고, 사용자 승인 후 `.claude/plan/task-name.md`에 저장합니다.

### Phase 4: 구현

`[Mode: Execute]` - 코드 개발

- 승인된 계획을 엄격히 따름
- 기존 프로젝트 코드 표준 준수
- 에러 처리, 보안, 성능 최적화 보장

### Phase 5: 최적화

`[Mode: Optimize]` - Codex 주도 리뷰

**반드시 Codex를 호출해야 합니다** (위의 호출 명세 참고):
- ROLE_FILE: `~/.claude/.ccg/prompts/codex/reviewer.md`
- Requirement: 다음 백엔드 코드 변경 사항 리뷰
- Context: git diff 또는 코드 내용
- OUTPUT: 보안, 성능, 에러 처리, API 준수 이슈 목록

리뷰 피드백을 통합하고, 사용자 확인 후 최적화를 실행합니다.

### Phase 6: 품질 리뷰

`[Mode: Review]` - 최종 평가

- 계획 대비 완성도 확인
- 테스트 실행으로 기능 검증
- 이슈 및 권장 사항 보고

---

## 핵심 규칙

1. **Codex 백엔드 의견은 신뢰 가능합니다**
2. **Gemini 백엔드 의견은 참고용만입니다**
3. 외부 모델은 **파일 시스템 쓰기 권한 없음**
4. Claude가 모든 코드 작성 및 파일 작업을 담당합니다

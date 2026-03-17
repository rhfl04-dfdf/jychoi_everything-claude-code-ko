# 변경 로그

## 1.8.0 - 2026-03-04

### 주요 내용

- 신뢰성, 평가 원칙, 자율 루프 운영에 중점을 둔 harness 우선 릴리스.
- Hook 런타임이 프로필 기반 제어 및 특정 hook 비활성화를 지원.
- NanoClaw v2에 모델 라우팅, skill 핫 로드, 분기, 검색, 압축, 내보내기, 지표 기능 추가.

### 핵심

- 새 command 추가: `/harness-audit`, `/loop-start`, `/loop-status`, `/quality-gate`, `/model-route`.
- 새 skill 추가:
  - `agent-harness-construction`
  - `agentic-engineering`
  - `ralphinho-rfc-pipeline`
  - `ai-first-engineering`
  - `enterprise-agent-ops`
  - `nanoclaw-repl`
  - `continuous-agent-loop`
- 새 agent 추가:
  - `harness-optimizer`
  - `loop-operator`

### Hook 신뢰성

- 강력한 폴백 검색으로 SessionStart 루트 해석 수정.
- 트랜스크립트 페이로드가 사용 가능한 `Stop`으로 세션 요약 지속성 이동.
- quality-gate 및 cost-tracker hook 추가.
- 취약한 인라인 hook 한 줄짜리를 전용 스크립트 파일로 교체.
- `ECC_HOOK_PROFILE` 및 `ECC_DISABLED_HOOKS` 컨트롤 추가.

### 크로스 플랫폼

- 문서 경고 로직에서 Windows 안전 경로 처리 개선.
- 비대화형 중단을 방지하기 위해 observer 루프 동작 강화.

### 참고

- `autonomous-loops`는 한 릴리스 동안 호환성 별칭으로 유지됨; `continuous-agent-loop`이 정식 이름.

### 기여

- [zarazhangrui](https://github.com/zarazhangrui)에서 영감을 받음
- homunculus는 [humanplane](https://github.com/humanplane)에서 영감을 받음

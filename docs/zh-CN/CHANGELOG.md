# 변경 이력

## 1.8.0 - 2026-03-04

### 하이라이트

* 안정성, 평가 규정 및 자율 루프 운영을 핵심으로 하는 첫 번째 릴리스.
* Hook 런타임이 프로필 기반 제어와 특정 Hook 비활성화를 지원.
* NanoClaw v2에 모델 라우팅, skill 핫 로딩, 분기, 검색, 압축, 내보내기 및 메트릭 기능 추가.

### 코어

* 새 command: `/harness-audit`, `/loop-start`, `/loop-status`, `/quality-gate`, `/model-route`.
* 새 skill:
  * `agent-harness-construction`
  * `agentic-engineering`
  * `ralphinho-rfc-pipeline`
  * `ai-first-engineering`
  * `enterprise-agent-ops`
  * `nanoclaw-repl`
  * `continuous-agent-loop`
* 새 agent:
  * `harness-optimizer`
  * `loop-operator`

### Hook 안정성

* SessionStart의 루트 경로 해석을 수정하고, 견고한 폴백 검색을 추가.
* 세션 요약 영속화를 `Stop`으로 이동, 이곳에서 트랜스크립트 페이로드를 사용 가능.
* 품질 게이트 및 비용 추적 hook 추가.
* 취약한 인라인 원라이너 hook을 전용 스크립트 파일로 교체.
* `ECC_HOOK_PROFILE` 및 `ECC_DISABLED_HOOKS` 제어 추가.

### 크로스 플랫폼

* 문서 경고 로직에서 Windows 보안 경로 처리 개선.
* 비대화형 환경에서 행(hang)을 방지하기 위해 옵저버 루프 동작을 강화.

### 비고

* `autonomous-loops`는 호환성 별칭으로 한 버전 동안 유지됨; `continuous-agent-loop`이 정식 명칭.

### 감사의 말

* [zarazhangrui](https://github.com/zarazhangrui)에서 영감을 받음
* homunculus는 [humanplane](https://github.com/humanplane)에서 영감을 받음

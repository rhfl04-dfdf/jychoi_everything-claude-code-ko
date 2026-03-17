# Everything Claude Code (ECC) — Agent 지침

이것은 소프트웨어 개발을 위한 16개의 전문 agent, 65개 이상의 skill, 40개의 command, 그리고 자동화된 hook 워크플로우를 제공하는 **프로덕션 수준의 AI 코딩 플러그인**입니다.

## 핵심 원칙

1. **Agent 우선** — 도메인 작업은 전문 agent에 위임
2. **테스트 주도** — 구현 전 테스트 작성, 80% 이상 커버리지 필수
3. **보안 우선** — 보안 타협 금지; 모든 입력 값 검증
4. **불변성** — 항상 새 객체 생성, 기존 객체 변경 금지
5. **실행 전 계획** — 복잡한 기능은 코드 작성 전 계획 수립

## 사용 가능한 Agent

| Agent | 목적 | 사용 시점 |
|-------|---------|-------------|
| planner | 구현 계획 수립 | 복잡한 기능, 리팩토링 |
| architect | 시스템 설계 및 확장성 | 아키텍처 결정 |
| tdd-guide | 테스트 주도 개발 | 새 기능, 버그 수정 |
| code-reviewer | 코드 품질 및 유지보수성 | 코드 작성/수정 후 |
| security-reviewer | 취약점 탐지 | 커밋 전, 민감한 코드 |
| build-error-resolver | 빌드/타입 오류 수정 | 빌드 실패 시 |
| e2e-runner | Playwright 종단 간 테스트 | 핵심 사용자 흐름 |
| refactor-cleaner | 데드 코드 정리 | 코드 유지보수 |
| doc-updater | 문서 및 코드맵 | 문서 업데이트 |
| go-reviewer | Go 코드 리뷰 | Go 프로젝트 |
| go-build-resolver | Go 빌드 오류 | Go 빌드 실패 |
| database-reviewer | PostgreSQL/Supabase 전문가 | 스키마 설계, 쿼리 최적화 |
| python-reviewer | Python 코드 리뷰 | Python 프로젝트 |
| chief-of-staff | 커뮤니케이션 분류 및 초안 | 이메일, Slack, LINE, Messenger 멀티채널 |
| loop-operator | 자율 루프 실행 | 루프 안전 실행, 중단 모니터링, 개입 |
| harness-optimizer | Harness 설정 조정 | 신뢰성, 비용, 처리량 |

## Agent 오케스트레이션

사용자 요청 없이도 proactive하게 agent를 사용:
- 복잡한 기능 요청 → **planner**
- 방금 작성/수정한 코드 → **code-reviewer**
- 버그 수정 또는 새 기능 → **tdd-guide**
- 아키텍처 결정 → **architect**
- 보안 민감 코드 → **security-reviewer**
- 멀티채널 커뮤니케이션 분류 → **chief-of-staff**
- 자율 루프 / 루프 모니터링 → **loop-operator**
- Harness 설정 신뢰성 및 비용 → **harness-optimizer**

독립적인 작업은 병렬 실행 — 여러 agent를 동시에 실행.

## 보안 지침

**모든 커밋 전:**
- 하드코딩된 시크릿 없음 (API 키, 비밀번호, 토큰)
- 모든 사용자 입력 검증
- SQL 인젝션 방지 (파라미터화된 쿼리)
- XSS 방지 (HTML 새니타이징)
- CSRF 보호 활성화
- 인증/인가 검증
- 모든 엔드포인트 속도 제한
- 에러 메시지에 민감한 데이터 노출 금지

**시크릿 관리:** 시크릿 절대 하드코딩 금지. 환경 변수 또는 시크릿 매니저 사용. 시작 시 필수 시크릿 검증. 노출된 시크릿은 즉시 교체.

**보안 문제 발견 시:** 중단 → security-reviewer agent 사용 → CRITICAL 문제 수정 → 노출된 시크릿 교체 → 유사 문제 코드베이스 검토.

## 코딩 스타일

**불변성 (CRITICAL):** 항상 새 객체 생성, 변경 금지. 변경 사항이 적용된 새 복사본 반환.

**파일 구성:** 대형 파일보다 소형 파일 다수. 일반적으로 200-400줄, 최대 800줄. 타입별 아닌 기능/도메인별 구성. 높은 응집도, 낮은 결합도.

**오류 처리:** 모든 레벨에서 오류 처리. UI 코드에서 사용자 친화적 메시지 제공. 서버 측 상세 컨텍스트 로깅. 오류 무시 금지.

**입력 검증:** 시스템 경계에서 모든 사용자 입력 검증. 스키마 기반 검증 사용. 명확한 메시지와 함께 빠른 실패. 외부 데이터 신뢰 금지.

**코드 품질 체크리스트:**
- 함수 소형(<50줄), 파일 집중(<800줄)
- 깊은 중첩 없음(>4 단계)
- 적절한 오류 처리, 하드코딩된 값 없음
- 읽기 쉽고 잘 명명된 식별자

## 테스트 요구사항

**최소 커버리지: 80%**

테스트 유형 (모두 필수):
1. **단위 테스트** — 개별 함수, 유틸리티, 컴포넌트
2. **통합 테스트** — API 엔드포인트, 데이터베이스 작업
3. **E2E 테스트** — 핵심 사용자 흐름

**TDD 워크플로우 (필수):**
1. 테스트 먼저 작성 (RED) — 테스트가 실패해야 함
2. 최소 구현 작성 (GREEN) — 테스트가 통과해야 함
3. 리팩토링 (IMPROVE) — 커버리지 80%+ 검증

실패 문제 해결: 테스트 격리 확인 → 목 검증 → 구현 수정 (테스트가 잘못된 경우가 아니면 테스트 수정 금지).

## 개발 워크플로우

1. **계획** — planner agent 사용, 의존성 및 위험 식별, 단계별 분해
2. **TDD** — tdd-guide agent 사용, 테스트 먼저 작성, 구현, 리팩토링
3. **리뷰** — code-reviewer agent 즉시 사용, CRITICAL/HIGH 문제 해결
4. **올바른 위치에 지식 기록**
   - 개인 디버깅 메모, 설정, 임시 컨텍스트 → 자동 메모리
   - 팀/프로젝트 지식 (아키텍처 결정, API 변경, 런북) → 프로젝트 기존 문서 구조
   - 현재 작업에서 이미 관련 문서나 코드 주석이 생성된다면 동일한 정보 중복 기록 금지
   - 명확한 프로젝트 문서 위치가 없으면 새 최상위 파일 생성 전 문의
5. **커밋** — Conventional commits 형식, 포괄적인 PR 요약

## Git 워크플로우

**커밋 형식:** `<type>: <description>` — 타입: feat, fix, refactor, docs, test, chore, perf, ci

**PR 워크플로우:** 전체 커밋 이력 분석 → 포괄적 요약 초안 → 테스트 계획 포함 → `-u` 플래그로 푸시.

## 아키텍처 패턴

**API 응답 형식:** 성공 지표, 데이터 페이로드, 오류 메시지, 페이지네이션 메타데이터를 포함한 일관된 envelope.

**Repository 패턴:** 표준 인터페이스(findAll, findById, create, update, delete) 뒤에 데이터 접근 캡슐화. 비즈니스 로직은 저장 메커니즘이 아닌 추상 인터페이스에 의존.

**스켈레톤 프로젝트:** 검증된 템플릿 검색, 병렬 agent로 평가(보안, 확장성, 관련성), 최적 템플릿 클론, 검증된 구조 내에서 반복.

## 성능

**컨텍스트 관리:** 대규모 리팩토링 및 멀티 파일 기능에서 컨텍스트 윈도우 마지막 20% 사용 회피. 낮은 민감도 작업(단일 편집, 문서, 간단한 수정)은 높은 활용도 허용.

**빌드 문제 해결:** build-error-resolver agent 사용 → 오류 분석 → 점진적 수정 → 각 수정 후 검증.

## 프로젝트 구조

```
agents/          — 13 specialized subagents
skills/          — 65+ workflow skills and domain knowledge
commands/        — 40 slash commands
hooks/           — Trigger-based automations
rules/           — Always-follow guidelines (common + per-language)
scripts/         — Cross-platform Node.js utilities
mcp-configs/     — 14 MCP server configurations
tests/           — Test suite
```

## 성공 지표

- 모든 테스트 80%+ 커버리지로 통과
- 보안 취약점 없음
- 코드 가독성 및 유지보수성 확보
- 허용 가능한 성능
- 사용자 요구사항 충족

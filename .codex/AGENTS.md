# Codex CLI용 ECC

이 문서는 루트 `AGENTS.md`를 Codex 특화 가이드로 보완합니다.

## 모델 권장사항

| 작업 유형 | 권장 모델 |
|-----------|------------------|
| 일상적인 코딩, 테스트, 포맷팅 | GPT 5.4 |
| 복잡한 기능, 아키텍처 | GPT 5.4 |
| 디버깅, 리팩토링 | GPT 5.4 |
| 보안 리뷰 | GPT 5.4 |

## Skill 탐색

Skill은 `.agents/skills/`에서 자동으로 로드됩니다. 각 skill에는 다음이 포함됩니다:
- `SKILL.md` — 상세 지침 및 워크플로우
- `agents/openai.yaml` — Codex 인터페이스 메타데이터

사용 가능한 skill:
- tdd-workflow — 80%+ 커버리지의 테스트 주도 개발
- security-review — 포괄적인 보안 체크리스트
- coding-standards — 범용 코딩 표준
- frontend-patterns — React/Next.js 패턴
- frontend-slides — 뷰포트 안전 HTML 프레젠테이션 및 PPTX-to-web 변환
- article-writing — 노트 및 목소리 참고자료로 장문 작성
- content-engine — 플랫폼 네이티브 소셜 콘텐츠 및 재활용
- market-research — 출처 표기 시장 및 경쟁사 리서치
- investor-materials — 덱, 메모, 모델, 원페이저
- investor-outreach — 개인화된 투자자 아웃리치 및 팔로업
- backend-patterns — API 설계, 데이터베이스, 캐싱
- e2e-testing — Playwright E2E 테스트
- eval-harness — 평가 주도 개발
- strategic-compact — 컨텍스트 관리
- api-design — REST API 설계 패턴
- verification-loop — 빌드, 테스트, 린트, 타입체크, 보안
- deep-research — firecrawl 및 exa MCP를 활용한 멀티소스 리서치
- exa-search — 웹, 코드, 회사를 위한 Exa MCP의 신경망 검색
- claude-api — Anthropic Claude API 패턴 및 SDK
- x-api — 게시, 스레드, 분석을 위한 X/Twitter API 통합
- crosspost — 멀티플랫폼 콘텐츠 배포
- fal-ai-media — fal.ai를 통한 AI 이미지/영상/오디오 생성
- dmux-workflows — dmux를 이용한 멀티 agent 오케스트레이션

## MCP 서버

ECC의 기본 Codex 베이스라인으로 프로젝트 로컬 `.codex/config.toml`을 사용하세요. 현재 ECC 베이스라인은 GitHub, Context7, Exa, Memory, Playwright, Sequential Thinking을 활성화합니다; 작업에 실제로 필요한 경우에만 `~/.codex/config.toml`에 더 무거운 추가 항목을 넣으세요.

## 멀티 Agent 지원

Codex는 이제 실험적인 `features.multi_agent` 플래그 뒤에서 멀티 agent 워크플로우를 지원합니다.

- `.codex/config.toml`에서 `[features] multi_agent = true`로 활성화
- `[agents.<name>]` 아래 프로젝트 로컬 역할 정의
- 각 역할을 `.codex/agents/` 아래 TOML 레이어로 지정
- Codex CLI 내 `/agent`를 사용하여 자식 agent 검사 및 조종

이 저장소의 샘플 역할 설정:
- `.codex/agents/explorer.toml` — 읽기 전용 증거 수집
- `.codex/agents/reviewer.toml` — 정확성/보안 리뷰
- `.codex/agents/docs-researcher.toml` — API 및 릴리즈 노트 검증

## Claude Code와의 주요 차이점

| 기능 | Claude Code | Codex CLI |
|---------|------------|-----------|
| Hook | 8개 이상의 이벤트 유형 | 아직 미지원 |
| 컨텍스트 파일 | CLAUDE.md + AGENTS.md | AGENTS.md만 |
| Skill | 플러그인을 통해 로드 | `.agents/skills/` 디렉토리 |
| Command | `/slash` command | 지시 기반 |
| Agent | 서브agent Task 도구 | `/agent` 및 `[agents.<name>]` 역할을 통한 멀티 agent |
| 보안 | Hook 기반 강제 | 지시 + 샌드박스 |
| MCP | 완전 지원 | `config.toml` 및 `codex mcp add`를 통해 지원 |

## Hook 없는 보안

Codex에는 hook이 없으므로 보안 강제는 지시 기반입니다:
1. 항상 시스템 경계에서 입력 유효성 검사
2. 시크릿을 절대 하드코딩하지 않기 — 환경 변수 사용
3. 커밋 전 `npm audit` / `pip audit` 실행
4. 모든 push 전 `git diff` 검토
5. 설정에서 `sandbox_mode = "workspace-write"` 사용

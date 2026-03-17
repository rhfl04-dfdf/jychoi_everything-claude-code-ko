# OpenCode ECC 플러그인

> ⚠️ 이 README는 OpenCode 사용에 특화되어 있습니다.
> npm을 통해 ECC를 설치한 경우(예: `npm install opencode-ecc`), 루트 README를 참조하세요.

Everything Claude Code (ECC)의 OpenCode용 플러그인 - agent, 명령어, hook, skills를 포함합니다.

## 설치

## 설치 개요

Everything Claude Code (ECC)를 사용하는 방법은 두 가지입니다:

1. **npm 패키지 (대부분의 사용자에게 권장)**
   npm/bun/yarn으로 설치하고 `ecc-install` CLI를 사용하여 규칙과 agent를 설정합니다.

2. **직접 클론 / 플러그인 모드**
   저장소를 클론하고 내부에서 OpenCode를 직접 실행합니다.

아래에서 워크플로우에 맞는 방법을 선택하세요.

### 옵션 1: npm 패키지

```bash
npm install ecc-universal
```

`opencode.json`에 추가:

```json
{
  "plugin": ["ecc-universal"]
}
```

이는 npm에서 ECC OpenCode 플러그인 모듈을 로드합니다:
- hook/이벤트 통합
- 플러그인이 내보내는 번들 커스텀 도구

ECC의 전체 명령어/agent/instructions 카탈로그를 프로젝트 설정에 자동으로 등록하지는 **않습니다**. 전체 OpenCode 설정을 원한다면:
- 이 저장소 안에서 OpenCode를 실행하거나,
- 관련 `.opencode/commands/`, `.opencode/prompts/`, `.opencode/instructions/`, 그리고 `instructions`, `agent`, `command` 설정 항목을 본인 프로젝트에 복사하세요

설치 후 `ecc-install` CLI도 사용 가능합니다:

```bash
npx ecc-install typescript
```

### 옵션 2: 직접 사용

저장소를 클론하고 OpenCode를 실행:

```bash
git clone https://github.com/affaan-m/everything-claude-code
cd everything-claude-code
opencode
```

## 기능

### Agent (12개)

| Agent | 설명 |
|-------|-------------|
| planner | 구현 계획 수립 |
| architect | 시스템 설계 |
| code-reviewer | 코드 검토 |
| security-reviewer | 보안 분석 |
| tdd-guide | 테스트 주도 개발 |
| build-error-resolver | 빌드 오류 수정 |
| e2e-runner | E2E 테스트 |
| doc-updater | 문서화 |
| refactor-cleaner | 데드 코드 정리 |
| go-reviewer | Go 코드 검토 |
| go-build-resolver | Go 빌드 오류 |
| database-reviewer | 데이터베이스 최적화 |

### 명령어 (31개)

| 명령어 | 설명 |
|---------|-------------|
| `/plan` | 구현 계획 생성 |
| `/tdd` | TDD 워크플로우 |
| `/code-review` | 코드 변경 사항 검토 |
| `/security` | 보안 검토 |
| `/build-fix` | 빌드 오류 수정 |
| `/e2e` | E2E 테스트 |
| `/refactor-clean` | 데드 코드 제거 |
| `/orchestrate` | 멀티 agent 워크플로우 |
| `/learn` | 패턴 추출 |
| `/checkpoint` | 진행 상황 저장 |
| `/verify` | 검증 루프 |
| `/eval` | 평가 |
| `/update-docs` | 문서 업데이트 |
| `/update-codemaps` | 코드맵 업데이트 |
| `/test-coverage` | 커버리지 분석 |
| `/setup-pm` | 패키지 매니저 |
| `/go-review` | Go 코드 검토 |
| `/go-test` | Go TDD |
| `/go-build` | Go 빌드 수정 |
| `/skill-create` | Skills 생성 |
| `/instinct-status` | Instinct 보기 |
| `/instinct-import` | Instinct 가져오기 |
| `/instinct-export` | Instinct 내보내기 |
| `/evolve` | Instinct 클러스터링 |
| `/promote` | 프로젝트 instinct 승격 |
| `/projects` | 알려진 프로젝트 목록 |
| `/harness-audit` | 하네스 신뢰도 및 eval 준비 상태 감사 |
| `/loop-start` | 제어된 agentic 루프 시작 |
| `/loop-status` | 루프 상태 및 체크포인트 확인 |
| `/quality-gate` | 파일/저장소 범위에서 품질 게이트 실행 |
| `/model-route` | 모델 및 예산 기준으로 작업 라우팅 |

### 플러그인 Hook

| Hook | 이벤트 | 목적 |
|------|-------|---------|
| Prettier | `file.edited` | JS/TS 자동 포맷 |
| TypeScript | `tool.execute.after` | 타입 오류 검사 |
| console.log | `file.edited` | 디버그 문 경고 |
| 알림 | `session.idle` | 데스크톱 알림 |
| 보안 | `tool.execute.before` | 시크릿 확인 |

### 커스텀 도구

| 도구 | 설명 |
|------|-------------|
| run-tests | 옵션과 함께 테스트 스위트 실행 |
| check-coverage | 테스트 커버리지 분석 |
| security-audit | 보안 취약점 스캔 |

## Hook 이벤트 매핑

OpenCode의 플러그인 시스템은 Claude Code hook에 매핑됩니다:

| Claude Code | OpenCode |
|-------------|----------|
| PreToolUse | `tool.execute.before` |
| PostToolUse | `tool.execute.after` |
| Stop | `session.idle` |
| SessionStart | `session.created` |
| SessionEnd | `session.deleted` |

OpenCode에는 Claude Code에서 사용할 수 없는 20개 이상의 추가 이벤트가 있습니다.

### Hook 런타임 제어

OpenCode 플러그인 hook은 Claude Code/Cursor에서 사용하는 동일한 런타임 제어를 지원합니다:

```bash
export ECC_HOOK_PROFILE=standard
export ECC_DISABLED_HOOKS="pre:bash:tmux-reminder,post:edit:typecheck"
```

- `ECC_HOOK_PROFILE`: `minimal`, `standard` (기본값), `strict`
- `ECC_DISABLED_HOOKS`: 비활성화할 hook ID의 쉼표 구분 목록

## Skills

기본 OpenCode 설정은 `instructions` 배열을 통해 11개의 선별된 ECC skills를 로드합니다:

- coding-standards
- backend-patterns
- frontend-patterns
- frontend-slides
- security-review
- tdd-workflow
- strategic-compact
- eval-harness
- verification-loop
- api-design
- e2e-testing

OpenCode 세션을 간결하게 유지하기 위해 기본적으로 로드되지 않는 추가 전문 skills가 `skills/`에 포함되어 있습니다:

- article-writing
- content-engine
- market-research
- investor-materials
- investor-outreach

## 설정

`opencode.json`의 전체 설정:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "anthropic/claude-sonnet-4-5",
  "small_model": "anthropic/claude-haiku-4-5",
  "plugin": ["./plugins"],
  "instructions": [
    "skills/tdd-workflow/SKILL.md",
    "skills/security-review/SKILL.md"
  ],
  "agent": { /* 12 agents */ },
  "command": { /* 24 commands */ }
}
```

## 라이선스

MIT

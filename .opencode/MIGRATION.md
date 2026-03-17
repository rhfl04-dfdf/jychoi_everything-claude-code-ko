# 마이그레이션 가이드: Claude Code에서 OpenCode로

이 가이드는 Everything Claude Code (ECC) 설정을 사용하면서 Claude Code에서 OpenCode로 마이그레이션하는 데 도움을 드립니다.

## 개요

OpenCode는 AI 보조 개발을 위한 대체 CLI로, Claude Code와 동일한 **모든** 기능을 지원하며 설정 형식에서 일부 차이가 있습니다.

## 주요 차이점

| 기능 | Claude Code | OpenCode | 비고 |
|---------|-------------|----------|-------|
| 설정 | `CLAUDE.md`, `plugin.json` | `opencode.json` | 파일 형식 다름 |
| Agent | Markdown frontmatter | JSON 객체 | 완전 동등 |
| 명령어 | `commands/*.md` | `command` 객체 또는 `.md` 파일 | 완전 동등 |
| Skills | `skills/*/SKILL.md` | `instructions` 배열 | 컨텍스트로 로드 |
| **Hooks** | `hooks.json` (3단계) | **플러그인 시스템 (20개 이상 이벤트)** | **완전 동등 + 더 많음!** |
| 규칙 | `rules/*.md` | `instructions` 배열 | 통합 또는 분리 |
| MCP | 전체 지원 | 전체 지원 | 완전 동등 |

## Hook 마이그레이션

**OpenCode는 hook을 완전히 지원합니다.** 플러그인 시스템을 통해 20개 이상의 이벤트 타입을 제공하며, 실제로 Claude Code보다 더 정교합니다.

### Hook 이벤트 매핑

| Claude Code Hook | OpenCode 플러그인 이벤트 | 비고 |
|-----------------|----------------------|-------|
| `PreToolUse` | `tool.execute.before` | 도구 입력 수정 가능 |
| `PostToolUse` | `tool.execute.after` | 도구 출력 수정 가능 |
| `Stop` | `session.idle` 또는 `session.status` | 세션 생명주기 |
| `SessionStart` | `session.created` | 세션 시작 |
| `SessionEnd` | `session.deleted` | 세션 종료 |
| N/A | `file.edited` | OpenCode 전용: 파일 변경 |
| N/A | `file.watcher.updated` | OpenCode 전용: 파일 시스템 감시 |
| N/A | `message.updated` | OpenCode 전용: 메시지 변경 |
| N/A | `lsp.client.diagnostics` | OpenCode 전용: LSP 통합 |
| N/A | `tui.toast.show` | OpenCode 전용: 알림 |

### Hook을 플러그인으로 변환하기

**Claude Code hook (hooks.json):**
```json
{
  "PostToolUse": [{
    "matcher": "tool == \"Edit\" && tool_input.file_path matches \"\\\\.(ts|tsx|js|jsx)$\"",
    "hooks": [{
      "type": "command",
      "command": "prettier --write \"$file_path\""
    }]
  }]
}
```

**OpenCode 플러그인 (.opencode/plugins/prettier-hook.ts):**
```typescript
export const PrettierPlugin = async ({ $ }) => {
  return {
    "file.edited": async (event) => {
      if (event.path.match(/\.(ts|tsx|js|jsx)$/)) {
        await $`prettier --write ${event.path}`
      }
    }
  }
}
```

### 포함된 ECC 플러그인 Hook

ECC OpenCode 설정에는 변환된 hook이 포함되어 있습니다:

| Hook | OpenCode 이벤트 | 목적 |
|------|----------------|---------|
| Prettier 자동 포맷 | `file.edited` | 편집 후 JS/TS 파일 포맷 |
| TypeScript 검사 | `tool.execute.after` | .ts 파일 편집 후 tsc 실행 |
| console.log 경고 | `file.edited` | console.log 문 경고 |
| 세션 알림 | `session.idle` | 작업 완료 시 알림 |
| 보안 검사 | `tool.execute.before` | 커밋 전 시크릿 확인 |

## 마이그레이션 단계

### 1. OpenCode 설치

```bash
# Install OpenCode CLI
npm install -g opencode
# or
curl -fsSL https://opencode.ai/install | bash
```

### 2. ECC OpenCode 설정 사용

이 저장소의 `.opencode/` 디렉토리에는 변환된 설정이 포함되어 있습니다:

```
.opencode/
├── opencode.json              # Main configuration
├── plugins/                   # Hook plugins (translated from hooks.json)
│   ├── ecc-hooks.ts           # All ECC hooks as plugins
│   └── index.ts               # Plugin exports
├── tools/                     # Custom tools
│   ├── run-tests.ts           # Run test suite
│   ├── check-coverage.ts      # Check coverage
│   └── security-audit.ts      # npm audit wrapper
├── commands/                  # All 23 commands (markdown)
│   ├── plan.md
│   ├── tdd.md
│   └── ... (21 more)
├── prompts/
│   └── agents/                # Agent prompt files (12)
├── instructions/
│   └── INSTRUCTIONS.md        # Consolidated rules
├── package.json               # For npm distribution
├── tsconfig.json              # TypeScript config
└── MIGRATION.md               # This file
```

### 3. OpenCode 실행

```bash
# In the repository root
opencode

# The configuration is automatically detected from .opencode/opencode.json
```

## 개념 매핑

### Agent

**Claude Code:**
```markdown
---
name: planner
description: Expert planning specialist...
tools: ["Read", "Grep", "Glob"]
model: opus
---

You are an expert planning specialist...
```

**OpenCode:**
```json
{
  "agent": {
    "planner": {
      "description": "Expert planning specialist...",
      "mode": "subagent",
      "model": "anthropic/claude-opus-4-5",
      "prompt": "{file:prompts/agents/planner.txt}",
      "tools": { "read": true, "bash": true }
    }
  }
}
```

### 명령어

**Claude Code:**
```markdown
---
name: plan
description: Create implementation plan
---

Create a detailed implementation plan for: {input}
```

**OpenCode (JSON):**
```json
{
  "command": {
    "plan": {
      "description": "Create implementation plan",
      "template": "Create a detailed implementation plan for: $ARGUMENTS",
      "agent": "planner"
    }
  }
}
```

**OpenCode (Markdown - .opencode/commands/plan.md):**
```markdown
---
description: Create implementation plan
agent: planner
---

Create a detailed implementation plan for: $ARGUMENTS
```

### Skills

**Claude Code:** Skills은 `skills/*/SKILL.md` 파일에서 로드됩니다.

**OpenCode:** Skills은 `instructions` 배열에 추가됩니다:
```json
{
  "instructions": [
    "skills/tdd-workflow/SKILL.md",
    "skills/security-review/SKILL.md",
    "skills/coding-standards/SKILL.md"
  ]
}
```

### 규칙

**Claude Code:** 규칙은 별도의 `rules/*.md` 파일에 있습니다.

**OpenCode:** 규칙은 `instructions`로 통합하거나 별도로 유지할 수 있습니다:
```json
{
  "instructions": [
    "instructions/INSTRUCTIONS.md",
    "rules/common/security.md",
    "rules/common/coding-style.md"
  ]
}
```

## 모델 매핑

| Claude Code | OpenCode |
|-------------|----------|
| `opus` | `anthropic/claude-opus-4-5` |
| `sonnet` | `anthropic/claude-sonnet-4-5` |
| `haiku` | `anthropic/claude-haiku-4-5` |

## 사용 가능한 명령어

마이그레이션 후 전체 23개 명령어를 사용할 수 있습니다:

| 명령어 | 설명 |
|---------|-------------|
| `/plan` | 구현 계획 생성 |
| `/tdd` | TDD 워크플로우 적용 |
| `/code-review` | 코드 변경 사항 검토 |
| `/security` | 보안 검토 실행 |
| `/build-fix` | 빌드 오류 수정 |
| `/e2e` | E2E 테스트 생성 |
| `/refactor-clean` | 데드 코드 제거 |
| `/orchestrate` | 멀티 agent 워크플로우 |
| `/learn` | 세션 중 패턴 추출 |
| `/checkpoint` | 검증 상태 저장 |
| `/verify` | 검증 루프 실행 |
| `/eval` | 평가 실행 |
| `/update-docs` | 문서 업데이트 |
| `/update-codemaps` | 코드맵 업데이트 |
| `/test-coverage` | 테스트 커버리지 확인 |
| `/setup-pm` | 패키지 매니저 설정 |
| `/go-review` | Go 코드 검토 |
| `/go-test` | Go TDD 워크플로우 |
| `/go-build` | Go 빌드 오류 수정 |
| `/skill-create` | git 히스토리에서 skills 생성 |
| `/instinct-status` | 학습된 instinct 보기 |
| `/instinct-import` | instinct 가져오기 |
| `/instinct-export` | instinct 내보내기 |
| `/evolve` | instinct를 skills로 클러스터링 |
| `/promote` | 프로젝트 instinct를 전역 범위로 승격 |
| `/projects` | 알려진 프로젝트 및 instinct 통계 보기 |

## 사용 가능한 Agent

| Agent | 설명 |
|-------|-------------|
| `planner` | 구현 계획 수립 |
| `architect` | 시스템 설계 |
| `code-reviewer` | 코드 검토 |
| `security-reviewer` | 보안 분석 |
| `tdd-guide` | 테스트 주도 개발 |
| `build-error-resolver` | 빌드 오류 수정 |
| `e2e-runner` | E2E 테스트 |
| `doc-updater` | 문서화 |
| `refactor-cleaner` | 데드 코드 정리 |
| `go-reviewer` | Go 코드 검토 |
| `go-build-resolver` | Go 빌드 오류 |
| `database-reviewer` | 데이터베이스 최적화 |

## 플러그인 설치

### 옵션 1: ECC 설정 직접 사용

`.opencode/` 디렉토리에 모든 것이 사전 설정되어 있습니다.

### 옵션 2: npm 패키지로 설치

```bash
npm install ecc-universal
```

그런 다음 `opencode.json`에 추가:
```json
{
  "plugin": ["ecc-universal"]
}
```

이는 게시된 ECC OpenCode 플러그인 모듈(hook/이벤트 및 내보낸 플러그인 도구)만 로드합니다.
ECC의 전체 `agent`, `command`, 또는 `instructions` 설정을 프로젝트에 자동으로 주입하지는 **않습니다**.

전체 ECC OpenCode 워크플로우 환경을 원한다면 저장소에 번들된 `.opencode/opencode.json`을 기본 설정으로 사용하거나 다음 항목을 프로젝트에 복사하세요:
- `.opencode/commands/`
- `.opencode/prompts/`
- `.opencode/instructions/INSTRUCTIONS.md`
- `.opencode/opencode.json`의 `agent` 및 `command` 섹션

## 문제 해결

### 설정이 로드되지 않는 경우

1. `.opencode/opencode.json`이 저장소 루트에 있는지 확인
2. JSON 구문이 유효한지 확인: `cat .opencode/opencode.json | jq .`
3. 참조된 모든 프롬프트 파일이 존재하는지 확인

### 플러그인이 로드되지 않는 경우

1. 플러그인 파일이 `.opencode/plugins/`에 있는지 확인
2. TypeScript 구문이 유효한지 확인
3. `opencode.json`의 `plugin` 배열에 해당 경로가 포함되어 있는지 확인

### Agent를 찾을 수 없는 경우

1. `opencode.json`의 `agent` 객체에 agent가 정의되어 있는지 확인
2. 프롬프트 파일 경로가 올바른지 확인
3. 지정된 경로에 프롬프트 파일이 있는지 확인

### 명령어가 작동하지 않는 경우

1. `opencode.json`에 명령어가 정의되어 있거나 `.opencode/commands/`에 `.md` 파일로 있는지 확인
2. 참조된 agent가 존재하는지 확인
3. 템플릿이 사용자 입력에 `$ARGUMENTS`를 사용하는지 확인
4. `plugin: ["ecc-universal"]`만 설치한 경우, npm 플러그인 설치는 ECC 명령어나 agent를 프로젝트 설정에 자동으로 추가하지 않습니다

## 모범 사례

1. **새로 시작하기**: Claude Code와 OpenCode를 동시에 실행하려 하지 마세요
2. **설정 확인**: `opencode.json`이 오류 없이 로드되는지 확인
3. **명령어 테스트**: 각 명령어를 한 번씩 실행하여 작동 여부 확인
4. **플러그인 활용**: 자동화를 위해 플러그인 hook 활용
5. **Agent 활용**: 특화된 목적에 맞게 전문 agent 활용

## Claude Code로 되돌리기

되돌려야 하는 경우:

1. `opencode` 대신 `claude`를 실행하면 됩니다
2. Claude Code는 자체 설정(`CLAUDE.md`, `plugin.json` 등)을 사용합니다
3. `.opencode/` 디렉토리는 Claude Code에 영향을 주지 않습니다

## 기능 동등성 요약

| 기능 | Claude Code | OpenCode | 상태 |
|---------|-------------|----------|--------|
| Agent | ✅ 12개 agent | ✅ 12개 agent | **완전 동등** |
| 명령어 | ✅ 23개 명령어 | ✅ 23개 명령어 | **완전 동등** |
| Skills | ✅ 16개 skills | ✅ 16개 skills | **완전 동등** |
| Hook | ✅ 3단계 | ✅ 20개 이상 이벤트 | **OpenCode가 더 많음** |
| 규칙 | ✅ 8개 규칙 | ✅ 8개 규칙 | **완전 동등** |
| MCP 서버 | ✅ 전체 | ✅ 전체 | **완전 동등** |
| 커스텀 도구 | ✅ Hook을 통해 | ✅ 기본 지원 | **OpenCode가 더 우수** |

## 피드백

다음 관련 이슈는 각각 신고하세요:
- **OpenCode CLI**: OpenCode 이슈 트래커에 신고
- **ECC 설정**: [github.com/affaan-m/everything-claude-code](https://github.com/affaan-m/everything-claude-code)에 신고

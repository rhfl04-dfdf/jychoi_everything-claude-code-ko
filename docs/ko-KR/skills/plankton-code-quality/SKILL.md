---
name: plankton-code-quality
description: "Plankton을 이용한 쓰기 시점(Write-time) code quality 강제 적용 — hook을 통해 모든 파일 수정 시 자동 포맷팅, linting 및 Claude 기반 수정 기능을 제공합니다."
origin: community
---

# Plankton Code Quality Skill

Claude Code를 위한 쓰기 시점(Write-time) code quality 강제 적용 시스템인 Plankton(@alxfazio 제작)의 통합 참조 가이드입니다. Plankton은 PostToolUse hook을 통해 모든 파일 수정 시 formatter와 linter를 실행하며, 이후 Claude subprocess를 생성하여 agent가 포착하지 못한 위반 사항을 수정합니다.

## 사용 시기

- 모든 파일 수정 시(commit 시점뿐만 아니라) 자동 formatting 및 linting을 원하는 경우
- 코드를 수정하는 대신 검사를 통과하기 위해 linter config를 수정하는 agent로부터 보호가 필요한 경우
- 수정 사항의 복잡도에 따른 단계별 model routing이 필요한 경우 (간단한 스타일은 Haiku, 로직은 Sonnet, 타입은 Opus)
- 다양한 언어(Python, TypeScript, Shell, YAML, JSON, TOML, Markdown, Dockerfile)를 사용하는 경우

## 작동 방식

### 3단계 아키텍처

Claude Code가 파일을 수정하거나 작성할 때마다 Plankton의 `multi_linter.sh` PostToolUse hook이 실행됩니다:

```
Phase 1: Auto-Format (Silent)
├─ Runs formatters (ruff format, biome, shfmt, taplo, markdownlint)
├─ Fixes 40-50% of issues silently
└─ No output to main agent

Phase 2: Collect Violations (JSON)
├─ Runs linters and collects unfixable violations
├─ Returns structured JSON: {line, column, code, message, linter}
└─ Still no output to main agent

Phase 3: Delegate + Verify
├─ Spawns claude -p subprocess with violations JSON
├─ Routes to model tier based on violation complexity:
│   ├─ Haiku: formatting, imports, style (E/W/F codes) — 120s timeout
│   ├─ Sonnet: complexity, refactoring (C901, PLR codes) — 300s timeout
│   └─ Opus: type system, deep reasoning (unresolved-attribute) — 600s timeout
├─ Re-runs Phase 1+2 to verify fixes
└─ Exit 0 if clean, Exit 2 if violations remain (reported to main agent)
```

### 기본 Agent가 보게 되는 내용

| 시나리오 | Agent에게 보이는 내용 | Hook 종료 코드 |
|----------|-----------|-----------|
| 위반 사항 없음 | 없음 | 0 |
| subprocess에 의해 모두 수정됨 | 없음 | 0 |
| subprocess 이후에도 위반 사항이 남은 경우 | `[hook] N violation(s) remain` | 2 |
| Advisory (중복, 오래된 툴링) | `[hook:advisory] ...` | 0 |

기본 agent는 subprocess가 수정하지 못한 문제만 확인하게 됩니다. 대부분의 quality 문제는 투명하게 해결됩니다.

### Config 보호 (규칙 우회 방지)

LLM은 코드를 수정하기보다 규칙을 비활성화하기 위해 `.ruff.toml` 또는 `biome.json`을 수정하곤 합니다. Plankton은 이를 세 가지 레이어로 차단합니다:

1. **PreToolUse hook** — `protect_linter_configs.sh`가 모든 linter config 수정 시도를 사전에 차단
2. **Stop hook** — `stop_config_guardian.sh`가 세션 종료 시 `git diff`를 통해 config 변경을 감지
3. **보호된 파일 목록** — `.ruff.toml`, `biome.json`, `.shellcheckrc`, `.yamllint`, `.hadolint.yaml` 등

### Package Manager 강제 적용

Bash의 PreToolUse hook이 기존 package manager 사용을 차단합니다:
- `pip`, `pip3`, `poetry`, `pipenv` → 차단됨 (`uv` 사용 권장)
- `npm`, `yarn`, `pnpm` → 차단됨 (`bun` 사용 권장)
- 허용되는 예외: `npm audit`, `npm view`, `npm publish`

## 설정 (Setup)

### 빠른 시작 (Quick Start)

```bash
# Clone Plankton into your project (or a shared location)
# Note: Plankton is by @alxfazio
git clone https://github.com/alexfazio/plankton.git
cd plankton

# Install core dependencies
brew install jaq ruff uv

# Install Python linters
uv sync --all-extras

# Start Claude Code — hooks activate automatically
claude
```

별도의 install command나 plugin config가 필요하지 않습니다. Plankton 디렉토리에서 Claude Code를 실행하면 `.claude/settings.json`의 hook이 자동으로 활성화됩니다.

### 프로젝트별 통합

개별 프로젝트에서 Plankton hook을 사용하려면:

1. `.claude/hooks/` 디렉토리를 프로젝트로 복사
2. `.claude/settings.json`의 hook 설정을 복사
3. linter config 파일들(`.ruff.toml`, `biome.json` 등)을 복사
4. 해당 언어에 맞는 linter를 설치

### 언어별 의존성

| 언어 | 필수 (Required) | 선택 (Optional) |
|----------|----------|----------|
| Python | `ruff`, `uv` | `ty` (types), `vulture` (dead code), `bandit` (security) |
| TypeScript/JS | `biome` | `oxlint`, `semgrep`, `knip` (dead exports) |
| Shell | `shellcheck`, `shfmt` | — |
| YAML | `yamllint` | — |
| Markdown | `markdownlint-cli2` | — |
| Dockerfile | `hadolint` (>= 2.12.0) | — |
| TOML | `taplo` | — |
| JSON | `jaq` | — |

## ECC와 함께 사용하기

### 중복이 아닌 상호 보완적 관계

| 관심 영역 | ECC | Plankton |
|---------|-----|----------|
| Code quality 강제 적용 | PostToolUse hooks (Prettier, tsc) | PostToolUse hooks (20개 이상의 linter + subprocess 수정) |
| Security scanning | AgentShield, security-reviewer agent | Bandit (Python), Semgrep (TypeScript) |
| Config 보호 | — | PreToolUse 차단 + Stop hook 감지 |
| Package manager | 감지 + 설정 | 강제 적용 (기존 PM 차단) |
| CI 통합 | — | git을 위한 Pre-commit hooks |
| Model routing | 수동 (`/model opus`) | 자동 (위반 복잡도에 따른 단계별 적용) |

### 권장 조합

1. plugin으로 ECC를 설치 (agent, skill, command, rule 활용)
2. 쓰기 시점(Write-time) quality 강제 적용을 위해 Plankton hook 추가
3. security 감사 시 AgentShield 사용
4. PR 전 최종 관문으로 ECC의 verification-loop 사용

### Hook 충돌 방지

ECC와 Plankton hook을 동시에 실행하는 경우:
- JS/TS 파일에서 ECC의 Prettier hook과 Plankton의 biome formatter가 충돌할 수 있습니다.
- 해결 방법: Plankton 사용 시 ECC의 Prettier PostToolUse hook을 비활성화하세요. (Plankton의 biome이 더 포괄적입니다.)
- 서로 다른 파일 타입에 대해서는 공존이 가능합니다. (ECC는 Plankton이 다루지 않는 영역을 처리합니다.)

## Configuration 참조

Plankton의 `.claude/hooks/config.json`에서 모든 동작을 제어합니다:

```json
{
  "languages": {
    "python": true,
    "shell": true,
    "yaml": true,
    "json": true,
    "toml": true,
    "dockerfile": true,
    "markdown": true,
    "typescript": {
      "enabled": true,
      "js_runtime": "auto",
      "biome_nursery": "warn",
      "semgrep": true
    }
  },
  "phases": {
    "auto_format": true,
    "subprocess_delegation": true
  },
  "subprocess": {
    "tiers": {
      "haiku":  { "timeout": 120, "max_turns": 10 },
      "sonnet": { "timeout": 300, "max_turns": 10 },
      "opus":   { "timeout": 600, "max_turns": 15 }
    },
    "volume_threshold": 5
  }
}
```

**주요 설정:**
- hook 속도 향상을 위해 사용하지 않는 언어는 비활성화하세요.
- `volume_threshold` — 위반 사항이 이 횟수를 초과하면 자동으로 더 높은 model 단계로 에스컬레이션됩니다.
- `subprocess_delegation: false` — Phase 3를 완전히 건너뜁니다 (위반 사항만 보고).

## 환경 변수 재정의 (Environment Overrides)

| 변수 | 목적 |
|----------|---------|
| `HOOK_SKIP_SUBPROCESS=1` | Phase 3를 건너뛰고 위반 사항을 직접 보고 |
| `HOOK_SUBPROCESS_TIMEOUT=N` | 단계별 timeout 재정의 |
| `HOOK_DEBUG_MODEL=1` | model 선택 결정 사항을 로깅 |
| `HOOK_SKIP_PM=1` | package manager 강제 적용 우회 |

## 참조

- Plankton (제작: @alxfazio)
- Plankton REFERENCE.md — 전체 아키텍처 문서 (제작: @alxfazio)
- Plankton SETUP.md — 상세 설치 가이드 (제작: @alxfazio)

## ECC v1.8 추가 사항

### 복사 가능한 Hook Profile

엄격한 quality 동작 설정:

```bash
export ECC_HOOK_PROFILE=strict
export ECC_QUALITY_GATE_FIX=true
export ECC_QUALITY_GATE_STRICT=true
```

### 언어별 Gate 테이블

- TypeScript/JavaScript: Biome 권장, Prettier 대체 사용 가능
- Python: Ruff format/check
- Go: gofmt

### Config 변조 방지 (Config Tamper Guard)

quality 강제 적용 중 동일한 반복 주기 내의 config 파일 변경을 감시합니다:

- `biome.json`, `.eslintrc*`, `prettier.config*`, `tsconfig.json`, `pyproject.toml`

위반 사항을 억제하기 위해 config가 변경된 경우, merge 전에 명시적인 review가 필요합니다.

### CI 통합 패턴

로컬 hook과 동일한 명령어를 CI에서 사용하세요:

1. formatter 체크 실행
2. lint/type 체크 실행
3. strict mode에서 빠른 실패(fail fast) 적용
4. 수정 요약사항 발행

### Health Metrics

다음을 추적합니다:
- gate에 의해 플래그된 수정 사항
- 평균 수정 시간
- 카테고리별 반복 위반 사항
- gate 통과 실패로 인한 merge 차단 횟수

---
name: security-scan
description: AgentShield를 사용하여 Claude Code 설정(.claude/ 디렉토리)의 보안 취약점, 설정 오류 및 injection risk를 스캔합니다. CLAUDE.md, settings.json, MCP server, hook 및 agent 정의를 확인합니다.
origin: ECC
---

# Security Scan Skill

[AgentShield](https://github.com/affaan-m/agentshield)를 사용하여 Claude Code 설정의 보안 문제를 감사하십시오.

## 활성화 시점

- 새로운 Claude Code 프로젝트 설정 시
- `.claude/settings.json`, `CLAUDE.md` 또는 MCP 설정을 수정한 후
- 설정 변경 사항을 commit하기 전
- 기존 Claude Code 설정이 있는 새로운 repository에 onboarding할 때
- 정기적인 보안 위생 점검 시

## 스캔 대상

| 파일 | 확인 사항 |
|------|--------|
| `CLAUDE.md` | Hardcoded secret, auto-run 명령, prompt injection 패턴 |
| `settings.json` | 과도하게 허용된 allow list, 누락된 deny list, 위험한 bypass 플래그 |
| `mcp.json` | 위험한 MCP server, hardcoded 환경 변수 secret, npx supply chain risk |
| `hooks/` | 보간(interpolation)을 통한 command injection, 데이터 유출, silent error 억제 |
| `agents/*.md` | 제한 없는 tool 액세스, prompt injection surface, 누락된 model 사양 |

## 사전 요구 사항

AgentShield가 설치되어 있어야 합니다. 필요한 경우 확인하고 설치하십시오:

```bash
# Check if installed
npx ecc-agentshield --version

# Install globally (recommended)
npm install -g ecc-agentshield

# Or run directly via npx (no install needed)
npx ecc-agentshield scan .
```

## 사용법

### 기본 스캔

현재 프로젝트의 `.claude/` 디렉토리에 대해 실행합니다:

```bash
# Scan current project
npx ecc-agentshield scan

# Scan a specific path
npx ecc-agentshield scan --path /path/to/.claude

# Scan with minimum severity filter
npx ecc-agentshield scan --min-severity medium
```

### 출력 형식

```bash
# Terminal output (default) — colored report with grade
npx ecc-agentshield scan

# JSON — for CI/CD integration
npx ecc-agentshield scan --format json

# Markdown — for documentation
npx ecc-agentshield scan --format markdown

# HTML — self-contained dark-theme report
npx ecc-agentshield scan --format html > security-report.html
```

### Auto-Fix

안전한 수정을 자동으로 적용합니다(auto-fix 가능으로 표시된 수정 사항만 해당):

```bash
npx ecc-agentshield scan --fix
```

이 작업은 다음을 수행합니다:
- Hardcoded secret을 환경 변수 참조로 교체
- 와일드카드 권한을 범위가 지정된 대안으로 강화
- 수동 전용 제안은 수정하지 않음

### Opus 4.6 심층 분석

더 심층적인 분석을 위해 adversarial 3-agent 파이프라인을 실행합니다:

```bash
# Requires ANTHROPIC_API_KEY
export ANTHROPIC_API_KEY=your-key
npx ecc-agentshield scan --opus --stream
```

다음이 실행됩니다:
1. **Attacker (Red Team)** — 공격 벡터 탐색
2. **Defender (Blue Team)** — 강화 권장 사항 제안
3. **Auditor (Final Verdict)** — 두 관점을 종합

### 보안 설정 초기화

처음부터 새로운 보안 `.claude/` 설정을 구성합니다:

```bash
npx ecc-agentshield init
```

생성되는 파일:
- 범위가 지정된 권한과 deny list가 포함된 `settings.json`
- 보안 best practice가 포함된 `CLAUDE.md`
- `mcp.json` placeholder

### GitHub Action

CI 파이프라인에 추가하십시오:

```yaml
- uses: affaan-m/agentshield@v1
  with:
    path: '.'
    min-severity: 'medium'
    fail-on-findings: true
```

## 심각도 수준

| 등급 | 점수 | 의미 |
|-------|-------|---------|
| A | 90-100 | 보안 설정됨 |
| B | 75-89 | 사소한 문제 |
| C | 60-74 | 주의 필요 |
| D | 40-59 | 상당한 위험 |
| F | 0-39 | 치명적인 취약점 |

## 결과 해석

### 치명적인 결과 (즉시 수정)
- 설정 파일의 Hardcoded API key 또는 토큰
- allow list의 `Bash(*)` (제한 없는 shell 액세스)
- `${file}` 보간을 통한 hook의 command injection
- shell을 실행하는 MCP server

### 높은 위험 결과 (운영 적용 전 수정)
- CLAUDE.md의 auto-run 명령 (prompt injection 벡터)
- 권한 설정의 deny list 누락
- 불필요한 Bash 액세스 권한이 있는 agent

### 중간 위험 결과 (권장)
- hook의 silent error 억제 (`2>/dev/null`, `|| true`)
- PreToolUse 보안 hook 누락
- MCP server 설정의 `npx -y` 자동 설치

### 정보 결과 (인지)
- MCP server 설명 누락
- good practice로 올바르게 표시된 금지 명령

## 링크

- **GitHub**: [github.com/affaan-m/agentshield](https://github.com/affaan-m/agentshield)
- **npm**: [npmjs.com/package/ecc-agentshield](https://www.npmjs.com/package/ecc-agentshield)

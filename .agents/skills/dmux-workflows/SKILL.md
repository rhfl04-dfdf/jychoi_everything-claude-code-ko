---
name: dmux-workflows
description: dmux(AI 에이전트용 tmux 패널 관리자)를 사용한 멀티 에이전트 오케스트레이션. Claude Code, Codex, OpenCode 등 다양한 에이전트 하네스에서의 병렬 에이전트 워크플로우 패턴. 여러 에이전트 세션을 병렬로 실행하거나 멀티 에이전트 개발 워크플로우를 조율할 때 사용합니다.
origin: ECC
---

# dmux 워크플로우

AI 에이전트 하네스용 tmux 패널 관리자인 dmux를 사용하여 병렬 AI 에이전트 세션을 오케스트레이션합니다.

## 활성화 조건

- 여러 에이전트 세션을 병렬로 실행할 때
- Claude Code, Codex 및 기타 하네스 간 작업을 조율할 때
- 분할 정복 병렬 처리가 유용한 복잡한 작업을 수행할 때
- 사용자가 "병렬로 실행", "작업 분할", "dmux 사용", "멀티 에이전트" 등을 말할 때

## dmux란

dmux는 AI 에이전트 패널을 관리하는 tmux 기반 오케스트레이션 도구입니다:
- `n`을 눌러 프롬프트와 함께 새 패널 생성
- `m`을 눌러 패널 출력을 메인 세션으로 병합
- 지원 도구: Claude Code, Codex, OpenCode, Cline, Gemini, Qwen

**설치:** `npm install -g dmux` 또는 [github.com/standardagents/dmux](https://github.com/standardagents/dmux) 참조

## 빠른 시작

```bash
# Start dmux session
dmux

# Create agent panes (press 'n' in dmux, then type prompt)
# Pane 1: "Implement the auth middleware in src/auth/"
# Pane 2: "Write tests for the user service"
# Pane 3: "Update API documentation"

# Each pane runs its own agent session
# Press 'm' to merge results back
```

## 워크플로우 패턴

### 패턴 1: 리서치 + 구현

리서치와 구현을 병렬 트랙으로 분할합니다:

```
Pane 1 (Research): "Research best practices for rate limiting in Node.js.
  Check current libraries, compare approaches, and write findings to
  /tmp/rate-limit-research.md"

Pane 2 (Implement): "Implement rate limiting middleware for our Express API.
  Start with a basic token bucket, we'll refine after research completes."

# After Pane 1 completes, merge findings into Pane 2's context
```

### 패턴 2: 멀티 파일 기능

독립적인 파일들에 걸친 작업을 병렬화합니다:

```
Pane 1: "Create the database schema and migrations for the billing feature"
Pane 2: "Build the billing API endpoints in src/api/billing/"
Pane 3: "Create the billing dashboard UI components"

# Merge all, then do integration in main pane
```

### 패턴 3: 테스트 + 수정 루프

한 패널에서 테스트를 실행하고, 다른 패널에서 수정합니다:

```
Pane 1 (Watcher): "Run the test suite in watch mode. When tests fail,
  summarize the failures."

Pane 2 (Fixer): "Fix failing tests based on the error output from pane 1"
```

### 패턴 4: 크로스 하네스

서로 다른 AI 도구를 서로 다른 작업에 사용합니다:

```
Pane 1 (Claude Code): "Review the security of the auth module"
Pane 2 (Codex): "Refactor the utility functions for performance"
Pane 3 (Claude Code): "Write E2E tests for the checkout flow"
```

### 패턴 5: 코드 리뷰 파이프라인

병렬 리뷰 관점을 제공합니다:

```
Pane 1: "Review src/api/ for security vulnerabilities"
Pane 2: "Review src/api/ for performance issues"
Pane 3: "Review src/api/ for test coverage gaps"

# Merge all reviews into a single report
```

## 모범 사례

1. **독립적인 작업만 병렬화하세요.** 서로의 출력에 의존하는 작업은 병렬화하지 마세요.
2. **명확한 경계를 설정하세요.** 각 패널은 별개의 파일이나 관심사를 다루어야 합니다.
3. **전략적으로 병합하세요.** 충돌을 방지하기 위해 병합 전 패널 출력을 검토하세요.
4. **git worktree를 활용하세요.** 파일 충돌이 발생할 수 있는 작업에는 패널별로 별도의 worktree를 사용하세요.
5. **리소스를 인식하세요.** 각 패널은 API 토큰을 사용합니다 — 전체 패널 수를 5-6개 이하로 유지하세요.

## Git Worktree 통합

겹치는 파일을 다루는 작업에 사용합니다:

```bash
# Create worktrees for isolation
git worktree add ../feature-auth feat/auth
git worktree add ../feature-billing feat/billing

# Run agents in separate worktrees
# Pane 1: cd ../feature-auth && claude
# Pane 2: cd ../feature-billing && claude

# Merge branches when done
git merge feat/auth
git merge feat/billing
```

## 보완 도구

| 도구 | 설명 | 사용 시점 |
|------|-------------|-------------|
| **dmux** | 에이전트용 tmux 패널 관리 | 병렬 에이전트 세션 |
| **Superset** | 10개 이상의 병렬 에이전트를 위한 터미널 IDE | 대규모 오케스트레이션 |
| **Claude Code Task tool** | 프로세스 내 서브에이전트 생성 | 세션 내 프로그래밍 방식의 병렬 처리 |
| **Codex multi-agent** | 내장 에이전트 역할 | Codex 전용 병렬 작업 |

## 문제 해결

- **패널이 응답하지 않을 때:** 에이전트 세션이 입력을 기다리고 있는지 확인하세요. `m`을 사용하여 출력을 읽으세요.
- **병합 충돌:** git worktree를 사용하여 패널별로 파일 변경을 격리하세요.
- **높은 토큰 사용량:** 병렬 패널 수를 줄이세요. 각 패널은 완전한 에이전트 세션입니다.
- **tmux를 찾을 수 없을 때:** `brew install tmux` (macOS) 또는 `apt install tmux` (Linux)로 설치하세요.

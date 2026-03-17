---
name: dmux-workflows
description: dmux(AI agent를 위한 tmux pane 매니저)를 사용한 multi-agent orchestration입니다. Claude Code, Codex, OpenCode 및 기타 harness 전반의 병렬 agent workflow를 위한 패턴을 제공합니다. 여러 agent 세션을 병렬로 실행하거나 multi-agent 개발 workflow를 조율할 때 사용합니다.
origin: ECC
---

# dmux Workflows

agent harness를 위한 tmux pane 매니저인 dmux를 사용하여 병렬 AI agent 세션을 조율(orchestrate)합니다.

## 활성화 시점

- 여러 agent 세션을 병렬로 실행할 때
- Claude Code, Codex 및 기타 harness 간의 작업을 조율할 때
- 분할 정복 병렬 처리의 이점이 있는 복잡한 작업
- 사용자가 "병렬로 실행", "이 작업을 분할", "dmux 사용" 또는 "multi-agent"라고 말할 때

## dmux란 무엇인가

dmux는 AI agent pane을 관리하는 tmux 기반 orchestration 도구입니다:
- 프롬프트와 함께 새 pane을 생성하려면 `n`을 누르세요
- pane 출력을 메인 세션으로 다시 병합(merge)하려면 `m`을 누르세요
- 지원: Claude Code, Codex, OpenCode, Cline, Gemini, Qwen

**설치:** `npm install -g dmux` 또는 [github.com/standardagents/dmux](https://github.com/standardagents/dmux) 참고

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

## Workflow 패턴

### 패턴 1: Research + 구현

리서치와 구현을 병렬 트랙으로 분할합니다:

```
Pane 1 (Research): "Research best practices for rate limiting in Node.js.
  Check current libraries, compare approaches, and write findings to
  /tmp/rate-limit-research.md"

Pane 2 (Implement): "Implement rate limiting middleware for our Express API.
  Start with a basic token bucket, we'll refine after research completes."

# After Pane 1 completes, merge findings into Pane 2's context
```

### 패턴 2: Multi-File 기능

독립적인 파일 전체에서 작업을 병렬화합니다:

```
Pane 1: "Create the database schema and migrations for the billing feature"
Pane 2: "Build the billing API endpoints in src/api/billing/"
Pane 3: "Create the billing dashboard UI components"

# Merge all, then do integration in main pane
```

### 패턴 3: Test + Fix 루프

한 pane에서 테스트를 실행하고 다른 pane에서 수정합니다:

```
Pane 1 (Watcher): "Run the test suite in watch mode. When tests fail,
  summarize the failures."

Pane 2 (Fixer): "Fix failing tests based on the error output from pane 1"
```

### 패턴 4: Cross-Harness

작업마다 서로 다른 AI 도구를 사용합니다:

```
Pane 1 (Claude Code): "Review the security of the auth module"
Pane 2 (Codex): "Refactor the utility functions for performance"
Pane 3 (Claude Code): "Write E2E tests for the checkout flow"
```

### 패턴 5: Code Review 파이프라인

병렬 리뷰 관점:

```
Pane 1: "Review src/api/ for security vulnerabilities"
Pane 2: "Review src/api/ for performance issues"
Pane 3: "Review src/api/ for test coverage gaps"

# Merge all reviews into a single report
```

## Best Practices

1. **독립적인 작업만 수행하세요.** 서로의 출력에 의존하는 작업은 병렬화하지 마세요.
2. **명확한 경계를 설정하세요.** 각 pane은 별개의 파일이나 관심사에서 작업해야 합니다.
3. **전략적으로 merge하세요.** 충돌을 피하기 위해 merge하기 전에 pane 출력을 검토하세요.
4. **git worktree를 사용하세요.** 파일 충돌이 발생하기 쉬운 작업의 경우 pane마다 별도의 worktree를 사용하세요.
5. **리소스 인지.** 각 pane은 API 토큰을 사용하므로 전체 pane 수를 5-6개 미만으로 유지하세요.

## Git Worktree 통합

겹치는 파일을 다루는 작업의 경우:

```bash
# Create worktrees for isolation
git worktree add -b feat/auth ../feature-auth HEAD
git worktree add -b feat/billing ../feature-billing HEAD

# Run agents in separate worktrees
# Pane 1: cd ../feature-auth && claude
# Pane 2: cd ../feature-billing && claude

# Merge branches when done
git merge feat/auth
git merge feat/billing
```

## 보완 도구

| 도구 | 역할 | 사용 시점 |
|------|-------------|-------------|
| **dmux** | agent를 위한 tmux pane 관리 | 병렬 agent 세션 |
| **Superset** | 10개 이상의 병렬 agent를 위한 터미널 IDE | 대규모 orchestration |
| **Claude Code Task tool** | 프로세스 내 subagent 생성 | 세션 내의 프로그래밍 방식 병렬 처리 |
| **Codex multi-agent** | 내장된 agent 역할 | Codex 전용 병렬 작업 |

## ECC Helper

ECC에는 이제 별도의 git worktree를 사용한 외부 tmux-pane orchestration을 위한 helper가 포함되어 있습니다:

```bash
node scripts/orchestrate-worktrees.js plan.json --execute
```

`plan.json` 예시:

```json
{
  "sessionName": "skill-audit",
  "baseRef": "HEAD",
  "launcherCommand": "codex exec --cwd {worktree_path} --task-file {task_file}",
  "workers": [
    { "name": "docs-a", "task": "Fix skills 1-4 and write handoff notes." },
    { "name": "docs-b", "task": "Fix skills 5-8 and write handoff notes." }
  ]
}
```

helper의 역할:
- worker당 하나의 브랜치 기반 git worktree를 생성합니다.
- 선택적으로 메인 체크아웃에서 선택한 `seedPaths`를 각 worker worktree에 덮어씌웁니다.
- `.orchestration/<session>/` 아래에 worker별 `task.md`, `handoff.md`, `status.md` 파일을 작성합니다.
- worker당 하나의 pane이 있는 tmux 세션을 시작합니다.
- 각 worker 명령을 고유한 pane에서 실행합니다.
- 메인 pane은 orchestrator를 위해 비워 둡니다.

로컬 orchestration script, 초안 계획 또는 문서와 같이 아직 HEAD의 일부가 아닌 dirty 또는 untracked 로컬 파일에 worker가 액세스해야 할 때 `seedPaths`를 사용하세요:

```json
{
  "sessionName": "workflow-e2e",
  "seedPaths": [
    "scripts/orchestrate-worktrees.js",
    "scripts/lib/tmux-worktree-orchestrator.js",
    ".claude/plan/workflow-e2e-test.json"
  ],
  "launcherCommand": "bash {repo_root}/scripts/orchestrate-codex-worker.sh {task_file} {handoff_file} {status_file}",
  "workers": [
    { "name": "seed-check", "task": "Verify seeded files are present before starting work." }
  ]
}
```

## Troubleshooting

- **Pane이 응답하지 않음:** 해당 pane으로 직접 전환하거나 `tmux capture-pane -pt <session>:0.<pane-index>`로 검사하세요.
- **Merge 충돌:** pane별로 파일 변경 사항을 격리하려면 git worktree를 사용하세요.
- **높은 토큰 사용량:** 병렬 pane 수를 줄이세요. 각 pane은 전체 agent 세션입니다.
- **tmux를 찾을 수 없음:** `brew install tmux`(macOS) 또는 `apt install tmux`(Linux)로 설치하세요.

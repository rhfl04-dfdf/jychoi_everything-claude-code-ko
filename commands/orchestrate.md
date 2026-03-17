# Orchestrate Command

복잡한 작업을 위한 순차적 agent 워크플로우입니다.

## 사용법

`/orchestrate [workflow-type] [task-description]`

## 워크플로우 유형

### feature
전체 기능 구현 워크플로우:
```
planner -> tdd-guide -> code-reviewer -> security-reviewer
```

### bugfix
버그 조사 및 수정 워크플로우:
```
planner -> tdd-guide -> code-reviewer
```

### refactor
안전한 리팩토링 워크플로우:
```
architect -> code-reviewer -> tdd-guide
```

### security
보안 중심 리뷰:
```
security-reviewer -> code-reviewer -> architect
```

## 실행 패턴

워크플로우의 각 agent에 대해:

1. **agent 호출** - 이전 agent의 컨텍스트와 함께
2. **출력 수집** - 구조화된 핸드오프 문서로
3. **다음 agent에 전달** - 체인의 다음 단계로
4. **결과 집계** - 최종 보고서로

## 핸드오프 문서 형식

agent 간에 핸드오프 문서를 작성합니다:

```markdown
## HANDOFF: [previous-agent] -> [next-agent]

### Context
[Summary of what was done]

### Findings
[Key discoveries or decisions]

### Files Modified
[List of files touched]

### Open Questions
[Unresolved items for next agent]

### Recommendations
[Suggested next steps]
```

## 예시: Feature 워크플로우

```
/orchestrate feature "Add user authentication"
```

실행 순서:

1. **Planner Agent**
   - 요구사항 분석
   - 구현 계획 수립
   - 의존성 식별
   - 출력: `HANDOFF: planner -> tdd-guide`

2. **TDD Guide Agent**
   - planner 핸드오프 읽기
   - 테스트 먼저 작성
   - 테스트 통과를 위한 구현
   - 출력: `HANDOFF: tdd-guide -> code-reviewer`

3. **Code Reviewer Agent**
   - 구현 리뷰
   - 이슈 확인
   - 개선사항 제안
   - 출력: `HANDOFF: code-reviewer -> security-reviewer`

4. **Security Reviewer Agent**
   - 보안 감사
   - 취약점 점검
   - 최종 승인
   - 출력: 최종 보고서

## 최종 보고서 형식

```
ORCHESTRATION REPORT
====================
Workflow: feature
Task: Add user authentication
Agents: planner -> tdd-guide -> code-reviewer -> security-reviewer

SUMMARY
-------
[One paragraph summary]

AGENT OUTPUTS
-------------
Planner: [summary]
TDD Guide: [summary]
Code Reviewer: [summary]
Security Reviewer: [summary]

FILES CHANGED
-------------
[List all files modified]

TEST RESULTS
------------
[Test pass/fail summary]

SECURITY STATUS
---------------
[Security findings]

RECOMMENDATION
--------------
[SHIP / NEEDS WORK / BLOCKED]
```

## 병렬 실행

독립적인 검사의 경우 agent를 병렬로 실행합니다:

```markdown
### Parallel Phase
Run simultaneously:
- code-reviewer (quality)
- security-reviewer (security)
- architect (design)

### Merge Results
Combine outputs into single report
```

외부 tmux-pane 워커를 별도의 git worktree와 함께 사용하려면 `node scripts/orchestrate-worktrees.js plan.json --execute`를 실행하세요. 내장 오케스트레이션 패턴은 프로세스 내에서 유지되며, 헬퍼는 장시간 실행되거나 크로스 하네스 세션을 위한 것입니다.

워커가 메인 체크아웃의 dirty 또는 untracked 로컬 파일을 확인해야 할 때 계획 파일에 `seedPaths`를 추가하세요. ECC는 `git worktree add` 후 선택된 경로만 각 워커 worktree에 오버레이하여, 브랜치를 격리된 상태로 유지하면서도 진행 중인 로컬 스크립트, 계획 또는 문서를 노출합니다.

```json
{
  "sessionName": "workflow-e2e",
  "seedPaths": [
    "scripts/orchestrate-worktrees.js",
    "scripts/lib/tmux-worktree-orchestrator.js",
    ".claude/plan/workflow-e2e-test.json"
  ],
  "workers": [
    { "name": "docs", "task": "Update orchestration docs." }
  ]
}
```

라이브 tmux/worktree 세션의 control-plane 스냅샷을 내보내려면 다음을 실행하세요:

```bash
node scripts/orchestration-status.js .claude/plan/workflow-visual-proof.json
```

스냅샷에는 세션 활동, tmux pane 메타데이터, 워커 상태, 목표, 시드된 오버레이, 최근 핸드오프 요약이 JSON 형식으로 포함됩니다.

## Operator Command-Center 핸드오프

워크플로우가 여러 세션, worktree 또는 tmux pane에 걸쳐 있을 때 최종 핸드오프에 control-plane 블록을 추가하세요:

```markdown
CONTROL PLANE
-------------
Sessions:
- active session ID or alias
- branch + worktree path for each active worker
- tmux pane or detached session name when applicable

Diffs:
- git status summary
- git diff --stat for touched files
- merge/conflict risk notes

Approvals:
- pending user approvals
- blocked steps awaiting confirmation

Telemetry:
- last activity timestamp or idle signal
- estimated token or cost drift
- policy events raised by hooks or reviewers
```

이를 통해 planner, implementer, reviewer, loop 워커를 operator 화면에서 파악할 수 있습니다.

## 인자

$ARGUMENTS:
- `feature <description>` - 전체 기능 워크플로우
- `bugfix <description>` - 버그 수정 워크플로우
- `refactor <description>` - 리팩토링 워크플로우
- `security <description>` - 보안 리뷰 워크플로우
- `custom <agents> <description>` - 커스텀 agent 시퀀스

## 커스텀 워크플로우 예시

```
/orchestrate custom "architect,tdd-guide,code-reviewer" "Redesign caching layer"
```

## 팁

1. **복잡한 기능은 planner로 시작** - planner를 먼저 실행하세요
2. **병합 전에 항상 code-reviewer 포함** - 코드 리뷰를 빠뜨리지 마세요
3. **인증/결제/PII에는 security-reviewer 사용** - 보안 리뷰가 필요합니다
4. **핸드오프는 간결하게 유지** - 다음 agent에 필요한 내용에 집중하세요
5. **필요시 agent 간 검증 실행** - 중간 확인 단계를 추가하세요

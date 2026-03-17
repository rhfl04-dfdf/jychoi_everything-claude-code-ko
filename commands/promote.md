---
name: promote
description: 프로젝트 범위의 instincts를 글로벌 범위로 승격합니다.
command: true
---

# Promote 명령

continuous-learning-v2에서 instincts를 프로젝트 범위에서 글로벌 범위로 승격합니다.

## 구현

플러그인 루트 경로를 사용하여 instinct CLI를 실행합니다:

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/skills/continuous-learning-v2/scripts/instinct-cli.py" promote [instinct-id] [--force] [--dry-run]
```

또는 CLAUDE_PLUGIN_ROOT가 설정되지 않은 경우 (수동 설치):

```bash
python3 ~/.claude/skills/continuous-learning-v2/scripts/instinct-cli.py promote [instinct-id] [--force] [--dry-run]
```

## 사용법

```bash
/promote                      # Auto-detect promotion candidates
/promote --dry-run            # Preview auto-promotion candidates
/promote --force              # Promote all qualified candidates without prompt
/promote grep-before-edit     # Promote one specific instinct from current project
```

## 수행할 작업

1. 현재 프로젝트 감지
2. `instinct-id`가 제공되면, 해당 instinct만 승격 (현재 프로젝트에 있는 경우)
3. 그렇지 않은 경우, 다음 조건에 맞는 프로젝트 간 후보를 탐색:
   - 최소 2개 이상의 프로젝트에서 나타남
   - 신뢰도 임계값 충족
4. 승격된 instincts를 `scope: global` 설정과 함께 `~/.claude/homunculus/instincts/personal/` 경로에 기록

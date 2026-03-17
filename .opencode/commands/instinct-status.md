---
description: 학습된 instinct(프로젝트 + 전역)와 신뢰도 표시
agent: build
---

# Instinct Status 명령어

continuous-learning-v2의 instinct 상태를 표시합니다: $ARGUMENTS

## 작업 내용

실행:

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/skills/continuous-learning-v2/scripts/instinct-cli.py" status
```

`CLAUDE_PLUGIN_ROOT`를 사용할 수 없는 경우:

```bash
python3 ~/.claude/skills/continuous-learning-v2/scripts/instinct-cli.py status
```

## 동작 참고사항

- 출력에는 프로젝트 범위 및 전역 instinct가 모두 포함됩니다.
- ID가 충돌하는 경우 프로젝트 instinct가 전역 instinct를 재정의합니다.
- 출력은 도메인별로 그룹화되며 신뢰도 막대가 표시됩니다.
- 이 명령어는 v2.1에서 추가 필터를 지원하지 않습니다.

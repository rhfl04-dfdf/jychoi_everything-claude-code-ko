---
description: 프로젝트 instinct를 글로벌 범위로 승격
agent: build
---

# Promote 커맨드

continuous-learning-v2에서 instinct 승격: $ARGUMENTS

## 작업 내용

실행:

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/skills/continuous-learning-v2/scripts/instinct-cli.py" promote $ARGUMENTS
```

`CLAUDE_PLUGIN_ROOT`를 사용할 수 없는 경우:

```bash
python3 ~/.claude/skills/continuous-learning-v2/scripts/instinct-cli.py promote $ARGUMENTS
```

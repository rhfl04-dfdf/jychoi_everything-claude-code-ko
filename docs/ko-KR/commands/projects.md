---
name: projects
description: 알려진 프로젝트 목록과 해당 instinct 통계를 표시합니다
command: true
---

# Projects 커맨드

continuous-learning-v2의 프로젝트 레지스트리 항목과 프로젝트별 instinct/observation 수를 나열합니다.

## 구현

플러그인 루트 경로를 사용하여 instinct CLI 실행:

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/skills/continuous-learning-v2/scripts/instinct-cli.py" projects
```

또는 `CLAUDE_PLUGIN_ROOT`가 설정되지 않은 경우 (수동 설치):

```bash
python3 ~/.claude/skills/continuous-learning-v2/scripts/instinct-cli.py projects
```

## 사용법

```bash
/projects
```

## 수행 작업

1. `~/.claude/homunculus/projects.json` 읽기
2. 각 프로젝트에 대해 다음을 표시:
   - 프로젝트 이름, id, root, remote
   - 개인 및 상속된 instinct 수
   - Observation 이벤트 수
   - 마지막 확인 타임스탬프
3. 전체 instinct 합계도 표시

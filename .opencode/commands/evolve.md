---
description: instinct를 분석하고 진화된 구조를 제안 또는 생성
agent: build
---

# Evolve 커맨드

continuous-learning-v2에서 instinct 분석 및 진화: $ARGUMENTS

## 작업 내용

실행:

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/skills/continuous-learning-v2/scripts/instinct-cli.py" evolve $ARGUMENTS
```

`CLAUDE_PLUGIN_ROOT`를 사용할 수 없는 경우:

```bash
python3 ~/.claude/skills/continuous-learning-v2/scripts/instinct-cli.py evolve $ARGUMENTS
```

## 지원 인수 (v2.1)

- 인수 없음: 분석만 수행
- `--generate`: `evolved/{skills,commands,agents}` 하위에 파일도 생성

## 동작 참고사항

- 분석에 프로젝트 + 글로벌 instinct를 사용합니다.
- trigger 및 도메인 클러스터링에서 skill/command/agent 후보를 표시합니다.
- 프로젝트 -> 글로벌 승격 후보를 표시합니다.
- `--generate` 사용 시 출력 경로:
  - 프로젝트 컨텍스트: `~/.claude/homunculus/projects/<project-id>/evolved/`
  - 글로벌 폴백: `~/.claude/homunculus/evolved/`

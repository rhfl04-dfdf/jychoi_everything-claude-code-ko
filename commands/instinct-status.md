---
name: instinct-status
description: 신뢰도와 함께 학습된 instinct(프로젝트 + 글로벌)를 표시합니다
command: true
---

# Instinct Status 커맨드

현재 프로젝트의 학습된 instinct와 글로벌 instinct를 도메인별로 그룹화하여 표시합니다.

## 구현

플러그인 루트 경로를 사용하여 instinct CLI를 실행합니다:

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/skills/continuous-learning-v2/scripts/instinct-cli.py" status
```

또는 `CLAUDE_PLUGIN_ROOT`가 설정되지 않은 경우 (수동 설치):

```bash
python3 ~/.claude/skills/continuous-learning-v2/scripts/instinct-cli.py status
```

## 사용법

```
/instinct-status
```

## 수행할 작업

1. 현재 프로젝트 컨텍스트 감지 (git remote/path 해시)
2. `~/.claude/homunculus/projects/<project-id>/instincts/`에서 프로젝트 instinct 읽기
3. `~/.claude/homunculus/instincts/`에서 글로벌 instinct 읽기
4. 우선순위 규칙과 함께 병합 (ID 충돌 시 프로젝트가 글로벌 override)
5. 신뢰도 바와 관찰 통계와 함께 도메인별로 그룹화하여 표시

## 출력 형식

```
============================================================
  INSTINCT STATUS - 12 total
============================================================

  Project: my-app (a1b2c3d4e5f6)
  Project instincts: 8
  Global instincts:  4

## PROJECT-SCOPED (my-app)
  ### WORKFLOW (3)
    ███████░░░  70%  grep-before-edit [project]
              trigger: when modifying code

## GLOBAL (apply to all projects)
  ### SECURITY (2)
    █████████░  85%  validate-user-input [global]
              trigger: when handling user input
```

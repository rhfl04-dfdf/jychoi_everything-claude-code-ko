---
name: instinct-import
description: 파일 또는 URL에서 프로젝트/글로벌 범위로 instinct를 가져옵니다
command: true
---

# Instinct Import 커맨드

## 구현

플러그인 루트 경로를 사용하여 instinct CLI를 실행합니다:

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/skills/continuous-learning-v2/scripts/instinct-cli.py" import <file-or-url> [--dry-run] [--force] [--min-confidence 0.7] [--scope project|global]
```

또는 `CLAUDE_PLUGIN_ROOT`가 설정되지 않은 경우 (수동 설치):

```bash
python3 ~/.claude/skills/continuous-learning-v2/scripts/instinct-cli.py import <file-or-url>
```

로컬 파일 경로 또는 HTTP(S) URL에서 instinct를 가져옵니다.

## 사용법

```
/instinct-import team-instincts.yaml
/instinct-import https://github.com/org/repo/instincts.yaml
/instinct-import team-instincts.yaml --dry-run
/instinct-import team-instincts.yaml --scope global --force
```

## 수행할 작업

1. instinct 파일 가져오기 (로컬 경로 또는 URL)
2. 형식 파싱 및 유효성 검사
3. 기존 instinct와 중복 확인
4. 새 instinct 병합 또는 추가
5. inherited instinct 디렉토리에 저장:
   - 프로젝트 범위: `~/.claude/homunculus/projects/<project-id>/instincts/inherited/`
   - 글로벌 범위: `~/.claude/homunculus/instincts/inherited/`

## 가져오기 프로세스

```
📥 Importing instincts from: team-instincts.yaml
================================================

Found 12 instincts to import.

Analyzing conflicts...

## New Instincts (8)
These will be added:
  ✓ use-zod-validation (confidence: 0.7)
  ✓ prefer-named-exports (confidence: 0.65)
  ✓ test-async-functions (confidence: 0.8)
  ...

## Duplicate Instincts (3)
Already have similar instincts:
  ⚠️ prefer-functional-style
     Local: 0.8 confidence, 12 observations
     Import: 0.7 confidence
     → Keep local (higher confidence)

  ⚠️ test-first-workflow
     Local: 0.75 confidence
     Import: 0.9 confidence
     → Update to import (higher confidence)

Import 8 new, update 1?
```

## 병합 동작

기존 ID를 가진 instinct를 가져올 때:
- 더 높은 신뢰도의 import는 업데이트 후보가 됩니다
- 같거나 낮은 신뢰도의 import는 건너뜁니다
- `--force`를 사용하지 않는 한 사용자가 확인합니다

## 소스 추적

가져온 instinct는 다음과 같이 표시됩니다:
```yaml
source: inherited
scope: project
imported_from: "team-instincts.yaml"
project_id: "a1b2c3d4e5f6"
project_name: "my-project"
```

## 플래그

- `--dry-run`: 가져오기 없이 미리 보기
- `--force`: 확인 프롬프트 건너뜀
- `--min-confidence <n>`: 임계값 이상의 instinct만 가져오기
- `--scope <project|global>`: 대상 범위 선택 (기본값: `project`)

## 출력

가져오기 후:
```
✅ Import complete!

Added: 8 instincts
Updated: 1 instinct
Skipped: 3 instincts (equal/higher confidence already exists)

New instincts saved to: ~/.claude/homunculus/instincts/inherited/

Run /instinct-status to see all instincts.
```

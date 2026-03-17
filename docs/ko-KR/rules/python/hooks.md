---
paths:
  - "**/*.py"
  - "**/*.pyi"
---
# Python Hooks

> 이 파일은 [common/hooks.md](../common/hooks.md)를 확장하여 Python 특화 내용을 다룹니다.

## PostToolUse Hooks

`~/.claude/settings.json`에서 설정:

- **black/ruff**: 편집 후 `.py` 파일 자동 포맷팅
- **mypy/pyright**: `.py` 파일 편집 후 타입 검사 실행

## 경고

- 편집된 파일의 `print()` 구문 경고 (대신 `logging` 모듈 사용)

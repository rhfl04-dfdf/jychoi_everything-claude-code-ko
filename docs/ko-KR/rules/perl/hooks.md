---
paths:
  - "**/*.pl"
  - "**/*.pm"
  - "**/*.t"
  - "**/*.psgi"
  - "**/*.cgi"
---
# Perl Hooks

> 이 파일은 [common/hooks.md](../common/hooks.md)를 확장하여 Perl 특화 내용을 다룹니다.

## PostToolUse Hooks

`~/.claude/settings.json`에서 설정:

- **perltidy**: 편집 후 `.pl` 및 `.pm` 파일 자동 포맷팅
- **perlcritic**: `.pm` 파일 편집 후 린트 검사 실행

## 경고

- 스크립트가 아닌 `.pm` 파일에서의 `print` 사용 경고 — `say` 또는 로깅 모듈 사용 (예: `Log::Any`)

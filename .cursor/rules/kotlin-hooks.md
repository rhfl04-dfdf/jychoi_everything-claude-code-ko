---
paths:
  - "**/*.kt"
  - "**/*.kts"
  - "**/build.gradle.kts"
---
# Kotlin Hooks

> 이 파일은 [common/hooks.md](../common/hooks.md)를 확장하여 Kotlin 특화 내용을 다룹니다.

## PostToolUse Hooks

`~/.claude/settings.json`에서 설정:

- **ktfmt/ktlint**: 편집 후 `.kt` 및 `.kts` 파일 자동 포맷팅
- **detekt**: Kotlin 파일 편집 후 정적 분석 실행
- **./gradlew build**: 변경 후 컴파일 검증

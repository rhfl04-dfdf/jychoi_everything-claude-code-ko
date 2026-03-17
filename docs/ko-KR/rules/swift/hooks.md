---
paths:
  - "**/*.swift"
  - "**/Package.swift"
---
# Swift Hooks

> 이 파일은 [common/hooks.md](../common/hooks.md)를 확장하여 Swift 특화 내용을 다룹니다.

## PostToolUse Hooks

`~/.claude/settings.json`에서 설정:

- **SwiftFormat**: 편집 후 `.swift` 파일 자동 포맷팅
- **SwiftLint**: `.swift` 파일 편집 후 린트 검사 실행
- **swift build**: 편집 후 수정된 패키지 타입 검사

## 경고

`print()` 구문에 플래그 — 프로덕션 코드에서는 `os.Logger` 또는 구조화된 로깅 사용.

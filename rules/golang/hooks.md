---
paths:
  - "**/*.go"
  - "**/go.mod"
  - "**/go.sum"
---
# Go Hooks

> 이 파일은 [common/hooks.md](../common/hooks.md)를 확장하여 Go 특화 내용을 다룹니다.

## PostToolUse Hooks

`~/.claude/settings.json`에서 설정:

- **gofmt/goimports**: 편집 후 `.go` 파일 자동 포맷팅
- **go vet**: `.go` 파일 편집 후 정적 분석 실행
- **staticcheck**: 수정된 패키지에 대한 확장 정적 검사 실행

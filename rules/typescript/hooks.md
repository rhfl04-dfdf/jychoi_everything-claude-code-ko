---
paths:
  - "**/*.ts"
  - "**/*.tsx"
  - "**/*.js"
  - "**/*.jsx"
---
# TypeScript/JavaScript Hooks

> 이 파일은 [common/hooks.md](../common/hooks.md)를 TypeScript/JavaScript 관련 내용으로 확장합니다.

## PostToolUse Hooks

`~/.claude/settings.json`에서 설정:

- **Prettier**: 수정 후 JS/TS 파일 자동 포맷팅
- **TypeScript check**: .ts/.tsx 파일 수정 후 tsc 실행
- **console.log warning**: 수정된 파일 내 console.log 경고

## Stop Hooks

- **console.log audit**: session 종료 전 모든 수정된 파일의 console.log 점검

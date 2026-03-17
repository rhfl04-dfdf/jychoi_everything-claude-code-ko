---
paths:
  - "**/*.ts"
  - "**/*.tsx"
  - "**/*.js"
  - "**/*.jsx"
---
# TypeScript/JavaScript 보안

> 이 파일은 TypeScript/JavaScript에 특화된 내용을 포함하여 [common/security.md](../common/security.md)를 확장합니다.

## 비밀 관리

```typescript
// NEVER: Hardcoded secrets
const apiKey = "sk-proj-xxxxx"

// ALWAYS: Environment variables
const apiKey = process.env.OPENAI_API_KEY

if (!apiKey) {
  throw new Error('OPENAI_API_KEY not configured')
}
```

## agent 지원

- 포괄적인 보안 감사를 위해 **security-reviewer** skill을 사용하세요.

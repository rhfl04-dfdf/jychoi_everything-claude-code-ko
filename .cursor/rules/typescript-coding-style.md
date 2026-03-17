---
description: "공통 규칙을 확장하는 TypeScript 코딩 스타일"
globs: ["**/*.ts", "**/*.tsx", "**/*.js", "**/*.jsx"]
alwaysApply: false
---
# TypeScript/JavaScript 코딩 스타일

> 이 파일은 공통 코딩 스타일 규칙을 TypeScript/JavaScript 관련 내용으로 확장합니다.

## 불변성

불변 업데이트를 위해 스프레드 연산자를 사용합니다:

```typescript
// WRONG: Mutation
function updateUser(user, name) {
  user.name = name  // MUTATION!
  return user
}

// CORRECT: Immutability
function updateUser(user, name) {
  return {
    ...user,
    name
  }
}
```

## 에러 처리

async/await와 try-catch를 사용합니다:

```typescript
try {
  const result = await riskyOperation()
  return result
} catch (error) {
  console.error('Operation failed:', error)
  throw new Error('Detailed user-friendly message')
}
```

## 입력 검증

스키마 기반 검증을 위해 Zod를 사용합니다:

```typescript
import { z } from 'zod'

const schema = z.object({
  email: z.string().email(),
  age: z.number().int().min(0).max(150)
})

const validated = schema.parse(input)
```

## Console.log

- 프로덕션 코드에 `console.log` 문 사용 금지
- 대신 적절한 로깅 라이브러리 사용
- 자동 감지에 대해서는 hook 참조

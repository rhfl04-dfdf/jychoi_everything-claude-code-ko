---
paths:
  - "**/*.ts"
  - "**/*.tsx"
  - "**/*.js"
  - "**/*.jsx"
---
# TypeScript/JavaScript 코딩 스타일

> 이 파일은 [common/coding-style.md](../common/coding-style.md)의 TypeScript/JavaScript 관련 내용을 확장합니다.

## Types와 Interfaces

Public APIs, 공유 모델, 컴포넌트 props를 명확하고 가독성 있으며 재사용 가능하게 만들기 위해 types를 사용하세요.

### Public APIs

- 내보낸 함수, 공용 유틸리티, public 클래스 메서드에 매개변수 및 반환 타입을 추가하세요.
- 명백한 지역 변수 타입은 TypeScript가 추론하도록 하세요.
- 반복되는 인라인 객체 구조는 명명된 types 또는 interfaces로 추출하세요.

```typescript
// WRONG: Exported function without explicit types
export function formatUser(user) {
  return `${user.firstName} ${user.lastName}`
}

// CORRECT: Explicit types on public APIs
interface User {
  firstName: string
  lastName: string
}

export function formatUser(user: User): string {
  return `${user.firstName} ${user.lastName}`
}
```

### Interfaces vs. Type Aliases

- 확장되거나 구현될 수 있는 객체 구조에는 `interface`를 사용하세요.
- unions, intersections, tuples, mapped types, utility types에는 `type`을 사용하세요.
- 상호 운용성을 위해 `enum`이 필요한 경우가 아니면 `enum`보다 string 리터럴 유니온을 선호하세요.

```typescript
interface User {
  id: string
  email: string
}

type UserRole = 'admin' | 'member'
type UserWithRole = User & {
  role: UserRole
}
```

### Avoid `any`

- 애플리케이션 코드에서 `any`를 피하세요.
- 외부 또는 신뢰할 수 없는 입력에는 `unknown`을 사용하고 안전하게 범위를 좁히세요.
- 값의 타입이 호출자에 따라 달라지는 경우 generics를 사용하세요.

```typescript
// WRONG: any removes type safety
function getErrorMessage(error: any) {
  return error.message
}

// CORRECT: unknown forces safe narrowing
function getErrorMessage(error: unknown): string {
  if (error instanceof Error) {
    return error.message
  }

  return 'Unexpected error'
}
```

### React Props

- 컴포넌트 props는 명명된 `interface` 또는 `type`으로 정의하세요.
- 콜백 props의 타입을 명시적으로 지정하세요.
- 특별한 이유가 없는 한 `React.FC`를 사용하지 마세요.

```typescript
interface User {
  id: string
  email: string
}

interface UserCardProps {
  user: User
  onSelect: (id: string) => void
}

function UserCard({ user, onSelect }: UserCardProps) {
  return <button onClick={() => onSelect(user.id)}>{user.email}</button>
}
```

### JavaScript 파일

- `.js` 및 `.jsx` 파일에서 타입이 명확성을 높여주고 TypeScript 마이그레이션이 실질적이지 않은 경우 JSDoc을 사용하세요.
- JSDoc을 런타임 동작과 일치하게 유지하세요.

```javascript
/**
 * @param {{ firstName: string, lastName: string }} user
 * @returns {string}
 */
export function formatUser(user) {
  return `${user.firstName} ${user.lastName}`
}
```

## Immutability

불변 업데이트를 위해 spread 연산자를 사용하세요:

```typescript
interface User {
  id: string
  name: string
}

// WRONG: Mutation
function updateUser(user: User, name: string): User {
  user.name = name // MUTATION!
  return user
}

// CORRECT: Immutability
function updateUser(user: Readonly<User>, name: string): User {
  return {
    ...user,
    name
  }
}
```

## 에러 핸들링

try-catch와 함께 async/await를 사용하고 unknown 에러를 안전하게 좁히세요:

```typescript
interface User {
  id: string
  email: string
}

declare function riskyOperation(userId: string): Promise<User>

function getErrorMessage(error: unknown): string {
  if (error instanceof Error) {
    return error.message
  }

  return 'Unexpected error'
}

const logger = {
  error: (message: string, error: unknown) => {
    // Replace with your production logger (for example, pino or winston).
  }
}

async function loadUser(userId: string): Promise<User> {
  try {
    const result = await riskyOperation(userId)
    return result
  } catch (error: unknown) {
    logger.error('Operation failed', error)
    throw new Error(getErrorMessage(error))
  }
}
```

## 입력 검증

스키마 기반 검증을 위해 Zod를 사용하고 스키마에서 타입을 추론하세요:

```typescript
import { z } from 'zod'

const userSchema = z.object({
  email: z.string().email(),
  age: z.number().int().min(0).max(150)
})

type UserInput = z.infer<typeof userSchema>

const validated: UserInput = userSchema.parse(input)
```

## Console.log

- 프로덕션 코드에 `console.log` 문을 남기지 마세요.
- 대신 적절한 로깅 라이브러리를 사용하세요.
- 자동 감지에 대해서는 hooks를 참조하세요.

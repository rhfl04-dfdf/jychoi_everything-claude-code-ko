# 예제 프로젝트 CLAUDE.md

이것은 프로젝트 수준의 CLAUDE.md 예제 파일입니다. 프로젝트 루트에 배치하세요.

## 프로젝트 개요

[프로젝트에 대한 간략한 설명 - 기능, 기술 스택]

## 핵심 규칙

### 1. 코드 구성

- 큰 파일 몇 개보다 작은 파일 여러 개
- 높은 응집도, 낮은 결합도
- 일반적으로 200~400줄, 파일당 최대 800줄
- 타입별이 아닌 기능/도메인별로 구성

### 2. 코드 스타일

- 코드, 주석, 문서에 이모지 사용 금지
- 항상 불변성 유지 - 객체나 배열 변경 금지
- 프로덕션 코드에 console.log 사용 금지
- try/catch를 이용한 적절한 에러 처리
- Zod 또는 유사 라이브러리로 입력값 검증

### 3. 테스트

- TDD: 테스트를 먼저 작성
- 최소 80% 커버리지
- 유틸리티에 대한 단위 테스트
- API에 대한 통합 테스트
- 중요 흐름에 대한 E2E 테스트

### 4. 보안

- 하드코딩된 시크릿 금지
- 민감한 데이터는 환경 변수 사용
- 모든 사용자 입력값 검증
- 매개변수화된 쿼리만 사용
- CSRF 보호 활성화

## 파일 구조

```
src/
|-- app/              # Next.js app router
|-- components/       # Reusable UI components
|-- hooks/            # Custom React hooks
|-- lib/              # Utility libraries
|-- types/            # TypeScript definitions
```

## 주요 패턴

### API 응답 형식

```typescript
interface ApiResponse<T> {
  success: boolean
  data?: T
  error?: string
}
```

### 에러 처리

```typescript
try {
  const result = await operation()
  return { success: true, data: result }
} catch (error) {
  console.error('Operation failed:', error)
  return { success: false, error: 'User-friendly message' }
}
```

## 환경 변수

```bash
# Required
DATABASE_URL=
API_KEY=

# Optional
DEBUG=false
```

## 사용 가능한 명령어

- `/tdd` - 테스트 주도 개발 워크플로우
- `/plan` - 구현 계획 생성
- `/code-review` - 코드 품질 리뷰
- `/build-fix` - 빌드 오류 수정

## Git 워크플로우

- 컨벤셔널 커밋: `feat:`, `fix:`, `refactor:`, `docs:`, `test:`
- main에 직접 커밋 금지
- PR은 리뷰 필요
- 머지 전 모든 테스트 통과 필수

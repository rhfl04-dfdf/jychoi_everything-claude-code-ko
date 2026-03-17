# 프로젝트 레벨 CLAUDE.md 예시

이것은 프로젝트 레벨 CLAUDE.md 파일의 예시입니다. 프로젝트 루트에 배치하세요.

## 프로젝트 개요

[프로젝트 간략 설명 - 기능, 기술 스택]

## 중요한 규칙

### 1. 코드 구성

- 소수의 큰 파일보다 다수의 작은 파일
- 높은 응집도, 낮은 결합도
- 일반적으로 200-400줄, 파일당 최대 800줄
- 타입이 아닌, 기능/도메인별로 정리

### 2. 코드 스타일

- 코드, 주석, 문서에 이모지를 사용하지 않는다
- 항상 불변성을 유지한다 - 객체나 배열을 변경하지 않는다
- 프로덕션 코드에 console.log를 사용하지 않는다
- try/catch로 적절한 에러 핸들링
- Zod 등으로 입력 검증

### 3. 테스트

- TDD: 먼저 테스트를 작성한다
- 최소 80% 커버리지
- 유틸리티의 단위 테스트
- API의 통합 테스트
- 중요한 플로우의 E2E 테스트

### 4. 보안

- 하드코딩된 시크릿을 사용하지 않는다
- 민감한 데이터에는 환경 변수를 사용
- 모든 사용자 입력을 검증
- 파라미터화된 쿼리만 사용
- CSRF 보호를 활성화

## 파일 구조

```
src/
|-- app/              # Next.js app router
|-- components/       # 再利用可能なUIコンポーネント
|-- hooks/            # カスタムReactフック
|-- lib/              # ユーティリティライブラリ
|-- types/            # TypeScript定義
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

### 에러 핸들링

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
# 必須
DATABASE_URL=
API_KEY=

# オプション
DEBUG=false
```

## 사용 가능한 Command

- `/tdd` - 테스트 주도 개발 워크플로우
- `/plan` - 구현 계획 작성
- `/code-review` - 코드 품질 리뷰
- `/build-fix` - 빌드 오류 수정

## Git 워크플로우

- Conventional Commits: `feat:`, `fix:`, `refactor:`, `docs:`, `test:`
- main에 직접 커밋하지 않는다
- PR에는 리뷰가 필요하다
- 머지 전에 모든 테스트가 통과해야 한다

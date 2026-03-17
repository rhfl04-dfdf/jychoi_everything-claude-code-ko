# Everything Claude Code - OpenCode 지침

이 문서는 OpenCode와 함께 사용하기 위해 Claude Code 설정의 핵심 규칙과 가이드라인을 통합합니다.

## 보안 지침 (중요)

### 필수 보안 검사

모든 커밋 전:
- [ ] 하드코딩된 시크릿 없음 (API 키, 비밀번호, 토큰)
- [ ] 모든 사용자 입력 검증
- [ ] SQL 인젝션 방지 (매개변수화된 쿼리)
- [ ] XSS 방지 (살균된 HTML)
- [ ] CSRF 보호 활성화
- [ ] 인증/인가 검증
- [ ] 모든 엔드포인트에 속도 제한
- [ ] 오류 메시지가 민감한 데이터를 누출하지 않음

### 시크릿 관리

```typescript
// NEVER: Hardcoded secrets
const apiKey = "sk-proj-xxxxx"

// ALWAYS: Environment variables
const apiKey = process.env.OPENAI_API_KEY

if (!apiKey) {
  throw new Error('OPENAI_API_KEY not configured')
}
```

### 보안 대응 프로토콜

보안 문제 발견 시:
1. 즉시 중단
2. **security-reviewer** agent 사용
3. 계속하기 전에 심각한 문제 수정
4. 노출된 시크릿 교체
5. 유사한 문제가 있는지 전체 코드베이스 검토

---

## 코딩 스타일

### 불변성 (중요)

항상 새 객체를 생성하고, 절대 변경하지 마세요:

```javascript
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

### 파일 구성

많은 작은 파일 > 적은 큰 파일:
- 높은 응집도, 낮은 결합도
- 일반적으로 200-400줄, 최대 800줄
- 큰 컴포넌트에서 유틸리티 추출
- 타입별이 아닌 기능/도메인별로 구성

### 오류 처리

항상 오류를 포괄적으로 처리:

```typescript
try {
  const result = await riskyOperation()
  return result
} catch (error) {
  console.error('Operation failed:', error)
  throw new Error('Detailed user-friendly message')
}
```

### 입력 유효성 검사

항상 사용자 입력 유효성 검사:

```typescript
import { z } from 'zod'

const schema = z.object({
  email: z.string().email(),
  age: z.number().int().min(0).max(150)
})

const validated = schema.parse(input)
```

### 코드 품질 체크리스트

작업 완료 전:
- [ ] 코드가 읽기 쉽고 이름이 적절함
- [ ] 함수가 작음 (50줄 미만)
- [ ] 파일이 집중되어 있음 (800줄 미만)
- [ ] 깊은 중첩 없음 (4단계 초과)
- [ ] 적절한 오류 처리
- [ ] console.log 문 없음
- [ ] 하드코딩된 값 없음
- [ ] 변경 없음 (불변 패턴 사용)

---

## 테스트 요구사항

### 최소 테스트 커버리지: 80%

테스트 유형 (모두 필수):
1. **단위 테스트** - 개별 함수, 유틸리티, 컴포넌트
2. **통합 테스트** - API 엔드포인트, 데이터베이스 작업
3. **E2E 테스트** - 핵심 사용자 흐름 (Playwright)

### 테스트 주도 개발

필수 워크플로우:
1. 먼저 테스트 작성 (RED)
2. 테스트 실행 - 실패해야 함
3. 최소한의 구현 작성 (GREEN)
4. 테스트 실행 - 통과해야 함
5. 리팩터링 (IMPROVE)
6. 커버리지 확인 (80% 이상)

### 테스트 실패 문제 해결

1. **tdd-guide** agent 사용
2. 테스트 격리 확인
3. 모의 객체가 올바른지 확인
4. 테스트가 아닌 구현 수정 (테스트가 잘못된 경우 제외)

---

## Git 워크플로우

### 커밋 메시지 형식

```
<type>: <description>

<optional body>
```

타입: feat, fix, refactor, docs, test, chore, perf, ci

### Pull Request 워크플로우

PR 생성 시:
1. 최신 커밋만이 아닌 전체 커밋 히스토리 분석
2. `git diff [base-branch]...HEAD`를 사용하여 모든 변경 사항 확인
3. 포괄적인 PR 요약 작성
4. TODO가 있는 테스트 계획 포함
5. 새 브랜치인 경우 `-u` 플래그와 함께 푸시

### 기능 구현 워크플로우

1. **먼저 계획하기**
   - **planner** agent를 사용하여 구현 계획 생성
   - 의존성 및 위험 요소 파악
   - 단계별로 분해

2. **TDD 접근 방식**
   - **tdd-guide** agent 사용
   - 먼저 테스트 작성 (RED)
   - 테스트를 통과하도록 구현 (GREEN)
   - 리팩터링 (IMPROVE)
   - 80% 이상 커버리지 확인

3. **코드 검토**
   - 코드 작성 직후 **code-reviewer** agent 사용
   - 심각한 문제 및 높은 우선순위 문제 해결
   - 가능하면 중간 우선순위 문제 수정

4. **커밋 및 푸시**
   - 상세한 커밋 메시지
   - 컨벤셔널 커밋 형식 준수

---

## Agent 조율

### 사용 가능한 Agent

| Agent | 목적 | 사용 시점 |
|-------|---------|-------------|
| planner | 구현 계획 수립 | 복잡한 기능, 리팩터링 |
| architect | 시스템 설계 | 아키텍처 결정 |
| tdd-guide | 테스트 주도 개발 | 새 기능, 버그 수정 |
| code-reviewer | 코드 검토 | 코드 작성 후 |
| security-reviewer | 보안 분석 | 커밋 전 |
| build-error-resolver | 빌드 오류 수정 | 빌드 실패 시 |
| e2e-runner | E2E 테스트 | 핵심 사용자 흐름 |
| refactor-cleaner | 데드 코드 정리 | 코드 유지보수 |
| doc-updater | 문서화 | 문서 업데이트 |
| go-reviewer | Go 코드 검토 | Go 프로젝트 |
| go-build-resolver | Go 빌드 오류 | Go 빌드 실패 |
| database-reviewer | 데이터베이스 최적화 | SQL, 스키마 설계 |

### 즉시 Agent 사용

사용자 프롬프트 불필요:
1. 복잡한 기능 요청 - **planner** agent 사용
2. 코드 방금 작성/수정 - **code-reviewer** agent 사용
3. 버그 수정 또는 새 기능 - **tdd-guide** agent 사용
4. 아키텍처 결정 - **architect** agent 사용

---

## 성능 최적화

### 모델 선택 전략

**Haiku** (Sonnet 능력의 90%, 3배 비용 절감):
- 자주 호출되는 경량 agent
- 페어 프로그래밍 및 코드 생성
- 멀티 agent 시스템의 작업자 agent

**Sonnet** (최고의 코딩 모델):
- 주요 개발 작업
- 멀티 agent 워크플로우 조율
- 복잡한 코딩 작업

**Opus** (가장 깊은 추론):
- 복잡한 아키텍처 결정
- 최대 추론이 필요한 작업
- 연구 및 분석 작업

### 컨텍스트 윈도우 관리

컨텍스트 윈도우의 마지막 20%에서는 다음을 피하세요:
- 대규모 리팩터링
- 여러 파일에 걸친 기능 구현
- 복잡한 상호작용 디버깅

### 빌드 문제 해결

빌드 실패 시:
1. **build-error-resolver** agent 사용
2. 오류 메시지 분석
3. 점진적으로 수정
4. 각 수정 후 확인

---

## 공통 패턴

### API 응답 형식

```typescript
interface ApiResponse<T> {
  success: boolean
  data?: T
  error?: string
  meta?: {
    total: number
    page: number
    limit: number
  }
}
```

### 커스텀 Hook 패턴

```typescript
export function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value)

  useEffect(() => {
    const handler = setTimeout(() => setDebouncedValue(value), delay)
    return () => clearTimeout(handler)
  }, [value, delay])

  return debouncedValue
}
```

### Repository 패턴

```typescript
interface Repository<T> {
  findAll(filters?: Filters): Promise<T[]>
  findById(id: string): Promise<T | null>
  create(data: CreateDto): Promise<T>
  update(id: string, data: UpdateDto): Promise<T>
  delete(id: string): Promise<void>
}
```

---

## OpenCode 특이 사항

OpenCode는 hook을 지원하지 않으므로, Claude Code에서 자동화되었던 다음 작업은 수동으로 수행해야 합니다:

### 코드 작성/편집 후
- `prettier --write <file>`을 실행하여 JS/TS 파일 포맷
- `npx tsc --noEmit`을 실행하여 TypeScript 오류 확인
- console.log 문 확인 및 제거

### 커밋 전
- 수동으로 보안 검사 실행
- 코드에 시크릿 없는지 확인
- 전체 테스트 스위트 실행

### 사용 가능한 명령어

OpenCode에서 다음 명령어를 사용하세요:
- `/plan` - 구현 계획 생성
- `/tdd` - TDD 워크플로우 적용
- `/code-review` - 코드 변경 사항 검토
- `/security` - 보안 검토 실행
- `/build-fix` - 빌드 오류 수정
- `/e2e` - E2E 테스트 생성
- `/refactor-clean` - 데드 코드 제거
- `/orchestrate` - 멀티 agent 워크플로우

---

## 성공 기준

다음 조건을 충족하면 성공입니다:
- 모든 테스트 통과 (80% 이상 커버리지)
- 보안 취약점 없음
- 코드가 읽기 쉽고 유지보수 가능
- 성능이 수용 가능한 수준
- 사용자 요구사항 충족

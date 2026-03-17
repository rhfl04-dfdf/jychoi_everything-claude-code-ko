# SaaS 애플리케이션 — 프로젝트 CLAUDE.md

> Next.js + Supabase + Stripe SaaS 애플리케이션의 실제 예제입니다.
> 프로젝트 루트에 복사하여 스택에 맞게 커스터마이징하세요.

## 프로젝트 개요

**스택:** Next.js 15 (App Router), TypeScript, Supabase (인증 + DB), Stripe (결제), Tailwind CSS, Playwright (E2E)

**아키텍처:** 기본적으로 Server Component 사용. Client Component는 상호작용이 필요한 경우에만 사용. 웹훅에는 API 라우트, 뮤테이션에는 server action 사용.

## 핵심 규칙

### 데이터베이스

- 모든 쿼리는 RLS가 활성화된 Supabase 클라이언트 사용 — RLS 우회 금지
- 마이그레이션은 `supabase/migrations/`에 관리 — 데이터베이스 직접 수정 금지
- `select('*')` 대신 명시적 컬럼 목록과 함께 `select()` 사용
- 모든 사용자 대상 쿼리에는 무제한 결과를 방지하기 위해 `.limit()` 포함

### 인증

- Server Component에서는 `@supabase/ssr`의 `createServerClient()` 사용
- Client Component에서는 `@supabase/ssr`의 `createBrowserClient()` 사용
- 보호된 라우트는 `getUser()` 확인 — 인증에 `getSession()`만 신뢰하지 않음
- `middleware.ts`의 미들웨어가 모든 요청에서 인증 토큰 갱신

### 결제

- Stripe 웹훅 핸들러는 `app/api/webhooks/stripe/route.ts`에 위치
- 클라이언트 측 가격 데이터를 신뢰하지 않음 — 항상 서버 측에서 Stripe에서 가져옴
- 구독 상태는 웹훅이 동기화하는 `subscription_status` 컬럼으로 확인
- 무료 플랜 사용자: 프로젝트 3개, 일일 API 호출 100회

### 코드 스타일

- 코드나 주석에 이모지 사용 금지
- 불변 패턴만 사용 — 스프레드 연산자, 객체 변경 금지
- Server Component: `'use client'` 지시어, `useState`/`useEffect` 사용 금지
- Client Component: 상단에 `'use client'`, 최소화 — 로직은 hook으로 분리
- 모든 입력 검증에 Zod 스키마 선호 (API 라우트, 폼, 환경 변수)

## 파일 구조

```
src/
  app/
    (auth)/          # Auth pages (login, signup, forgot-password)
    (dashboard)/     # Protected dashboard pages
    api/
      webhooks/      # Stripe, Supabase webhooks
    layout.tsx       # Root layout with providers
  components/
    ui/              # Shadcn/ui components
    forms/           # Form components with validation
    dashboard/       # Dashboard-specific components
  hooks/             # Custom React hooks
  lib/
    supabase/        # Supabase client factories
    stripe/          # Stripe client and helpers
    utils.ts         # General utilities
  types/             # Shared TypeScript types
supabase/
  migrations/        # Database migrations
  seed.sql           # Development seed data
```

## 주요 패턴

### API 응답 형식

```typescript
type ApiResponse<T> =
  | { success: true; data: T }
  | { success: false; error: string; code?: string }
```

### Server Action 패턴

```typescript
'use server'

import { z } from 'zod'
import { createServerClient } from '@/lib/supabase/server'

const schema = z.object({
  name: z.string().min(1).max(100),
})

export async function createProject(formData: FormData) {
  const parsed = schema.safeParse({ name: formData.get('name') })
  if (!parsed.success) {
    return { success: false, error: parsed.error.flatten() }
  }

  const supabase = await createServerClient()
  const { data: { user } } = await supabase.auth.getUser()
  if (!user) return { success: false, error: 'Unauthorized' }

  const { data, error } = await supabase
    .from('projects')
    .insert({ name: parsed.data.name, user_id: user.id })
    .select('id, name, created_at')
    .single()

  if (error) return { success: false, error: 'Failed to create project' }
  return { success: true, data }
}
```

## 환경 변수

```bash
# Supabase
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=     # Server-only, never expose to client

# Stripe
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=

# App
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

## 테스트 전략

```bash
/tdd                    # Unit + integration tests for new features
/e2e                    # Playwright tests for auth flow, billing, dashboard
/test-coverage          # Verify 80%+ coverage
```

### 주요 E2E 흐름

1. 회원가입 → 이메일 인증 → 첫 프로젝트 생성
2. 로그인 → 대시보드 → CRUD 작업
3. 플랜 업그레이드 → Stripe 체크아웃 → 구독 활성화
4. 웹훅: 구독 취소 → 무료 플랜으로 다운그레이드

## ECC 워크플로우

```bash
# Planning a feature
/plan "Add team invitations with email notifications"

# Developing with TDD
/tdd

# Before committing
/code-review
/security-scan

# Before release
/e2e
/test-coverage
```

## Git 워크플로우

- `feat:` 새 기능, `fix:` 버그 수정, `refactor:` 코드 변경
- `main`에서 기능 브랜치 생성, PR 필수
- CI 실행: 린트, 타입 체크, 단위 테스트, E2E 테스트
- 배포: PR에서 Vercel 프리뷰, `main` 머지 시 프로덕션

---
description: Playwright로 E2E 테스트 생성 및 실행
agent: e2e-runner
subtask: true
---

# E2E 명령어

Playwright를 사용하여 end-to-end 테스트를 생성하고 실행합니다: $ARGUMENTS

## 작업 내용

1. **사용자 흐름 분석** - 테스트할 대상
2. **Playwright로 테스트 여정 생성**
3. **테스트 실행** 및 아티팩트 캡처
4. **결과 보고** - 스크린샷/비디오 포함

## 테스트 구조

```typescript
import { test, expect } from '@playwright/test'

test.describe('Feature: [Name]', () => {
  test.beforeEach(async ({ page }) => {
    // Setup: Navigate, authenticate, prepare state
  })

  test('should [expected behavior]', async ({ page }) => {
    // Arrange: Set up test data

    // Act: Perform user actions
    await page.click('[data-testid="button"]')
    await page.fill('[data-testid="input"]', 'value')

    // Assert: Verify results
    await expect(page.locator('[data-testid="result"]')).toBeVisible()
  })

  test.afterEach(async ({ page }, testInfo) => {
    // Capture screenshot on failure
    if (testInfo.status !== 'passed') {
      await page.screenshot({ path: `test-results/${testInfo.title}.png` })
    }
  })
})
```

## 모범 사례

### 셀렉터
- `data-testid` 속성을 우선 사용
- CSS 클래스 사용 지양 (변경될 수 있음)
- 시맨틱 셀렉터 사용 (role, label)

### 대기
- Playwright의 자동 대기 기능 활용
- `page.waitForTimeout()` 사용 지양
- assertion에는 `expect().toBeVisible()` 사용

### 테스트 격리
- 각 테스트는 독립적이어야 함
- 테스트 데이터를 사후 정리
- 테스트 순서에 의존하지 않기

## 캡처할 아티팩트

- 실패 시 스크린샷
- 디버깅용 비디오
- 상세 분석용 trace 파일
- 필요한 경우 네트워크 로그

## 테스트 카테고리

1. **핵심 사용자 흐름**
   - 인증 (로그인, 로그아웃, 회원가입)
   - 핵심 기능 정상 경로
   - 결제/체크아웃 흐름

2. **엣지 케이스**
   - 네트워크 장애
   - 잘못된 입력
   - 세션 만료

3. **크로스 브라우저**
   - Chrome, Firefox, Safari
   - 모바일 뷰포트

## 보고서 형식

```
E2E Test Results
================
✅ Passed: X
❌ Failed: Y
⏭️ Skipped: Z

Failed Tests:
- test-name: Error message
  Screenshot: path/to/screenshot.png
  Video: path/to/video.webm
```

---

**팁**: 디버깅 시 `--headed` 플래그와 함께 실행하세요: `npx playwright test --headed`

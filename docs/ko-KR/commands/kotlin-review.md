---
description: 관용적 패턴, null 안전성, coroutine 안전성, 보안을 위한 포괄적인 Kotlin 코드 리뷰. kotlin-reviewer 에이전트를 호출합니다.
---

# Kotlin Code Review

이 커맨드는 포괄적인 Kotlin 전용 코드 리뷰를 위해 **kotlin-reviewer** 에이전트를 호출합니다.

## 이 커맨드가 하는 일

1. **Kotlin 변경 사항 식별**: `git diff`를 통해 수정된 `.kt` 및 `.kts` 파일 찾기
2. **빌드 및 정적 분석 실행**: `./gradlew build`, `detekt`, `ktlintCheck` 실행
3. **보안 스캔**: SQL injection, command injection, 하드코딩된 시크릿 확인
4. **Null 안전성 검토**: `!!` 사용, platform type 처리, 안전하지 않은 캐스트 분석
5. **Coroutine 검토**: structured concurrency, dispatcher 사용, 취소 확인
6. **보고서 생성**: 심각도별로 이슈 분류

## 사용 시점

다음 경우 `/kotlin-review`를 사용합니다:
- Kotlin 코드 작성 또는 수정 후
- Kotlin 변경 사항 커밋 전
- Kotlin 코드가 있는 pull request 검토 시
- 새로운 Kotlin 코드베이스에 온보딩할 때
- 관용적 Kotlin 패턴을 배울 때

## 리뷰 범주

### CRITICAL (반드시 수정)
- SQL/Command injection 취약점
- 정당한 이유 없는 force-unwrap `!!`
- Platform type null 안전성 위반
- GlobalScope 사용 (structured concurrency 위반)
- 하드코딩된 credential
- 안전하지 않은 역직렬화

### HIGH (수정 권장)
- 불변으로 충분한 곳에서 가변 상태 사용
- coroutine 컨텍스트 내 blocking 호출
- 긴 루프에서 취소 확인 누락
- sealed type에서 비완전한 `when`
- 큰 함수 (>50줄)
- 깊은 중첩 (>4단계)

### MEDIUM (고려 사항)
- 비관용적 Kotlin (Java 스타일 패턴)
- 누락된 trailing comma
- Scope 함수 오용 또는 중첩
- 큰 컬렉션 체인에 sequence 미사용
- 불필요한 명시적 타입

## 실행되는 자동화 검사

```bash
# 빌드 확인
./gradlew build

# 정적 분석
./gradlew detekt

# 포매팅 확인
./gradlew ktlintCheck

# 테스트
./gradlew test
```

## 사용 예시

````text
User: /kotlin-review

Agent:
# Kotlin Code Review Report

## Files Reviewed
- src/main/kotlin/com/example/service/UserService.kt (modified)
- src/main/kotlin/com/example/routes/UserRoutes.kt (modified)

## Static Analysis Results
✓ Build: Successful
✓ detekt: No issues
⚠ ktlint: 2 formatting warnings

## Issues Found

[CRITICAL] Force-Unwrap Null Safety
File: src/main/kotlin/com/example/service/UserService.kt:28
Issue: Using !! on nullable repository result
```kotlin
val user = repository.findById(id)!!  // NPE risk
```
Fix: Use safe call with error handling
```kotlin
val user = repository.findById(id)
    ?: throw UserNotFoundException("User $id not found")
```

[HIGH] GlobalScope Usage
File: src/main/kotlin/com/example/routes/UserRoutes.kt:45
Issue: Using GlobalScope breaks structured concurrency
```kotlin
GlobalScope.launch {
    notificationService.sendWelcome(user)
}
```
Fix: Use the call's coroutine scope
```kotlin
launch {
    notificationService.sendWelcome(user)
}
```

## Summary
- CRITICAL: 1
- HIGH: 1
- MEDIUM: 0

Recommendation: ❌ Block merge until CRITICAL issue is fixed
````

## 승인 기준

| 상태 | 조건 |
|--------|-----------|
| ✅ 승인 | CRITICAL 또는 HIGH 이슈 없음 |
| ⚠️ 경고 | MEDIUM 이슈만 있음 (주의하여 병합) |
| ❌ 차단 | CRITICAL 또는 HIGH 이슈 발견 |

## 다른 커맨드와의 연계

- 테스트가 통과하는지 먼저 `/kotlin-test` 사용
- 빌드 에러 발생 시 `/kotlin-build` 사용
- 커밋 전 `/kotlin-review` 사용
- Kotlin 전용이 아닌 우려 사항에는 `/code-review` 사용

## 관련

- Agent: `agents/kotlin-reviewer.md`
- Skills: `skills/kotlin-patterns/`, `skills/kotlin-testing/`

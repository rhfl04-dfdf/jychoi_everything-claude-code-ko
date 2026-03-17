---
description: Kotlin에서 TDD 워크플로를 적용합니다. Kotest 테스트를 먼저 작성한 후 구현합니다. Kover로 80%+ 커버리지를 확인합니다.
---

# Kotlin TDD 커맨드

이 커맨드는 Kotest, MockK, Kover를 사용하여 Kotlin 코드에서 테스트 주도 개발 방법론을 적용합니다.

## 이 커맨드가 하는 일

1. **타입/인터페이스 정의**: 먼저 함수 시그니처 스캐폴딩
2. **Kotest 테스트 작성**: 포괄적인 테스트 spec 작성 (RED)
3. **테스트 실행**: 올바른 이유로 테스트가 실패하는지 확인
4. **코드 구현**: 통과할 최소한의 코드 작성 (GREEN)
5. **리팩토링**: 테스트가 통과된 상태에서 개선
6. **커버리지 확인**: Kover로 80%+ 커버리지 보장

## 사용 시점

다음 경우 `/kotlin-test`를 사용합니다:
- 새 Kotlin 함수나 클래스를 구현할 때
- 기존 Kotlin 코드에 테스트 커버리지 추가 시
- 버그 수정 시 (먼저 실패하는 테스트 작성)
- 중요한 비즈니스 로직 구축 시
- Kotlin에서 TDD 워크플로를 배울 때

## TDD 사이클

```
RED     -> Write failing Kotest test
GREEN   -> Implement minimal code to pass
REFACTOR -> Improve code, tests stay green
REPEAT  -> Next test case
```

## 세션 예시

````text
User: /kotlin-test I need a function to validate user registration requests

Agent:
# TDD Session: Registration Validator

## Step 1: Define Interface

```kotlin
// validator/RegistrationValidator.kt
package com.example.validator

data class RegistrationRequest(
    val name: String,
    val email: String,
    val password: String,
)

sealed class ValidationResult {
    data object Valid : ValidationResult()
    data class Invalid(val errors: List<String>) : ValidationResult()
}

fun validateRegistration(request: RegistrationRequest): ValidationResult {
    TODO("not implemented")
}
```

## Step 2: Write Kotest Tests (RED)

```kotlin
// validator/RegistrationValidatorTest.kt
package com.example.validator

import io.kotest.core.spec.style.FunSpec
import io.kotest.matchers.shouldBe
import io.kotest.matchers.types.shouldBeInstanceOf

class RegistrationValidatorTest : FunSpec({
    test("valid registration returns Valid") {
        val request = RegistrationRequest(
            name = "Alice",
            email = "alice@example.com",
            password = "SecureP@ss1",
        )

        val result = validateRegistration(request)

        result.shouldBeInstanceOf<ValidationResult.Valid>()
    }

    test("blank name returns Invalid") {
        val request = RegistrationRequest(
            name = "",
            email = "alice@example.com",
            password = "SecureP@ss1",
        )

        val result = validateRegistration(request)

        val invalid = result.shouldBeInstanceOf<ValidationResult.Invalid>()
        invalid.errors shouldBe listOf("Name is required")
    }

    test("invalid email returns Invalid") {
        val request = RegistrationRequest(
            name = "Alice",
            email = "not-an-email",
            password = "SecureP@ss1",
        )

        val result = validateRegistration(request)

        val invalid = result.shouldBeInstanceOf<ValidationResult.Invalid>()
        invalid.errors shouldBe listOf("Invalid email format")
    }

    test("short password returns Invalid") {
        val request = RegistrationRequest(
            name = "Alice",
            email = "alice@example.com",
            password = "short",
        )

        val result = validateRegistration(request)

        val invalid = result.shouldBeInstanceOf<ValidationResult.Invalid>()
        invalid.errors shouldBe listOf("Password must be at least 8 characters")
    }

    test("multiple errors returns all errors") {
        val request = RegistrationRequest(
            name = "",
            email = "bad",
            password = "short",
        )

        val result = validateRegistration(request)

        val invalid = result.shouldBeInstanceOf<ValidationResult.Invalid>()
        invalid.errors.size shouldBe 3
    }
})
```

## Step 3: Run Tests - Verify FAIL

```bash
$ ./gradlew test

RegistrationValidatorTest > valid registration returns Valid FAILED
  kotlin.NotImplementedError: An operation is not implemented

FAILED (5 tests, 0 passed, 5 failed)
```

✓ Tests fail as expected (NotImplementedError).

## Step 4: Implement Minimal Code (GREEN)

```kotlin
// validator/RegistrationValidator.kt
package com.example.validator

private val EMAIL_REGEX = Regex("^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$")
private const val MIN_PASSWORD_LENGTH = 8

fun validateRegistration(request: RegistrationRequest): ValidationResult {
    val errors = buildList {
        if (request.name.isBlank()) add("Name is required")
        if (!EMAIL_REGEX.matches(request.email)) add("Invalid email format")
        if (request.password.length < MIN_PASSWORD_LENGTH) add("Password must be at least $MIN_PASSWORD_LENGTH characters")
    }

    return if (errors.isEmpty()) ValidationResult.Valid
    else ValidationResult.Invalid(errors)
}
```

## Step 5: Run Tests - Verify PASS

```bash
$ ./gradlew test

RegistrationValidatorTest > valid registration returns Valid PASSED
RegistrationValidatorTest > blank name returns Invalid PASSED
RegistrationValidatorTest > invalid email returns Invalid PASSED
RegistrationValidatorTest > short password returns Invalid PASSED
RegistrationValidatorTest > multiple errors returns all errors PASSED

PASSED (5 tests, 5 passed, 0 failed)
```

✓ All tests passing!

## Step 6: Check Coverage

```bash
$ ./gradlew koverHtmlReport

Coverage: 100.0% of statements
```

✓ Coverage: 100%

## TDD Complete!
````

## 테스트 패턴

### StringSpec (가장 간단)

```kotlin
class CalculatorTest : StringSpec({
    "add two positive numbers" {
        Calculator.add(2, 3) shouldBe 5
    }
})
```

### BehaviorSpec (BDD)

```kotlin
class OrderServiceTest : BehaviorSpec({
    Given("a valid order") {
        When("placed") {
            Then("should be confirmed") { /* ... */ }
        }
    }
})
```

### 데이터 기반 테스트

```kotlin
class ParserTest : FunSpec({
    context("valid inputs") {
        withData("2026-01-15", "2026-12-31", "2000-01-01") { input ->
            parseDate(input).shouldNotBeNull()
        }
    }
})
```

### Coroutine 테스트

```kotlin
class AsyncServiceTest : FunSpec({
    test("concurrent fetch completes") {
        runTest {
            val result = service.fetchAll()
            result.shouldNotBeEmpty()
        }
    }
})
```

## 커버리지 커맨드

```bash
# Run tests with coverage
./gradlew koverHtmlReport

# Verify coverage thresholds
./gradlew koverVerify

# XML report for CI
./gradlew koverXmlReport

# Open HTML report
open build/reports/kover/html/index.html

# Run specific test class
./gradlew test --tests "com.example.UserServiceTest"

# Run with verbose output
./gradlew test --info
```

## 커버리지 목표

| 코드 유형 | 목표 |
|-----------|--------|
| 중요한 비즈니스 로직 | 100% |
| 공개 API | 90%+ |
| 일반 코드 | 80%+ |
| 생성된 코드 | 제외 |

## TDD 모범 사례

**해야 할 것:**
- 구현 전에 먼저 테스트 작성
- 각 변경 후 테스트 실행
- 표현력 있는 assertion을 위해 Kotest matcher 사용
- suspend 함수에 MockK의 `coEvery`/`coVerify` 사용
- 구현 세부 사항이 아닌 동작 테스트
- 엣지 케이스 포함 (빈 값, null, 최대값)

**하지 말 것:**
- 테스트 전에 구현 작성
- RED 단계 건너뛰기
- private 함수 직접 테스트
- coroutine 테스트에서 `Thread.sleep()` 사용
- 불안정한 테스트 무시

## 관련 커맨드

- `/kotlin-build` - 빌드 에러 수정
- `/kotlin-review` - 구현 후 코드 검토
- `/verify` - 전체 검증 루프 실행

## 관련

- Skill: `skills/kotlin-testing/`
- Skill: `skills/tdd-workflow/`

---
description: "공통 규칙을 확장하는 Kotlin 테스트"
globs: ["**/*.kt", "**/*.kts", "**/build.gradle.kts"]
alwaysApply: false
---
# Kotlin 테스트

> 이 파일은 공통 테스트 규칙을 Kotlin 관련 내용으로 확장합니다.

## 프레임워크

spec 스타일(StringSpec, FunSpec, BehaviorSpec)의 **Kotest**와 mocking을 위한 **MockK**를 사용합니다.

## 코루틴 테스트

`kotlinx-coroutines-test`의 `runTest`를 사용합니다:

```kotlin
test("async operation completes") {
    runTest {
        val result = service.fetchData()
        result.shouldNotBeEmpty()
    }
}
```

## 커버리지

커버리지 리포팅에는 **Kover**를 사용합니다:

```bash
./gradlew koverHtmlReport
./gradlew koverVerify
```

## 참고

상세한 Kotest 패턴, MockK 사용법, 속성 기반 테스트는 skill: `kotlin-testing`을 참조하세요.

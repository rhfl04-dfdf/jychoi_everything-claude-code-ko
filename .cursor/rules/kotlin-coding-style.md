---
description: "공통 규칙을 확장하는 Kotlin 코딩 스타일"
globs: ["**/*.kt", "**/*.kts", "**/build.gradle.kts"]
alwaysApply: false
---
# Kotlin 코딩 스타일

> 이 파일은 공통 코딩 스타일 규칙을 Kotlin 관련 내용으로 확장합니다.

## 포매팅

- **ktfmt** 또는 **ktlint**를 통한 자동 포매팅 (`kotlin-hooks.md`에서 설정)
- 여러 줄 선언에서는 후행 쉼표 사용

## 불변성

전역 불변성 요구사항은 공통 코딩 스타일 규칙에서 적용됩니다.
Kotlin에서는 구체적으로:

- `var`보다 `val` 선호
- 불변 컬렉션 타입 사용 (`List`, `Map`, `Set`)
- 불변 업데이트를 위해 `data class`의 `copy()` 사용

## Null 안전성

- `!!` 사용 금지 -- `?.`, `?:`, `require`, 또는 `checkNotNull` 사용
- Java 상호운용 경계에서 플랫폼 타입을 명시적으로 처리

## 표현식 본문

단일 표현식 함수에는 표현식 본문을 선호합니다:

```kotlin
fun isAdult(age: Int): Boolean = age >= 18
```

## 참고

포괄적인 Kotlin 관용구 및 패턴은 skill: `kotlin-patterns`을 참조하세요.

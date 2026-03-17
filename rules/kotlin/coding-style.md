---
paths:
  - "**/*.kt"
  - "**/*.kts"
---
# Kotlin 코딩 스타일

> 이 파일은 [common/coding-style.md](../common/coding-style.md)를 확장하여 Kotlin 특화 내용을 다룹니다.

## 포맷팅

- 스타일 강제를 위해 **ktlint** 또는 **Detekt** 사용
- 공식 Kotlin 코드 스타일 (`gradle.properties`에 `kotlin.code.style=official` 설정)

## 불변성

- `var`보다 `val` 선호 — 기본적으로 `val`을 사용하고, 변경이 필요한 경우에만 `var` 사용
- 값 타입에는 `data class` 사용; 공개 API에서는 불변 컬렉션(`List`, `Map`, `Set`) 사용
- 상태 업데이트 시 Copy-on-write 방식: `state.copy(field = newValue)`

## 네이밍

Kotlin 컨벤션 준수:
- 함수와 프로퍼티에는 `camelCase`
- 클래스, 인터페이스, 객체, 타입 별칭에는 `PascalCase`
- 상수(`const val` 또는 `@JvmStatic`)에는 `SCREAMING_SNAKE_CASE`
- 인터페이스 접두사는 `I`가 아닌 동작으로 표현: `IClickable`이 아닌 `Clickable`

## Null 안전성

- `!!`는 절대 사용 금지 — `?.`, `?:`, `requireNotNull()`, `checkNotNull()` 선호
- 범위 지정 null 안전 작업에는 `?.let {}` 사용
- 결과가 없을 수 있는 경우 함수에서 nullable 타입 반환

```kotlin
// BAD
val name = user!!.name

// GOOD
val name = user?.name ?: "Unknown"
val name = requireNotNull(user) { "User must be set before accessing name" }.name
```

## Sealed 타입

닫힌 상태 계층을 모델링할 때 sealed 클래스/인터페이스 사용:

```kotlin
sealed interface UiState<out T> {
    data object Loading : UiState<Nothing>
    data class Success<T>(val data: T) : UiState<T>
    data class Error(val message: String) : UiState<Nothing>
}
```

sealed 타입과 함께 항상 완전한 `when` 사용 — `else` 분기 없음.

## 확장 함수

유틸리티 작업에 확장 함수를 사용하되, 발견 가능성을 유지:
- 수신자 타입의 이름을 딴 파일에 배치 (`StringExt.kt`, `FlowExt.kt`)
- 범위를 제한 — `Any`나 지나치게 제네릭한 타입에 확장 추가 금지

## 스코프 함수

올바른 스코프 함수 사용:
- `let` — null 체크 + 변환: `user?.let { greet(it) }`
- `run` — 수신자를 사용하여 결과 계산: `service.run { fetch(config) }`
- `apply` — 객체 설정: `builder.apply { timeout = 30 }`
- `also` — 부수 효과: `result.also { log(it) }`
- 스코프 함수 깊은 중첩 금지 (최대 2단계)

## 에러 처리

- `Result<T>` 또는 커스텀 sealed 타입 사용
- throwable 코드 래핑에는 `runCatching {}` 사용
- `CancellationException`은 절대 catch 금지 — 항상 rethrow
- 제어 흐름에 `try-catch` 사용 금지

```kotlin
// BAD — using exceptions for control flow
val user = try { repository.getUser(id) } catch (e: NotFoundException) { null }

// GOOD — nullable return
val user: User? = repository.findUser(id)
```

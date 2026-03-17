---
description: "공통 규칙을 확장하는 Kotlin 패턴"
globs: ["**/*.kt", "**/*.kts", "**/build.gradle.kts"]
alwaysApply: false
---
# Kotlin 패턴

> 이 파일은 공통 패턴 규칙을 Kotlin 관련 내용으로 확장합니다.

## Sealed 클래스

완전한 타입 계층 구조를 위해 sealed 클래스/인터페이스를 사용합니다:

```kotlin
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Failure(val error: AppError) : Result<Nothing>()
}
```

## 확장 함수

상속 없이 동작을 추가하며, 사용되는 범위에 한정합니다:

```kotlin
fun String.toSlug(): String =
    lowercase().replace(Regex("[^a-z0-9\\s-]"), "").replace(Regex("\\s+"), "-")
```

## Scope 함수

- `let`: nullable 또는 범위가 지정된 결과를 변환
- `apply`: 객체를 구성
- `also`: 부수 효과 처리
- scope 함수 중첩 사용 금지

## 의존성 주입

Ktor 프로젝트에서는 Koin을 사용하여 DI를 구성합니다:

```kotlin
val appModule = module {
    single<UserRepository> { ExposedUserRepository(get()) }
    single { UserService(get()) }
}
```

## 참고

코루틴, DSL 빌더, 위임 등 포괄적인 Kotlin 패턴은 skill: `kotlin-patterns`을 참조하세요.

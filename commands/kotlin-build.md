---
description: Kotlin/Gradle 빌드 에러, 컴파일러 경고, 의존성 이슈를 단계적으로 수정합니다. 최소한의 수술적 수정을 위해 kotlin-build-resolver 에이전트를 호출합니다.
---

# Kotlin Build and Fix

이 커맨드는 **kotlin-build-resolver** 에이전트를 호출하여 최소한의 변경으로 Kotlin 빌드 에러를 단계적으로 수정합니다.

## 이 커맨드가 하는 일

1. **진단 실행**: `./gradlew build`, `detekt`, `ktlintCheck` 실행
2. **에러 파싱**: 파일별로 그룹화하고 심각도별로 정렬
3. **단계적 수정**: 한 번에 하나의 에러
4. **각 수정 확인**: 각 변경 후 빌드 재실행
5. **요약 보고**: 무엇이 수정되었고 무엇이 남아 있는지 표시

## 사용 시점

다음 경우 `/kotlin-build`를 사용합니다:
- `./gradlew build`가 에러와 함께 실패할 때
- Kotlin 컴파일러가 에러를 보고할 때
- `./gradlew detekt`가 위반을 보고할 때
- Gradle 의존성 해결이 실패할 때
- 빌드를 깨는 변경 사항을 pull한 후

## 실행되는 진단 커맨드

```bash
# Primary build check
./gradlew build 2>&1

# Static analysis
./gradlew detekt 2>&1 || echo "detekt not configured"
./gradlew ktlintCheck 2>&1 || echo "ktlint not configured"

# Dependency issues
./gradlew dependencies --configuration runtimeClasspath 2>&1 | head -100

# Optional deep refresh when caches or dependency metadata are suspect
./gradlew build --refresh-dependencies
```

## 세션 예시

````text
User: /kotlin-build

Agent:
# Kotlin Build Resolution

## Initial Diagnostics

```bash
$ ./gradlew build
e: src/main/kotlin/com/example/service/UserService.kt:25:15 Unresolved reference: UserRepository
e: src/main/kotlin/com/example/routes/UserRoutes.kt:42:9 Type mismatch: inferred type is String but Int was expected
e: src/main/kotlin/com/example/routes/UserRoutes.kt:58:5 'when' expression must be exhaustive
```

Errors found: 3

## Fix 1: Unresolved Reference

File: src/main/kotlin/com/example/service/UserService.kt:25
Error: Unresolved reference: UserRepository
Cause: Missing import

```kotlin
// Added import
import com.example.repository.UserRepository
```

```bash
$ ./gradlew build
# 2 errors remaining
```

## Fix 2: Type Mismatch

File: src/main/kotlin/com/example/routes/UserRoutes.kt:42
Error: Type mismatch: inferred type is String but Int was expected

```kotlin
// Changed
val count = call.parameters["count"]
// To
val count = call.parameters["count"]?.toIntOrNull()
    ?: return@get call.respond(HttpStatusCode.BadRequest, "Invalid count")
```

```bash
$ ./gradlew build
# 1 error remaining
```

## Fix 3: Non-Exhaustive When

File: src/main/kotlin/com/example/routes/UserRoutes.kt:58
Error: 'when' expression must be exhaustive

```kotlin
// Added missing branch
when (user.role) {
    Role.ADMIN -> handleAdmin(user)
    Role.USER -> handleUser(user)
    Role.MODERATOR -> handleModerator(user) // Added
}
```

```bash
$ ./gradlew build
# Build successful!
```

## Final Verification

```bash
$ ./gradlew detekt
# No issues

$ ./gradlew test
# All tests passed
```

## Summary

| Metric | Count |
|--------|-------|
| Build errors fixed | 3 |
| Detekt issues fixed | 0 |
| Files modified | 2 |
| Remaining issues | 0 |

Build Status: ✅ SUCCESS
````

## 수정되는 일반적인 에러

| 에러 | 일반적인 수정 |
|-------|-------------|
| `Unresolved reference: X` | import 또는 의존성 추가 |
| `Type mismatch` | 타입 변환 또는 할당 수정 |
| `'when' must be exhaustive` | 누락된 sealed class 브랜치 추가 |
| `Suspend function can only be called from coroutine` | `suspend` 수정자 추가 |
| `Smart cast impossible` | 로컬 `val` 또는 `let` 사용 |
| `None of the following candidates is applicable` | 인수 타입 수정 |
| `Could not resolve dependency` | 버전 수정 또는 저장소 추가 |

## 수정 전략

1. **빌드 에러 우선** - 코드가 컴파일되어야 함
2. **Detekt 위반 두 번째** - 코드 품질 이슈 수정
3. **ktlint 경고 세 번째** - 포매팅 수정
4. **한 번에 하나씩 수정** - 각 변경 사항 확인
5. **최소한의 변경** - 리팩토링하지 말고 수정만

## 중단 조건

에이전트는 다음 경우 중단하고 보고합니다:
- 3번 시도 후에도 동일한 에러가 지속될 때
- 수정이 더 많은 에러를 도입할 때
- 아키텍처 변경이 필요할 때
- 외부 의존성이 누락되었을 때

## 관련 커맨드

- `/kotlin-test` - 빌드 성공 후 테스트 실행
- `/kotlin-review` - 코드 품질 검토
- `/verify` - 전체 검증 루프

## 관련

- Agent: `agents/kotlin-build-resolver.md`
- Skill: `skills/kotlin-patterns/`

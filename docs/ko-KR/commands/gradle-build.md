---
description: Android 및 KMP 프로젝트의 Gradle 빌드 에러를 수정합니다
---

# Gradle Build Fix

Android 및 Kotlin Multiplatform 프로젝트의 Gradle 빌드 및 컴파일 에러를 단계적으로 수정합니다.

## 1단계: 빌드 설정 감지

프로젝트 유형을 식별하고 적절한 빌드를 실행합니다:

| 지표 | 빌드 커맨드 |
|-----------|---------------|
| `build.gradle.kts` + `composeApp/` (KMP) | `./gradlew composeApp:compileKotlinMetadata 2>&1` |
| `build.gradle.kts` + `app/` (Android) | `./gradlew app:compileDebugKotlin 2>&1` |
| 모듈이 있는 `settings.gradle.kts` | `./gradlew assemble 2>&1` |
| Detekt 설정됨 | `./gradlew detekt 2>&1` |

설정을 위해 `gradle.properties`와 `local.properties`도 확인합니다.

## 2단계: 에러 파싱 및 그룹화

1. 빌드 커맨드를 실행하고 출력을 캡처합니다
2. Kotlin 컴파일 에러와 Gradle 설정 에러를 분리합니다
3. 모듈과 파일 경로별로 그룹화합니다
4. 정렬: 설정 에러 먼저, 그 다음 의존성 순서에 따른 컴파일 에러

## 3단계: 수정 루프

각 에러에 대해:

1. **파일 읽기** — 에러 줄 주변의 전체 컨텍스트
2. **진단** — 일반적인 범주:
   - 누락된 import 또는 확인되지 않은 참조
   - 타입 불일치 또는 호환되지 않는 타입
   - `build.gradle.kts`에 누락된 의존성
   - Expect/actual 불일치 (KMP)
   - Compose 컴파일러 에러
3. **최소한으로 수정** — 에러를 해결하는 가장 작은 변경
4. **빌드 재실행** — 수정을 확인하고 새 에러 점검
5. **계속** — 다음 에러로 이동

## 4단계: 가드레일

다음 경우 중단하고 사용자에게 묻습니다:
- 수정이 해결하는 것보다 더 많은 에러를 도입할 때
- 3번 시도 후에도 동일한 에러가 지속될 때
- 에러가 새 의존성 추가나 모듈 구조 변경을 필요로 할 때
- Gradle sync 자체가 실패할 때 (설정 단계 에러)
- 에러가 생성된 코드에 있을 때 (Room, SQLDelight, KSP)

## 5단계: 요약

보고:
- 수정된 에러 (모듈, 파일, 설명)
- 남은 에러
- 새로 도입된 에러 (0이어야 함)
- 제안된 다음 단계

## 일반적인 Gradle/KMP 수정

| 에러 | 수정 |
|-------|-----|
| `commonMain`에서 확인되지 않은 참조 | 의존성이 `commonMain.dependencies {}`에 있는지 확인 |
| actual 없는 expect 선언 | 각 플랫폼 소스 셋에 `actual` 구현 추가 |
| Compose 컴파일러 버전 불일치 | `libs.versions.toml`에서 Kotlin과 Compose 컴파일러 버전 맞추기 |
| 중복 클래스 | `./gradlew dependencies`로 충돌하는 의존성 확인 |
| KSP 에러 | `./gradlew kspCommonMainKotlinMetadata`로 재생성 |
| 설정 캐시 이슈 | 직렬화할 수 없는 태스크 입력 확인 |

---
paths:
  - "**/*.kt"
  - "**/*.kts"
---
# Kotlin 보안

> 이 파일은 [common/security.md](../common/security.md)를 확장하여 Kotlin 및 Android/KMP 특화 내용을 다룹니다.

## 시크릿 관리

- 소스 코드에 API 키, 토큰, 자격증명을 절대 하드코딩하지 않기
- 로컬 개발 시크릿에는 `local.properties` (git에서 제외) 사용
- 릴리스 빌드에는 CI 시크릿으로 생성된 `BuildConfig` 필드 사용
- 런타임 시크릿 저장에는 `EncryptedSharedPreferences` (Android) 또는 Keychain (iOS) 사용

```kotlin
// 나쁜 예
val apiKey = "sk-abc123..."

// 좋은 예 — BuildConfig에서 (빌드 시 생성)
val apiKey = BuildConfig.API_KEY

// 좋은 예 — 런타임 시 보안 저장소에서
val token = secureStorage.get("auth_token")
```

## 네트워크 보안

- HTTPS만 사용 — cleartext를 차단하도록 `network_security_config.xml` 설정
- 민감한 엔드포인트에는 OkHttp `CertificatePinner` 또는 Ktor 동등 도구로 인증서 피닝
- 모든 HTTP 클라이언트에 타임아웃 설정 — 기본값 사용 금지 (무한일 수 있음)
- 서버 응답을 사용 전에 검증하고 정제

```xml
<!-- res/xml/network_security_config.xml -->
<network-security-config>
    <base-config cleartextTrafficPermitted="false" />
</network-security-config>
```

## 입력 검증

- 처리 또는 API 전송 전에 모든 사용자 입력 검증
- Room/SQLDelight에는 매개변수화된 쿼리 사용 — 사용자 입력을 SQL에 직접 연결 금지
- 경로 탐색 방지를 위해 사용자 입력의 파일 경로를 정제

```kotlin
// 나쁜 예 — SQL 인젝션
@Query("SELECT * FROM items WHERE name = '$input'")

// 좋은 예 — 매개변수화
@Query("SELECT * FROM items WHERE name = :input")
fun findByName(input: String): List<ItemEntity>
```

## 데이터 보호

- Android에서 민감한 키-값 데이터에 `EncryptedSharedPreferences` 사용
- 명시적 필드 이름과 함께 `@Serializable` 사용 — 내부 프로퍼티 이름 노출 금지
- 더 이상 필요하지 않을 때 메모리에서 민감한 데이터 삭제
- 이름 변환(name mangling) 방지를 위해 직렬화 클래스에 `@Keep` 또는 ProGuard 규칙 사용

## 인증

- 일반 SharedPreferences가 아닌 보안 저장소에 토큰 저장
- 적절한 401/403 처리와 함께 토큰 갱신 구현
- 로그아웃 시 모든 인증 상태 삭제 (토큰, 캐시된 사용자 데이터, 쿠키)
- 민감한 작업에 생체 인증(`BiometricPrompt`) 사용

## ProGuard / R8

- 모든 직렬화 모델의 유지 규칙 (`@Serializable`, Gson, Moshi)
- 리플렉션 기반 라이브러리의 유지 규칙 (Koin, Retrofit)
- 릴리스 빌드 테스트 — 난독화가 직렬화를 자동으로 깨뜨릴 수 있음

## WebView 보안

- 명시적으로 필요한 경우가 아니면 JavaScript 비활성화: `settings.javaScriptEnabled = false`
- WebView에서 로드하기 전에 URL 검증
- 민감한 데이터에 접근하는 `@JavascriptInterface` 메서드 절대 노출 금지
- 네비게이션 제어를 위해 `WebViewClient.shouldOverrideUrlLoading()` 사용

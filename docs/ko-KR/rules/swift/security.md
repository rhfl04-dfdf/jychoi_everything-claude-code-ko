---
paths:
  - "**/*.swift"
  - "**/Package.swift"
---
# Swift 보안

> 이 파일은 [common/security.md](../common/security.md)를 확장하여 Swift 특화 내용을 다룹니다.

## 시크릿 관리

- 민감한 데이터(토큰, 비밀번호, 키)에는 **Keychain Services** 사용 — `UserDefaults` 절대 금지
- 빌드 타임 시크릿에는 환경 변수 또는 `.xcconfig` 파일 사용
- 소스에 시크릿 하드코딩 절대 금지 — 디컴파일 도구로 쉽게 추출 가능

```swift
let apiKey = ProcessInfo.processInfo.environment["API_KEY"]
guard let apiKey, !apiKey.isEmpty else {
    fatalError("API_KEY not configured")
}
```

## 전송 보안

- App Transport Security (ATS)는 기본적으로 적용 — 비활성화 금지
- 중요한 엔드포인트에는 인증서 피닝 사용
- 모든 서버 인증서 검증

## 입력 검증

- 인젝션 방지를 위해 표시 전에 모든 사용자 입력 정제
- 강제 언래핑 대신 검증과 함께 `URL(string:)` 사용
- 처리 전에 외부 소스(API, 딥링크, 클립보드)의 데이터 검증

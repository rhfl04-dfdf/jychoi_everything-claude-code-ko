---
paths:
  - "**/*.swift"
  - "**/Package.swift"
---
# Swift 테스팅

> 이 파일은 [common/testing.md](../common/testing.md)를 Swift 관련 내용으로 확장합니다.

## 프레임워크

새 테스트에는 **Swift Testing** (`import Testing`)을 사용하세요. `@Test`와 `#expect`를 사용합니다:

```swift
@Test("User creation validates email")
func userCreationValidatesEmail() throws {
    #expect(throws: ValidationError.invalidEmail) {
        try User(email: "not-an-email")
    }
}
```

## 테스트 격리

각 테스트는 새로운 인스턴스를 받습니다 — `init`에서 설정하고, `deinit`에서 해제합니다. 테스트 간에 변경 가능한 상태를 공유하지 마세요.

## 매개변수화 테스트

```swift
@Test("Validates formats", arguments: ["json", "xml", "csv"])
func validatesFormat(format: String) throws {
    let parser = try Parser(format: format)
    #expect(parser.isValid)
}
```

## 커버리지

```bash
swift test --enable-code-coverage
```

## 참고 자료

Swift Testing을 활용한 프로토콜 기반 의존성 주입 및 mock 패턴은 skill: `swift-protocol-di-testing`을 참조하세요.

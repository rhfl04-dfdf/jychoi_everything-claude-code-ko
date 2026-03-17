---
description: "공통 규칙을 확장하는 Swift 테스트"
globs: ["**/*.swift", "**/Package.swift"]
alwaysApply: false
---
# Swift 테스트

> 이 파일은 공통 테스트 규칙을 Swift 관련 내용으로 확장합니다.

## 프레임워크

새로운 테스트에는 **Swift Testing** (`import Testing`)을 사용합니다. `@Test`와 `#expect`를 사용합니다:

```swift
@Test("User creation validates email")
func userCreationValidatesEmail() throws {
    #expect(throws: ValidationError.invalidEmail) {
        try User(email: "not-an-email")
    }
}
```

## 테스트 격리

각 테스트는 새로운 인스턴스를 생성합니다 -- `init`에서 설정하고, `deinit`에서 해제합니다. 테스트 간 공유 가변 상태가 없어야 합니다.

## 매개변수화된 테스트

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

## 참고

Swift Testing을 사용한 프로토콜 기반 의존성 주입 및 mock 패턴은 skill: `swift-protocol-di-testing`을 참조하세요.

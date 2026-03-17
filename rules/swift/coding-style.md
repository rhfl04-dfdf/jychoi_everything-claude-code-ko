---
paths:
  - "**/*.swift"
  - "**/Package.swift"
---
# Swift 코딩 스타일

> 이 파일은 [common/coding-style.md](../common/coding-style.md)를 확장하여 Swift 특화 내용을 다룹니다.

## 포맷팅

- 자동 포맷팅에는 **SwiftFormat**, 스타일 강제에는 **SwiftLint**
- `swift-format`은 대안으로 Xcode 16+에 번들 포함

## 불변성

- `var`보다 `let` 선호 — 모든 것을 `let`으로 정의하고 컴파일러가 요구할 때만 `var`로 변경
- 기본적으로 값 시맨틱이 있는 `struct` 사용; 식별성 또는 참조 시맨틱이 필요할 때만 `class` 사용

## 네이밍

[Apple API 설계 가이드라인](https://www.swift.org/documentation/api-design-guidelines/) 준수:

- 사용 시점에서의 명확성 — 불필요한 단어 제거
- 타입이 아닌 역할에 따라 메서드 및 프로퍼티 이름 지정
- 전역 상수보다 `static let` 상수 사용

## 에러 처리

타입이 지정된 throws(Swift 6+)와 패턴 매칭 사용:

```swift
func load(id: String) throws(LoadError) -> Item {
    guard let data = try? read(from: path) else {
        throw .fileNotFound(id)
    }
    return try decode(data)
}
```

## 동시성

Swift 6 엄격한 동시성 검사 활성화. 다음을 선호:

- 격리 경계를 넘는 데이터에는 `Sendable` 값 타입
- 공유 가변 상태에는 Actor
- 구조화되지 않은 `Task {}` 대신 구조화된 동시성 (`async let`, `TaskGroup`)

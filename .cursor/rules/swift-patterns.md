---
paths:
  - "**/*.swift"
  - "**/Package.swift"
---
# Swift 패턴

> 이 파일은 [common/patterns.md](../common/patterns.md)를 확장하여 Swift 특화 내용을 다룹니다.

## 프로토콜 지향 설계

작고 집중된 프로토콜을 정의합니다. 공유 기본값에는 프로토콜 확장 사용:

```swift
protocol Repository: Sendable {
    associatedtype Item: Identifiable & Sendable
    func find(by id: Item.ID) async throws -> Item?
    func save(_ item: Item) async throws
}
```

## 값 타입

- 데이터 전송 객체와 모델에는 struct 사용
- 연관 값이 있는 enum으로 별개의 상태 모델링:

```swift
enum LoadState<T: Sendable>: Sendable {
    case idle
    case loading
    case loaded(T)
    case failed(Error)
}
```

## Actor 패턴

락이나 dispatch queue 대신 공유 가변 상태에 actor 사용:

```swift
actor Cache<Key: Hashable & Sendable, Value: Sendable> {
    private var storage: [Key: Value] = [:]

    func get(_ key: Key) -> Value? { storage[key] }
    func set(_ key: Key, value: Value) { storage[key] = value }
}
```

## 의존성 주입

기본 파라미터로 프로토콜 주입 — 프로덕션은 기본값 사용, 테스트는 mock 주입:

```swift
struct UserService {
    private let repository: any UserRepository

    init(repository: any UserRepository = DefaultUserRepository()) {
        self.repository = repository
    }
}
```

## 참고

actor 기반 영속성 패턴은 skill: `swift-actor-persistence`를 참조하세요.
프로토콜 기반 DI 및 테스팅은 skill: `swift-protocol-di-testing`을 참조하세요.

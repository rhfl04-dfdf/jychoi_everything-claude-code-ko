---
name: swift-protocol-di-testing
description: 테스트 가능한 Swift 코드를 위한 Protocol 기반 dependency injection — 집중된 protocol 및 Swift Testing을 사용하여 file system, network 및 외부 API를 mock으로 만듭니다.
origin: ECC
---

# 테스트를 위한 Swift Protocol 기반 Dependency Injection

외부 종속성(file system, network, iCloud)을 작고 집중된 protocol 뒤로 추상화하여 Swift 코드를 테스트 가능하게 만드는 패턴입니다. I/O 없이 결정론적인 테스트를 가능하게 합니다.

## 활성화 시점

- file system, network 또는 외부 API에 액세스하는 Swift 코드를 작성할 때
- 실제 실패를 유발하지 않고 error handling 경로를 테스트해야 할 때
- 다양한 환경(app, test, SwiftUI preview)에서 작동하는 모듈을 빌드할 때
- Swift concurrency(actors, Sendable)를 사용하여 테스트 가능한 아키텍처를 설계할 때

## 핵심 패턴

### 1. 작고 집중된 Protocol 정의

각 protocol은 정확히 하나의 외부 관심사만 처리합니다.

```swift
// File system access
public protocol FileSystemProviding: Sendable {
    func containerURL(for purpose: Purpose) -> URL?
}

// File read/write operations
public protocol FileAccessorProviding: Sendable {
    func read(from url: URL) throws -> Data
    func write(_ data: Data, to url: URL) throws
    func fileExists(at url: URL) -> Bool
}

// Bookmark storage (e.g., for sandboxed apps)
public protocol BookmarkStorageProviding: Sendable {
    func saveBookmark(_ data: Data, for key: String) throws
    func loadBookmark(for key: String) throws -> Data?
}
```

### 2. 기본(Production) 구현체 생성

```swift
public struct DefaultFileSystemProvider: FileSystemProviding {
    public init() {}

    public func containerURL(for purpose: Purpose) -> URL? {
        FileManager.default.url(forUbiquityContainerIdentifier: nil)
    }
}

public struct DefaultFileAccessor: FileAccessorProviding {
    public init() {}

    public func read(from url: URL) throws -> Data {
        try Data(contentsOf: url)
    }

    public func write(_ data: Data, to url: URL) throws {
        try data.write(to: url, options: .atomic)
    }

    public func fileExists(at url: URL) -> Bool {
        FileManager.default.fileExists(atPath: url.path)
    }
}
```

### 3. 테스트를 위한 Mock 구현체 생성

```swift
public final class MockFileAccessor: FileAccessorProviding, @unchecked Sendable {
    public var files: [URL: Data] = [:]
    public var readError: Error?
    public var writeError: Error?

    public init() {}

    public func read(from url: URL) throws -> Data {
        if let error = readError { throw error }
        guard let data = files[url] else {
            throw CocoaError(.fileReadNoSuchFile)
        }
        return data
    }

    public func write(_ data: Data, to url: URL) throws {
        if let error = writeError { throw error }
        files[url] = data
    }

    public func fileExists(at url: URL) -> Bool {
        files[url] != nil
    }
}
```

### 4. 기본 파라미터를 사용한 Dependency Injection

Production 코드는 기본값을 사용하고, 테스트에서는 mock을 주입합니다.

```swift
public actor SyncManager {
    private let fileSystem: FileSystemProviding
    private let fileAccessor: FileAccessorProviding

    public init(
        fileSystem: FileSystemProviding = DefaultFileSystemProvider(),
        fileAccessor: FileAccessorProviding = DefaultFileAccessor()
    ) {
        self.fileSystem = fileSystem
        self.fileAccessor = fileAccessor
    }

    public func sync() async throws {
        guard let containerURL = fileSystem.containerURL(for: .sync) else {
            throw SyncError.containerNotAvailable
        }
        let data = try fileAccessor.read(
            from: containerURL.appendingPathComponent("data.json")
        )
        // Process data...
    }
}
```

### 5. Swift Testing으로 테스트 작성

```swift
import Testing

 @everything-claude-code/docs/ko-KR/rules/typescript/testing.md("Sync manager handles missing container")
func testMissingContainer() async {
    let mockFileSystem = MockFileSystemProvider(containerURL: nil)
    let manager = SyncManager(fileSystem: mockFileSystem)

    await #expect(throws: SyncError.containerNotAvailable) {
        try await manager.sync()
    }
}

 @everything-claude-code/docs/ko-KR/rules/typescript/testing.md("Sync manager reads data correctly")
func testReadData() async throws {
    let mockFileAccessor = MockFileAccessor()
    mockFileAccessor.files[testURL] = testData

    let manager = SyncManager(fileAccessor: mockFileAccessor)
    let result = try await manager.loadData()

    #expect(result == expectedData)
}

 @everything-claude-code/docs/ko-KR/rules/typescript/testing.md("Sync manager handles read errors gracefully")
func testReadError() async {
    let mockFileAccessor = MockFileAccessor()
    mockFileAccessor.readError = CocoaError(.fileReadCorruptFile)

    let manager = SyncManager(fileAccessor: mockFileAccessor)

    await #expect(throws: SyncError.self) {
        try await manager.sync()
    }
}
```

## Best Practices

- **Single Responsibility**: 각 protocol은 하나의 관심사만 처리해야 합니다 — 많은 메서드를 가진 "god protocols"를 만들지 마세요.
- **Sendable conformance**: protocol이 actor 경계를 넘어 사용될 때 필요합니다.
- **Default parameters**: Production 코드가 기본적으로 실제 구현을 사용하도록 하세요. 테스트에서만 mock을 지정하면 됩니다.
- **Error simulation**: 실패 경로 테스트를 위해 구성 가능한 error 프로퍼티를 가진 mock을 설계하세요.
- **Only mock boundaries**: 내부 타입이 아닌 외부 종속성(file system, network, API)만 mock으로 만드세요.

## 피해야 할 Anti-Patterns

- 모든 외부 액세스를 다루는 하나의 거대한 protocol 생성
- 외부 종속성이 없는 내부 타입의 mock 생성
- 적절한 dependency injection 대신 `#if DEBUG` 조건부 사용
- actor와 함께 사용할 때 `Sendable` 준수를 잊음
- 오버 엔지니어링: 타입에 외부 종속성이 없다면 protocol이 필요하지 않습니다.

## 사용 시기

- file system, network 또는 외부 API를 다루는 모든 Swift 코드
- 실제 환경에서 트리거하기 어려운 error handling 경로 테스트
- app, test 및 SwiftUI preview 컨텍스트에서 작동해야 하는 모듈 빌드
- 테스트 가능한 아키텍처가 필요한 Swift concurrency(actors, structured concurrency) 사용 앱
--- 참조된 파일의 내용 ---
@everything-claude-code/docs/ko-KR/rules/typescript/testing.md의 내용:
---
paths:
  - "**/*.ts"
  - "**/*.tsx"
  - "**/*.js"
  - "**/*.jsx"
---
# TypeScript/JavaScript Testing

> 이 파일은 [common/testing.md](../common/testing.md)을 TypeScript/JavaScript 전용 내용으로 확장합니다.

## E2E Testing

주요 user flows를 위한 E2E testing framework로 **Playwright**를 사용하세요.

## Agent Support

- **e2e-runner** - Playwright E2E testing specialist
--- 내용 끝 ---

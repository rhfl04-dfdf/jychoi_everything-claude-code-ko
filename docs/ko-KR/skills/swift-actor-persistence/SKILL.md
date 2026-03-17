---
name: swift-actor-persistence
description: Swift actor를 사용한 Thread-safe 데이터 persistence — 설계상 data race를 제거하는 file-backed storage 기반의 in-memory cache.
origin: ECC
---

# Thread-Safe Persistence를 위한 Swift Actor

Swift actor를 사용하여 thread-safe한 데이터 persistence 레이어를 구축하는 패턴입니다. Actor model을 활용하여 컴파일 타임에 data race를 제거하며, in-memory caching과 file-backed storage를 결합합니다.

## 활성화 시점

- Swift 5.5+에서 데이터 persistence 레이어를 구축할 때
- 공유된 mutable state에 대해 thread-safe한 접근이 필요할 때
- 수동 synchronization(lock, DispatchQueue)을 제거하고 싶을 때
- 로컬 스토리지를 사용하는 offline-first 앱을 구축할 때

## 핵심 패턴

### Actor 기반 Repository

Actor model은 serialized access를 보장하여 컴파일러에 의해 data race가 발생하지 않도록 강제합니다.

```swift
public actor LocalRepository<T: Codable & Identifiable> where T.ID == String {
    private var cache: [String: T] = [:]
    private let fileURL: URL

    public init(directory: URL = .documentsDirectory, filename: String = "data.json") {
        self.fileURL = directory.appendingPathComponent(filename)
        // Synchronous load during init (actor isolation not yet active)
        self.cache = Self.loadSynchronously(from: fileURL)
    }

    // MARK: - Public API

    public func save(_ item: T) throws {
        cache[item.id] = item
        try persistToFile()
    }

    public func delete(_ id: String) throws {
        cache[id] = nil
        try persistToFile()
    }

    public func find(by id: String) -> T? {
        cache[id]
    }

    public func loadAll() -> [T] {
        Array(cache.values)
    }

    // MARK: - Private

    private func persistToFile() throws {
        let data = try JSONEncoder().encode(Array(cache.values))
        try data.write(to: fileURL, options: .atomic)
    }

    private static func loadSynchronously(from url: URL) -> [String: T] {
        guard let data = try? Data(contentsOf: url),
              let items = try? JSONDecoder().decode([T].self, from: data) else {
            return [:]
        }
        return Dictionary(uniqueKeysWithValues: items.map { ($0.id, $0) })
    }
}
```

### 사용법

Actor isolation으로 인해 모든 호출은 자동으로 async가 됩니다:

```swift
let repository = LocalRepository<Question>()

// Read — fast O(1) lookup from in-memory cache
let question = await repository.find(by: "q-001")
let allQuestions = await repository.loadAll()

// Write — updates cache and persists to file atomically
try await repository.save(newQuestion)
try await repository.delete("q-001")
```

### @Observable ViewModel과 결합

```swift
@Observable
final class QuestionListViewModel {
    private(set) var questions: [Question] = []
    private let repository: LocalRepository<Question>

    init(repository: LocalRepository<Question> = LocalRepository()) {
        self.repository = repository
    }

    func load() async {
        questions = await repository.loadAll()
    }

    func add(_ question: Question) async throws {
        try await repository.save(question)
        questions = await repository.loadAll()
    }
}
```

## 주요 설계 결정

| 결정 | 근거 |
|----------|-----------|
| Actor (class + lock 대신) | 컴파일러가 강제하는 thread safety, 수동 synchronization 불필요 |
| In-memory cache + file persistence | 캐시로부터의 빠른 읽기, 디스크로의 안정적인 쓰기 |
| Synchronous init 로딩 | Async 초기화의 복잡성 방지 |
| ID를 키로 하는 Dictionary | identifier를 통한 O(1) 조회 |
| Codable & Identifiable 기반 Generic | 모든 모델 타입에서 재사용 가능 |
| Atomic file 쓰기 (.atomic) | 크래시 발생 시 불완전한 쓰기 방지 |

## Best Practice

- Actor 경계를 넘나드는 모든 데이터에 **`Sendable` 타입을 사용**하세요.
- **Actor의 public API를 최소화**하세요 — persistence 세부 사항이 아닌 도메인 작업만 노출하세요.
- 쓰기 도중 앱이 크래시될 때 데이터 손상을 방지하기 위해 **`.atomic` 쓰기를 사용**하세요.
- **`init`에서 동기식으로 로드**하세요 — 로컬 파일의 경우 async initializer는 이점에 비해 복잡성만 더합니다.
- 반응형 UI 업데이트를 위해 **`@Observable` ViewModel과 결합**하세요.

## 피해야 할 Anti-Pattern

- 새로운 Swift concurrency 코드에서 actor 대신 `DispatchQueue` 또는 `NSLock` 사용
- 내부 캐시 dictionary를 외부 호출자에게 노출
- 검증 없이 파일 URL을 구성 가능하게 만듦
- 모든 actor 메서드 호출이 `await`임을 잊는 것 — 호출자는 반드시 async context를 처리해야 함
- Actor isolation을 우회하기 위해 `nonisolated` 사용 (목적에 어긋남)

## 사용 사례

- iOS/macOS 앱의 로컬 데이터 저장 (사용자 데이터, 설정, 캐시된 콘텐츠)
- 나중에 서버와 동기화하는 offline-first 아키텍처
- 앱의 여러 부분이 동시에 액세스하는 모든 공유 mutable state
- 기존의 `DispatchQueue` 기반 thread safety를 현대적인 Swift concurrency로 교체

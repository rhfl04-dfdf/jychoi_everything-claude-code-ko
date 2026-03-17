---
name: swift-concurrency-6-2
description: Swift 6.2의 접근 가능한 Concurrency — 기본적으로 싱글 스레드로 동작하며, 명시적인 백그라운드 오프로딩을 위한 @concurrent, main actor 타입을 위한 isolated conformances를 제공합니다.
---

# Swift 6.2 Approachable Concurrency

코드가 기본적으로 싱글 스레드에서 실행되고 concurrency가 명시적으로 도입되는 Swift 6.2의 concurrency 모델을 채택하기 위한 패턴입니다. 성능 저하 없이 일반적인 data-race 에러를 제거합니다.

## 활성화 시점

- Swift 5.x 또는 6.0/6.1 프로젝트를 Swift 6.2로 마이그레이션할 때
- data-race safety 컴파일러 에러를 해결할 때
- MainActor 기반 앱 아키텍처를 설계할 때
- CPU 집약적인 작업을 백그라운드 스레드로 오프로딩할 때
- MainActor-isolated 타입에서 프로토콜 conformances를 구현할 때
- Xcode 26에서 Approachable Concurrency 빌드 설정을 활성화할 때

## 핵심 문제: 암시적 백그라운드 오프로딩

Swift 6.1 및 이전 버전에서는 async 함수가 암시적으로 백그라운드 스레드로 오프로딩되어, 겉보기에 안전해 보이는 코드에서도 data-race 에러가 발생할 수 있었습니다:

```swift
// Swift 6.1: ERROR
@MainActor
final class StickerModel {
    let photoProcessor = PhotoProcessor()

    func extractSticker(_ item: PhotosPickerItem) async throws -> Sticker? {
        guard let data = try await item.loadTransferable(type: Data.self) else { return nil }

        // Error: Sending 'self.photoProcessor' risks causing data races
        return await photoProcessor.extractSticker(data: data, with: item.itemIdentifier)
    }
}
```

Swift 6.2는 이를 해결합니다: async 함수는 기본적으로 호출한 actor에 머무릅니다.

```swift
// Swift 6.2: OK — async stays on MainActor, no data race
@MainActor
final class StickerModel {
    let photoProcessor = PhotoProcessor()

    func extractSticker(_ item: PhotosPickerItem) async throws -> Sticker? {
        guard let data = try await item.loadTransferable(type: Data.self) else { return nil }
        return await photoProcessor.extractSticker(data: data, with: item.itemIdentifier)
    }
}
```

## 핵심 패턴 — Isolated Conformances

이제 MainActor 타입이 non-isolated 프로토콜을 안전하게 준수(conform)할 수 있습니다:

```swift
protocol Exportable {
    func export()
}

// Swift 6.1: ERROR — crosses into main actor-isolated code
// Swift 6.2: OK with isolated conformance
extension StickerModel: @MainActor Exportable {
    func export() {
        photoProcessor.exportAsPNG()
    }
}
```

컴파일러는 conformance가 오직 main actor에서만 사용되도록 보장합니다:

```swift
// OK — ImageExporter is also @MainActor
@MainActor
struct ImageExporter {
    var items: [any Exportable]

    mutating func add(_ item: StickerModel) {
        items.append(item)  // Safe: same actor isolation
    }
}

// ERROR — nonisolated context can't use MainActor conformance
nonisolated struct ImageExporter {
    var items: [any Exportable]

    mutating func add(_ item: StickerModel) {
        items.append(item)  // Error: Main actor-isolated conformance cannot be used here
    }
}
```

## 핵심 패턴 — 전역 및 정적 변수

MainActor로 전역/정적 상태를 보호하세요:

```swift
// Swift 6.1: ERROR — non-Sendable type may have shared mutable state
final class StickerLibrary {
    static let shared: StickerLibrary = .init()  // Error
}

// Fix: Annotate with @MainActor
@MainActor
final class StickerLibrary {
    static let shared: StickerLibrary = .init()  // OK
}
```

### MainActor 기본 추론 모드

Swift 6.2는 MainActor가 기본적으로 추론되는 모드를 도입하여 수동 어노테이션이 필요하지 않습니다:

```swift
// With MainActor default inference enabled:
final class StickerLibrary {
    static let shared: StickerLibrary = .init()  // Implicitly @MainActor
}

final class StickerModel {
    let photoProcessor: PhotoProcessor
    var selection: [PhotosPickerItem]  // Implicitly @MainActor
}

extension StickerModel: Exportable {  // Implicitly @MainActor conformance
    func export() {
        photoProcessor.exportAsPNG()
    }
}
```

이 모드는 선택 사항(opt-in)이며 앱, 스크립트 및 기타 실행 가능한 타겟에 권장됩니다.

## 핵심 패턴 — 백그라운드 작업을 위한 @concurrent

실제 병렬 처리가 필요한 경우, `@concurrent`를 사용하여 명시적으로 오프로딩하세요:

> **중요:** 이 예제는 Approachable Concurrency 빌드 설정(SE-0466 (MainActor 기본 isolation) 및 SE-0461 (NonisolatedNonsendingByDefault))이 필요합니다. 이 설정이 활성화되면 `extractSticker`는 호출자의 actor에 머물러 가변 상태 접근이 안전해집니다. **이러한 설정이 없으면 이 코드는 data race가 발생하며**, 컴파일러가 이를 표시합니다.

```swift
nonisolated final class PhotoProcessor {
    private var cachedStickers: [String: Sticker] = [:]

    func extractSticker(data: Data, with id: String) async -> Sticker {
        if let sticker = cachedStickers[id] {
            return sticker
        }

        let sticker = await Self.extractSubject(from: data)
        cachedStickers[id] = sticker
        return sticker
    }

    // Offload expensive work to concurrent thread pool
    @concurrent
    static func extractSubject(from data: Data) async -> Sticker { /* ... */ }
}

// Callers must await
let processor = PhotoProcessor()
processedPhotos[item.id] = await processor.extractSticker(data: data, with: item.id)
```

`@concurrent`를 사용하는 방법:
1. 포함하는 타입을 `nonisolated`로 표시
2. 함수에 `@concurrent` 추가
3. 아직 비동기가 아니라면 `async` 추가
4. 호출 지점에 `await` 추가

## 주요 설계 결정

| 결정 | 근거 |
|----------|-----------|
| 기본적으로 싱글 스레드 | 대부분의 자연스러운 코드는 data race 프리이며, concurrency는 선택 사항임 |
| 호출한 actor에 머무는 Async | data race 에러를 유발하던 암시적 오프로딩을 제거 |
| Isolated conformances | MainActor 타입이 안전하지 않은 우회 방법 없이 프로토콜을 준수할 수 있음 |
| `@concurrent` 명시적 선택 | 백그라운드 실행은 우연이 아닌 의도적인 성능 선택임 |
| MainActor 기본 추론 | 앱 타겟에 대한 보일러플레이트 `@MainActor` 어노테이션을 줄임 |
| 선택적 채택 | 중단 없는 마이그레이션 경로 — 기능을 점진적으로 활성화 |

## 마이그레이션 단계

1. **Xcode에서 활성화**: 빌드 설정의 Swift Compiler > Concurrency 섹션
2. **SPM에서 활성화**: 패키지 매니페스트에서 `SwiftSettings` API 사용
3. **마이그레이션 도구 사용**: swift.org/migration을 통한 자동 코드 변경
4. **MainActor 기본값으로 시작**: 앱 타겟에 대해 추론 모드 활성화
5. **필요한 곳에 `@concurrent` 추가**: 먼저 프로파일링한 후 hot path를 오프로딩
6. **철저히 테스트**: data race 이슈가 컴파일 타임 에러가 됨

## Best Practices

- **MainActor에서 시작** — 먼저 싱글 스레드 코드를 작성하고 나중에 최적화
- **CPU 집약적인 작업에만 `@concurrent` 사용** — 이미지 처리, 압축, 복잡한 계산 등
- **MainActor 추론 모드 활성화** — 대부분 싱글 스레드인 앱 타겟에 대해
- **오프로딩 전 프로파일링** — Instruments를 사용하여 실제 병목 현상 찾기
- **MainActor로 전역 변수 보호** — 전역/정적 가변 상태는 actor isolation이 필요함
- **isolated conformances 사용** — nonisolated 우회 방법이나 `@Sendable` 래퍼 대신
- **점진적으로 마이그레이션** — 빌드 설정에서 기능을 하나씩 활성화

## 피해야 할 Anti-Patterns

- 모든 async 함수에 `@concurrent` 적용 (대부분은 백그라운드 실행이 필요 없음)
- isolation을 이해하지 못한 채 컴파일러 에러를 억제하기 위해 `nonisolated` 사용
- actor가 동일한 안전성을 제공함에도 레거시 `DispatchQueue` 패턴 유지
- concurrency 관련 Foundation Models 코드에서 `model.availability` 체크 건너뛰기
- 컴파일러와 싸우기 — 데이터 레이스가 보고된다면 코드에 실제 concurrency 이슈가 있는 것임
- 모든 async 코드가 백그라운드에서 실행된다고 가정 (Swift 6.2 기본값: 호출한 actor에 머묾)

## 사용 시점

- 모든 새로운 Swift 6.2+ 프로젝트 (Approachable Concurrency가 권장되는 기본값임)
- 기존 앱을 Swift 5.x 또는 6.0/6.1 concurrency에서 마이그레이션할 때
- Xcode 26 채택 시 data race safety 컴파일러 에러를 해결할 때
- MainActor 중심의 앱 아키텍처 구축 (대부분의 UI 앱)
- 성능 최적화 — 특정 무거운 계산을 백그라운드로 오프로딩할 때

---
name: foundation-models-on-device
description: 온디바이스 LLM을 위한 Apple FoundationModels 프레임워크 — iOS 26+에서 텍스트 생성, @Generable을 이용한 가이드된 생성, 도구 호출, 스냅샷 스트리밍.
---

# FoundationModels: 온디바이스 LLM (iOS 26)

FoundationModels 프레임워크를 사용하여 Apple의 온디바이스 언어 모델을 앱에 통합하기 위한 패턴. 텍스트 생성, `@Generable`을 이용한 구조화된 출력, 커스텀 도구 호출, 스냅샷 스트리밍을 다루며, 개인정보 보호와 오프라인 지원을 위해 모두 온디바이스에서 실행됩니다.

## 활성화 시점

- 온디바이스 Apple Intelligence를 사용한 AI 기능 구축
- 클라우드 의존 없이 텍스트 생성 또는 요약
- 자연어 입력에서 구조화된 데이터 추출
- 도메인별 AI 작업을 위한 커스텀 도구 호출 구현
- 실시간 UI 업데이트를 위한 구조화된 응답 스트리밍
- 개인정보 보호 AI 필요 (데이터가 디바이스를 벗어나지 않음)

## 핵심 패턴 — 가용성 확인

세션 생성 전 항상 모델 가용성을 확인하세요:

```swift
struct GenerativeView: View {
    private var model = SystemLanguageModel.default

    var body: some View {
        switch model.availability {
        case .available:
            ContentView()
        case .unavailable(.deviceNotEligible):
            Text("Device not eligible for Apple Intelligence")
        case .unavailable(.appleIntelligenceNotEnabled):
            Text("Please enable Apple Intelligence in Settings")
        case .unavailable(.modelNotReady):
            Text("Model is downloading or not ready")
        case .unavailable(let other):
            Text("Model unavailable: \(other)")
        }
    }
}
```

## 핵심 패턴 — 기본 세션

```swift
// Single-turn: create a new session each time
let session = LanguageModelSession()
let response = try await session.respond(to: "What's a good month to visit Paris?")
print(response.content)

// Multi-turn: reuse session for conversation context
let session = LanguageModelSession(instructions: """
    You are a cooking assistant.
    Provide recipe suggestions based on ingredients.
    Keep suggestions brief and practical.
    """)

let first = try await session.respond(to: "I have chicken and rice")
let followUp = try await session.respond(to: "What about a vegetarian option?")
```

instructions의 핵심 포인트:
- 모델의 역할 정의 ("You are a mentor")
- 할 일 지정 ("Help extract calendar events")
- 스타일 설정 ("Respond as briefly as possible")
- 안전 조치 추가 ("Respond with 'I can't help with that' for dangerous requests")

## 핵심 패턴 — @Generable을 이용한 가이드된 생성

원시 문자열 대신 구조화된 Swift 타입 생성:

### 1. Generable 타입 정의

```swift
@Generable(description: "Basic profile information about a cat")
struct CatProfile {
    var name: String

    @Guide(description: "The age of the cat", .range(0...20))
    var age: Int

    @Guide(description: "A one sentence profile about the cat's personality")
    var profile: String
}
```

### 2. 구조화된 출력 요청

```swift
let response = try await session.respond(
    to: "Generate a cute rescue cat",
    generating: CatProfile.self
)

// Access structured fields directly
print("Name: \(response.content.name)")
print("Age: \(response.content.age)")
print("Profile: \(response.content.profile)")
```

### 지원하는 @Guide 제약 조건

- `.range(0...20)` — 숫자 범위
- `.count(3)` — 배열 요소 수
- `description:` — 생성을 위한 의미론적 가이드

## 핵심 패턴 — 도구 호출

도메인별 작업을 위해 모델이 커스텀 코드를 호출하도록 허용:

### 1. 도구 정의

```swift
struct RecipeSearchTool: Tool {
    let name = "recipe_search"
    let description = "Search for recipes matching a given term and return a list of results."

    @Generable
    struct Arguments {
        var searchTerm: String
        var numberOfResults: Int
    }

    func call(arguments: Arguments) async throws -> ToolOutput {
        let recipes = await searchRecipes(
            term: arguments.searchTerm,
            limit: arguments.numberOfResults
        )
        return .string(recipes.map { "- \($0.name): \($0.description)" }.joined(separator: "\n"))
    }
}
```

### 2. 도구와 함께 세션 생성

```swift
let session = LanguageModelSession(tools: [RecipeSearchTool()])
let response = try await session.respond(to: "Find me some pasta recipes")
```

### 3. 도구 오류 처리

```swift
do {
    let answer = try await session.respond(to: "Find a recipe for tomato soup.")
} catch let error as LanguageModelSession.ToolCallError {
    print(error.tool.name)
    if case .databaseIsEmpty = error.underlyingError as? RecipeSearchToolError {
        // Handle specific tool error
    }
}
```

## 핵심 패턴 — 스냅샷 스트리밍

`PartiallyGenerated` 타입으로 실시간 UI를 위한 구조화된 응답 스트리밍:

```swift
@Generable
struct TripIdeas {
    @Guide(description: "Ideas for upcoming trips")
    var ideas: [String]
}

let stream = session.streamResponse(
    to: "What are some exciting trip ideas?",
    generating: TripIdeas.self
)

for try await partial in stream {
    // partial: TripIdeas.PartiallyGenerated (all properties Optional)
    print(partial)
}
```

### SwiftUI 통합

```swift
@State private var partialResult: TripIdeas.PartiallyGenerated?
@State private var errorMessage: String?

var body: some View {
    List {
        ForEach(partialResult?.ideas ?? [], id: \.self) { idea in
            Text(idea)
        }
    }
    .overlay {
        if let errorMessage { Text(errorMessage).foregroundStyle(.red) }
    }
    .task {
        do {
            let stream = session.streamResponse(to: prompt, generating: TripIdeas.self)
            for try await partial in stream {
                partialResult = partial
            }
        } catch {
            errorMessage = error.localizedDescription
        }
    }
}
```

## 주요 설계 결정사항

| 결정 | 근거 |
|----------|-----------|
| 온디바이스 실행 | 개인정보 보호 — 데이터가 디바이스를 벗어나지 않음; 오프라인 동작 |
| 4,096 토큰 제한 | 온디바이스 모델 제약; 세션 간에 대용량 데이터를 청크로 분할 |
| 스냅샷 스트리밍 (델타 아님) | 구조화된 출력 친화적; 각 스냅샷은 완전한 부분 상태 |
| `@Generable` 매크로 | 구조화된 생성을 위한 컴파일 타임 안전성; `PartiallyGenerated` 타입 자동 생성 |
| 세션당 단일 요청 | `isResponding`이 동시 요청 방지; 필요 시 여러 세션 생성 |
| `response.content` (`.output` 아님) | 올바른 API — 항상 `.content` 프로퍼티로 결과에 접근 |

## 모범 사례

- **항상 `model.availability` 확인** — 세션 생성 전 모든 비가용 케이스 처리
- **`instructions` 사용** — 모델 동작 안내 — 프롬프트보다 우선순위가 높음
- **새 요청 전 `isResponding` 확인** — 세션은 한 번에 하나의 요청만 처리
- **결과에는 `response.content` 접근** — `.output`이 아님
- **대용량 입력을 청크로 분할** — 4,096 토큰 제한은 instructions + 프롬프트 + 출력의 합산에 적용
- **구조화된 출력에는 `@Generable` 사용** — 원시 문자열 파싱보다 더 강력한 보장
- **`GenerationOptions(temperature:)` 사용** — 창의성 조절 (높을수록 더 창의적)
- **Instruments로 모니터링** — Xcode Instruments로 요청 성능 프로파일링

## 피해야 할 안티패턴

- `model.availability` 확인 없이 세션 생성
- 4,096 토큰 컨텍스트 윈도우를 초과하는 입력 전송
- 단일 세션에서 동시 요청 시도
- 응답 데이터 접근에 `.content` 대신 `.output` 사용
- `@Generable` 구조화된 출력을 사용할 수 있는데 원시 문자열 응답 파싱
- 단일 프롬프트에 복잡한 다단계 로직 구축 — 여러 집중된 프롬프트로 분리
- 모델이 항상 가용하다고 가정 — 디바이스 적합성 및 설정이 다를 수 있음

## 사용 시점

- 개인정보 보호가 중요한 앱의 온디바이스 텍스트 생성
- 사용자 입력에서 구조화된 데이터 추출 (폼, 자연어 명령)
- 오프라인에서도 동작해야 하는 AI 보조 기능
- 생성된 콘텐츠를 점진적으로 표시하는 스트리밍 UI
- 도구 호출을 통한 도메인별 AI 작업 (검색, 계산, 조회)

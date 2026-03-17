---
name: swiftui-patterns
description: SwiftUI 아키텍처 패턴, @Observable을 활용한 상태 관리, 뷰 합성, 내비게이션, 성능 최적화 및 최신 iOS/macOS UI 모범 사례입니다.
---

# SwiftUI 패턴

Apple 플랫폼에서 선언적이고 성능 좋은 사용자 인터페이스를 구축하기 위한 최신 SwiftUI 패턴입니다. Observation 프레임워크, 뷰 합성, 타입 안전 내비게이션, 성능 최적화를 다룹니다.

## 활성화 시점

- SwiftUI 뷰 구축 및 상태 관리 시 (`@State`, `@Observable`, `@Binding`)
- `NavigationStack`을 사용한 내비게이션 흐름 설계 시
- 뷰 모델 및 데이터 흐름 구조화 시
- 리스트 및 복잡한 레이아웃의 렌더링 성능 최적화 시
- SwiftUI에서 environment 값 및 의존성 주입 작업 시

## 상태 관리

### Property Wrapper 선택

가장 단순한 wrapper를 선택하세요:

| Wrapper | 사용 사례 |
|---------|----------|
| `@State` | 뷰 로컬 값 타입 (토글, 폼 필드, 시트 표시) |
| `@Binding` | 부모의 `@State`에 대한 양방향 참조 |
| `@Observable` class + `@State` | 여러 속성을 가진 소유 모델 |
| `@Observable` class (wrapper 없음) | 부모로부터 전달된 읽기 전용 참조 |
| `@Bindable` | `@Observable` 속성에 대한 양방향 바인딩 |
| `@Environment` | `.environment()`를 통해 주입된 공유 의존성 |

### @Observable ViewModel

`@Observable`을 사용하세요 (`ObservableObject` 아님) -- 속성 수준의 변경을 추적하여 SwiftUI가 변경된 속성을 읽는 뷰만 다시 렌더링합니다:

```swift
@Observable
final class ItemListViewModel {
    private(set) var items: [Item] = []
    private(set) var isLoading = false
    var searchText = ""

    private let repository: any ItemRepository

    init(repository: any ItemRepository = DefaultItemRepository()) {
        self.repository = repository
    }

    func load() async {
        isLoading = true
        defer { isLoading = false }
        items = (try? await repository.fetchAll()) ?? []
    }
}
```

### ViewModel을 사용하는 뷰

```swift
struct ItemListView: View {
    @State private var viewModel: ItemListViewModel

    init(viewModel: ItemListViewModel = ItemListViewModel()) {
        _viewModel = State(initialValue: viewModel)
    }

    var body: some View {
        List(viewModel.items) { item in
            ItemRow(item: item)
        }
        .searchable(text: $viewModel.searchText)
        .overlay { if viewModel.isLoading { ProgressView() } }
        .task { await viewModel.load() }
    }
}
```

### Environment 주입

`@EnvironmentObject`를 `@Environment`로 대체하세요:

```swift
// Inject
ContentView()
    .environment(authManager)

// Consume
struct ProfileView: View {
    @Environment(AuthManager.self) private var auth

    var body: some View {
        Text(auth.currentUser?.name ?? "Guest")
    }
}
```

## 뷰 합성

### 서브뷰 추출로 무효화 범위 제한

뷰를 작고 집중된 struct로 분리하세요. 상태가 변경되면 해당 상태를 읽는 서브뷰만 다시 렌더링됩니다:

```swift
struct OrderView: View {
    @State private var viewModel = OrderViewModel()

    var body: some View {
        VStack {
            OrderHeader(title: viewModel.title)
            OrderItemList(items: viewModel.items)
            OrderTotal(total: viewModel.total)
        }
    }
}
```

### 재사용 가능한 스타일링을 위한 ViewModifier

```swift
struct CardModifier: ViewModifier {
    func body(content: Content) -> some View {
        content
            .padding()
            .background(.regularMaterial)
            .clipShape(RoundedRectangle(cornerRadius: 12))
    }
}

extension View {
    func cardStyle() -> some View {
        modifier(CardModifier())
    }
}
```

## 내비게이션

### 타입 안전 NavigationStack

프로그래밍 방식의 타입 안전 라우팅을 위해 `NavigationStack`과 `NavigationPath`를 사용하세요:

```swift
@Observable
final class Router {
    var path = NavigationPath()

    func navigate(to destination: Destination) {
        path.append(destination)
    }

    func popToRoot() {
        path = NavigationPath()
    }
}

enum Destination: Hashable {
    case detail(Item.ID)
    case settings
    case profile(User.ID)
}

struct RootView: View {
    @State private var router = Router()

    var body: some View {
        NavigationStack(path: $router.path) {
            HomeView()
                .navigationDestination(for: Destination.self) { dest in
                    switch dest {
                    case .detail(let id): ItemDetailView(itemID: id)
                    case .settings: SettingsView()
                    case .profile(let id): ProfileView(userID: id)
                    }
                }
        }
        .environment(router)
    }
}
```

## 성능

### 대량 컬렉션에 Lazy 컨테이너 사용

`LazyVStack`과 `LazyHStack`은 화면에 보일 때만 뷰를 생성합니다:

```swift
ScrollView {
    LazyVStack(spacing: 8) {
        ForEach(items) { item in
            ItemRow(item: item)
        }
    }
}
```

### 안정적인 식별자

`ForEach`에서 항상 안정적이고 고유한 ID를 사용하세요 -- 배열 인덱스 사용을 피하세요:

```swift
// Use Identifiable conformance or explicit id
ForEach(items, id: \.stableID) { item in
    ItemRow(item: item)
}
```

### body에서 비용이 큰 작업 금지

- `body` 내에서 I/O, 네트워크 호출 또는 무거운 연산을 절대 수행하지 마세요
- 비동기 작업에는 `.task {}`를 사용하세요 -- 뷰가 사라지면 자동으로 취소됩니다
- 스크롤 뷰에서 `.sensoryFeedback()`과 `.geometryGroup()`을 절제하여 사용하세요
- 리스트에서 `.shadow()`, `.blur()`, `.mask()`를 최소화하세요 -- 오프스크린 렌더링을 유발합니다

### Equatable 준수

비용이 큰 body를 가진 뷰의 경우 `Equatable`을 준수하여 불필요한 재렌더링을 건너뛸 수 있습니다:

```swift
struct ExpensiveChartView: View, Equatable {
    let dataPoints: [DataPoint] // DataPoint must conform to Equatable

    static func == (lhs: Self, rhs: Self) -> Bool {
        lhs.dataPoints == rhs.dataPoints
    }

    var body: some View {
        // Complex chart rendering
    }
}
```

## 미리보기

빠른 반복을 위해 인라인 모의 데이터와 함께 `#Preview` 매크로를 사용하세요:

```swift
#Preview("Empty state") {
    ItemListView(viewModel: ItemListViewModel(repository: EmptyMockRepository()))
}

#Preview("Loaded") {
    ItemListView(viewModel: ItemListViewModel(repository: PopulatedMockRepository()))
}
```

## 피해야 할 안티패턴

- 새 코드에서 `ObservableObject` / `@Published` / `@StateObject` / `@EnvironmentObject` 사용 -- `@Observable`로 마이그레이션하세요
- `body`나 `init`에서 직접 비동기 작업 수행 -- `.task {}` 또는 명시적 load 메서드를 사용하세요
- 데이터를 소유하지 않는 자식 뷰 내부에서 뷰 모델을 `@State`로 생성 -- 대신 부모에서 전달하세요
- `AnyView` 타입 소거 사용 -- 조건부 뷰에는 `@ViewBuilder` 또는 `Group`을 선호하세요
- actor와 데이터를 주고받을 때 `Sendable` 요구사항 무시

## 참조

actor 기반 영속성 패턴은 skill `swift-actor-persistence`를 참조하세요.
프로토콜 기반 DI와 Swift Testing 테스트는 skill `swift-protocol-di-testing`을 참조하세요.

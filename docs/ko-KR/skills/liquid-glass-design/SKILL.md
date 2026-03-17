---
name: liquid-glass-design
description: iOS 26 Liquid Glass 디자인 시스템 — SwiftUI, UIKit, WidgetKit을 위한 블러, 반사, 인터랙티브 모핑이 적용된 동적 유리 재질.
---

# Liquid Glass 디자인 시스템 (iOS 26)

Apple의 Liquid Glass를 구현하는 패턴 — 배경 콘텐츠를 블러 처리하고, 주변 콘텐츠의 색상과 빛을 반사하며, 터치 및 포인터 상호작용에 반응하는 동적 재질입니다. SwiftUI, UIKit, WidgetKit 연동을 다룹니다.

## 활성화 시점

- 새로운 디자인 언어를 적용한 iOS 26+ 앱을 개발하거나 업데이트할 때
- 유리 스타일 버튼, 카드, 툴바, 컨테이너를 구현할 때
- 유리 요소 간의 모핑 전환을 만들 때
- 위젯에 Liquid Glass 효과를 적용할 때
- 기존 blur/material 효과를 새로운 Liquid Glass API로 마이그레이션할 때

## 핵심 패턴 — SwiftUI

### 기본 유리 효과

모든 뷰에 Liquid Glass를 추가하는 가장 간단한 방법:

```swift
Text("Hello, World!")
    .font(.title)
    .padding()
    .glassEffect()  // 기본값: regular 변형, capsule 모양
```

### 모양과 색조 커스터마이징

```swift
Text("Hello, World!")
    .font(.title)
    .padding()
    .glassEffect(.regular.tint(.orange).interactive(), in: .rect(cornerRadius: 16.0))
```

주요 커스터마이징 옵션:
- `.regular` — 표준 유리 효과
- `.tint(Color)` — 강조를 위한 색조 추가
- `.interactive()` — 터치 및 포인터 상호작용에 반응
- 모양: `.capsule` (기본값), `.rect(cornerRadius:)`, `.circle`

### 유리 버튼 스타일

```swift
Button("Click Me") { /* action */ }
    .buttonStyle(.glass)

Button("Important") { /* action */ }
    .buttonStyle(.glassProminent)
```

### 여러 요소를 위한 GlassEffectContainer

성능 최적화와 모핑을 위해 항상 여러 유리 뷰를 컨테이너로 감싸세요:

```swift
GlassEffectContainer(spacing: 40.0) {
    HStack(spacing: 40.0) {
        Image(systemName: "scribble.variable")
            .frame(width: 80.0, height: 80.0)
            .font(.system(size: 36))
            .glassEffect()

        Image(systemName: "eraser.fill")
            .frame(width: 80.0, height: 80.0)
            .font(.system(size: 36))
            .glassEffect()
    }
}
```

`spacing` 파라미터는 병합 거리를 제어합니다 — 요소가 가까울수록 유리 모양이 서로 혼합됩니다.

### 유리 효과 통합

`glassEffectUnion`으로 여러 뷰를 하나의 유리 모양으로 합칩니다:

```swift
@Namespace private var namespace

GlassEffectContainer(spacing: 20.0) {
    HStack(spacing: 20.0) {
        ForEach(symbolSet.indices, id: \.self) { item in
            Image(systemName: symbolSet[item])
                .frame(width: 80.0, height: 80.0)
                .glassEffect()
                .glassEffectUnion(id: item < 2 ? "group1" : "group2", namespace: namespace)
        }
    }
}
```

### 모핑 전환

유리 요소가 나타나거나 사라질 때 부드러운 모핑을 만듭니다:

```swift
@State private var isExpanded = false
@Namespace private var namespace

GlassEffectContainer(spacing: 40.0) {
    HStack(spacing: 40.0) {
        Image(systemName: "scribble.variable")
            .frame(width: 80.0, height: 80.0)
            .glassEffect()
            .glassEffectID("pencil", in: namespace)

        if isExpanded {
            Image(systemName: "eraser.fill")
                .frame(width: 80.0, height: 80.0)
                .glassEffect()
                .glassEffectID("eraser", in: namespace)
        }
    }
}

Button("Toggle") {
    withAnimation { isExpanded.toggle() }
}
.buttonStyle(.glass)
```

### 사이드바 아래로 수평 스크롤 확장

수평 스크롤 콘텐츠가 사이드바나 인스펙터 아래로 확장되도록 하려면, `ScrollView` 콘텐츠가 컨테이너의 leading/trailing 가장자리까지 도달하도록 합니다. 시스템은 레이아웃이 가장자리까지 확장될 때 사이드바 아래 스크롤 동작을 자동으로 처리합니다 — 추가 modifier는 필요하지 않습니다.

## 핵심 패턴 — UIKit

### 기본 UIGlassEffect

```swift
let glassEffect = UIGlassEffect()
glassEffect.tintColor = UIColor.systemBlue.withAlphaComponent(0.3)
glassEffect.isInteractive = true

let visualEffectView = UIVisualEffectView(effect: glassEffect)
visualEffectView.translatesAutoresizingMaskIntoConstraints = false
visualEffectView.layer.cornerRadius = 20
visualEffectView.clipsToBounds = true

view.addSubview(visualEffectView)
NSLayoutConstraint.activate([
    visualEffectView.centerXAnchor.constraint(equalTo: view.centerXAnchor),
    visualEffectView.centerYAnchor.constraint(equalTo: view.centerYAnchor),
    visualEffectView.widthAnchor.constraint(equalToConstant: 200),
    visualEffectView.heightAnchor.constraint(equalToConstant: 120)
])

// contentView에 콘텐츠 추가
let label = UILabel()
label.text = "Liquid Glass"
label.translatesAutoresizingMaskIntoConstraints = false
visualEffectView.contentView.addSubview(label)
NSLayoutConstraint.activate([
    label.centerXAnchor.constraint(equalTo: visualEffectView.contentView.centerXAnchor),
    label.centerYAnchor.constraint(equalTo: visualEffectView.contentView.centerYAnchor)
])
```

### 여러 요소를 위한 UIGlassContainerEffect

```swift
let containerEffect = UIGlassContainerEffect()
containerEffect.spacing = 40.0

let containerView = UIVisualEffectView(effect: containerEffect)

let firstGlass = UIVisualEffectView(effect: UIGlassEffect())
let secondGlass = UIVisualEffectView(effect: UIGlassEffect())

containerView.contentView.addSubview(firstGlass)
containerView.contentView.addSubview(secondGlass)
```

### 스크롤 엣지 효과

```swift
scrollView.topEdgeEffect.style = .automatic
scrollView.bottomEdgeEffect.style = .hard
scrollView.leftEdgeEffect.isHidden = true
```

### 툴바 유리 연동

```swift
let favoriteButton = UIBarButtonItem(image: UIImage(systemName: "heart"), style: .plain, target: self, action: #selector(favoriteAction))
favoriteButton.hidesSharedBackground = true  // 공유 유리 배경에서 제외
```

## 핵심 패턴 — WidgetKit

### 렌더링 모드 감지

```swift
struct MyWidgetView: View {
    @Environment(\.widgetRenderingMode) var renderingMode

    var body: some View {
        if renderingMode == .accented {
            // 색조 모드: 흰색 색조, 테마 유리 배경
        } else {
            // 풀 컬러 모드: 표준 외관
        }
    }
}
```

### 시각적 계층을 위한 Accent 그룹

```swift
HStack {
    VStack(alignment: .leading) {
        Text("Title")
            .widgetAccentable()  // Accent 그룹
        Text("Subtitle")
            // Primary 그룹 (기본값)
    }
    Image(systemName: "star.fill")
        .widgetAccentable()  // Accent 그룹
}
```

### Accented 모드의 이미지 렌더링

```swift
Image("myImage")
    .widgetAccentedRenderingMode(.monochrome)
```

### 컨테이너 배경

```swift
VStack { /* content */ }
    .containerBackground(for: .widget) {
        Color.blue.opacity(0.2)
    }
```

## 주요 설계 결정

| 결정 | 근거 |
|----------|-----------|
| GlassEffectContainer 감싸기 | 성능 최적화, 유리 요소 간 모핑 활성화 |
| `spacing` 파라미터 | 병합 거리 제어 — 요소가 혼합되기 위한 근접도 미세 조정 |
| `@Namespace` + `glassEffectID` | 뷰 계층 변경 시 부드러운 모핑 전환 활성화 |
| `interactive()` modifier | 터치/포인터 반응에 대한 명시적 옵트인 — 모든 유리가 반응할 필요는 없음 |
| UIKit의 UIGlassContainerEffect | SwiftUI와의 일관성을 위한 동일한 컨테이너 패턴 |
| 위젯의 Accented 렌더링 모드 | 사용자가 색조 홈 화면을 선택할 때 시스템이 색조 유리를 적용 |

## 모범 사례

- **항상 GlassEffectContainer를 사용하세요** 여러 형제 뷰에 유리를 적용할 때 — 모핑을 활성화하고 렌더링 성능을 향상시킵니다
- **`.glassEffect()`는 나중에 적용하세요** 다른 외관 modifier (frame, font, padding) 다음에
- **`.interactive()`는** 사용자 상호작용에 반응하는 요소에만 사용하세요 (버튼, 토글 가능한 항목)
- **컨테이너의 spacing을 신중하게 선택하세요** 유리 효과가 병합되는 시점을 제어합니다
- **`withAnimation`을 사용하세요** 뷰 계층이 변경될 때 부드러운 모핑 전환을 활성화합니다
- **여러 외관에서 테스트하세요** — 라이트 모드, 다크 모드, accented/tinted 모드
- **접근성 대비를 보장하세요** — 유리 위의 텍스트는 읽기 쉬워야 합니다

## 피해야 할 안티 패턴

- GlassEffectContainer 없이 여러 독립 `.glassEffect()` 뷰를 사용하는 것
- 너무 많은 유리 효과를 중첩하는 것 — 성능과 시각적 명확성 저하
- 모든 뷰에 유리를 적용하는 것 — 인터랙티브 요소, 툴바, 카드에만 사용
- UIKit에서 코너 반경 사용 시 `clipsToBounds = true` 잊기
- 위젯에서 accented 렌더링 모드를 무시하는 것 — 색조 홈 화면 외관이 깨짐
- 유리 뒤에 불투명 배경 사용 — 반투명 효과가 무력화됨

## 사용 시기

- 새로운 iOS 26 디자인의 네비게이션 바, 툴바, 탭 바
- 플로팅 액션 버튼 및 카드 스타일 컨테이너
- 시각적 깊이와 터치 피드백이 필요한 인터랙티브 컨트롤
- 시스템의 Liquid Glass 외관과 통합되어야 하는 위젯
- 관련 UI 상태 간의 모핑 전환

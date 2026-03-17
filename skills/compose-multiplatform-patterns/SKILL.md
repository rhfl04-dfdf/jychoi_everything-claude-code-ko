---
name: compose-multiplatform-patterns
description: KMP 프로젝트를 위한 Compose Multiplatform 및 Jetpack Compose 패턴 — state management, navigation, theming, performance 및 platform-specific UI.
origin: ECC
---

# Compose Multiplatform 패턴

Compose Multiplatform 및 Jetpack Compose를 사용하여 Android, iOS, Desktop, Web에 걸쳐 공유 UI를 구축하기 위한 패턴입니다. State management, navigation, theming 및 performance를 다룹니다.

## 활성화 시점

- Compose UI 구축 시 (Jetpack Compose 또는 Compose Multiplatform)
- ViewModel 및 Compose state를 사용하여 UI state 관리 시
- KMP 또는 Android 프로젝트에서 navigation 구현 시
- 재사용 가능한 composable 및 design system 설계 시
- Recomposition 및 렌더링 performance 최적화 시

## State Management

### ViewModel + Single State Object

화면 state를 위해 단일 data class를 사용하세요. 이를 `StateFlow`로 노출하고 Compose에서 수집(collect)합니다.

```kotlin
data class ItemListState(
    val items: List<Item> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null,
    val searchQuery: String = ""
)

class ItemListViewModel(
    private val getItems: GetItemsUseCase
) : ViewModel() {
    private val _state = MutableStateFlow(ItemListState())
    val state: StateFlow<ItemListState> = _state.asStateFlow()

    fun onSearch(query: String) {
        _state.update { it.copy(searchQuery = query) }
        loadItems(query)
    }

    private fun loadItems(query: String) {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true) }
            getItems(query).fold(
                onSuccess = { items -> _state.update { it.copy(items = items, isLoading = false) } },
                onFailure = { e -> _state.update { it.copy(error = e.message, isLoading = false) } }
            )
        }
    }
}
```

### Compose에서 State 수집하기

```kotlin
@Composable
fun ItemListScreen(viewModel: ItemListViewModel = koinViewModel()) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    ItemListContent(
        state = state,
        onSearch = viewModel::onSearch
    )
}

@Composable
private fun ItemListContent(
    state: ItemListState,
    onSearch: (String) -> Unit
) {
    // Stateless composable — easy to preview and test
}
```

### Event Sink 패턴

복잡한 화면의 경우, 여러 개의 콜백 람다 대신 이벤트를 위한 sealed interface를 사용하세요.

```kotlin
sealed interface ItemListEvent {
    data class Search(val query: String) : ItemListEvent
    data class Delete(val itemId: String) : ItemListEvent
    data object Refresh : ItemListEvent
}

// In ViewModel
fun onEvent(event: ItemListEvent) {
    when (event) {
        is ItemListEvent.Search -> onSearch(event.query)
        is ItemListEvent.Delete -> deleteItem(event.itemId)
        is ItemListEvent.Refresh -> loadItems(_state.value.searchQuery)
    }
}

// In Composable — single lambda instead of many
ItemListContent(
    state = state,
    onEvent = viewModel::onEvent
)
```

## Navigation

### Type-Safe Navigation (Compose Navigation 2.8+)

경로(route)를 `@Serializable` 객체로 정의하세요.

```kotlin
@Serializable data object HomeRoute
@Serializable data class DetailRoute(val id: String)
@Serializable data object SettingsRoute

@Composable
fun AppNavHost(navController: NavHostController = rememberNavController()) {
    NavHost(navController, startDestination = HomeRoute) {
        composable<HomeRoute> {
            HomeScreen(onNavigateToDetail = { id -> navController.navigate(DetailRoute(id)) })
        }
        composable<DetailRoute> { backStackEntry ->
            val route = backStackEntry.toRoute<DetailRoute>()
            DetailScreen(id = route.id)
        }
        composable<SettingsRoute> { SettingsScreen() }
    }
}
```

### Dialog 및 Bottom Sheet Navigation

명령형 show/hide 대신 `dialog()` 및 오버레이 패턴을 사용하세요.

```kotlin
NavHost(navController, startDestination = HomeRoute) {
    composable<HomeRoute> { /* ... */ }
    dialog<ConfirmDeleteRoute> { backStackEntry ->
        val route = backStackEntry.toRoute<ConfirmDeleteRoute>()
        ConfirmDeleteDialog(
            itemId = route.itemId,
            onConfirm = { navController.popBackStack() },
            onDismiss = { navController.popBackStack() }
        )
    }
}
```

## Composable 설계

### Slot-Based API

유연성을 위해 슬롯(slot) 파라미터를 사용하여 composable을 설계하세요.

```kotlin
@Composable
fun AppCard(
    modifier: Modifier = Modifier,
    header: @Composable () -> Unit = {},
    content: @Composable ColumnScope.() -> Unit,
    actions: @Composable RowScope.() -> Unit = {}
) {
    Card(modifier = modifier) {
        Column {
            header()
            Column(content = content)
            Row(horizontalArrangement = Arrangement.End, content = actions)
        }
    }
}
```

### Modifier 순서

Modifier 순서가 중요합니다 — 다음 순서대로 적용하세요.

```kotlin
Text(
    text = "Hello",
    modifier = Modifier
        .padding(16.dp)          // 1. Layout (padding, size)
        .clip(RoundedCornerShape(8.dp))  // 2. Shape
        .background(Color.White) // 3. Drawing (background, border)
        .clickable { }           // 4. Interaction
)
```

## KMP Platform-Specific UI

### Platform Composable을 위한 expect/actual

```kotlin
// commonMain
@Composable
expect fun PlatformStatusBar(darkIcons: Boolean)

// androidMain
@Composable
actual fun PlatformStatusBar(darkIcons: Boolean) {
    val systemUiController = rememberSystemUiController()
    SideEffect { systemUiController.setStatusBarColor(Color.Transparent, darkIcons) }
}

// iosMain
@Composable
actual fun PlatformStatusBar(darkIcons: Boolean) {
    // iOS handles this via UIKit interop or Info.plist
}
```

## Performance

### Skippable Recomposition을 위한 Stable 타입

모든 프로퍼티가 stable할 때 클래스를 `@Stable` 또는 `@Immutable`로 표시하세요.

```kotlin
@Immutable
data class ItemUiModel(
    val id: String,
    val title: String,
    val description: String,
    val progress: Float
)
```

### key() 및 Lazy List의 올바른 사용

```kotlin
LazyColumn {
    items(
        items = items,
        key = { it.id }  // Stable keys enable item reuse and animations
    ) { item ->
        ItemRow(item = item)
    }
}
```

### derivedStateOf를 사용하여 읽기 지연시키기

```kotlin
val listState = rememberLazyListState()
val showScrollToTop by remember {
    derivedStateOf { listState.firstVisibleItemIndex > 5 }
}
```

### Recomposition 중 할당 피하기

```kotlin
// BAD — new lambda and list every recomposition
items.filter { it.isActive }.forEach { ActiveItem(it, onClick = { handle(it) }) }

// GOOD — key each item so callbacks stay attached to the right row
val activeItems = remember(items) { items.filter { it.isActive } }
activeItems.forEach { item ->
    key(item.id) {
        ActiveItem(item, onClick = { handle(item) })
    }
}
```

## Theming

### Material 3 Dynamic Theming

```kotlin
@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true,
    content: @Composable () -> Unit
) {
    val colorScheme = when {
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            if (darkTheme) dynamicDarkColorScheme(LocalContext.current)
            else dynamicLightColorScheme(LocalContext.current)
        }
        darkTheme -> darkColorScheme()
        else -> lightColorScheme()
    }

    MaterialTheme(colorScheme = colorScheme, content = content)
}
```

## 피해야 할 안티 패턴

- Lifecycle 측면에서 `MutableStateFlow`와 `collectAsStateWithLifecycle`을 사용하는 것이 더 안전한데도 ViewModel에서 `mutableStateOf`를 사용하는 것
- `NavController`를 composable 깊숙이 전달하는 것 — 대신 람다 콜백을 전달하세요.
- `@Composable` 함수 내부에서의 무거운 계산 — ViewModel 또는 `remember {}`로 이동하세요.
- ViewModel 초기화의 대용으로 `LaunchedEffect(Unit)`을 사용하는 것 — 일부 설정에서 configuration change 시 재실행될 수 있습니다.
- composable 파라미터에서 새로운 객체 인스턴스를 생성하는 것 — 불필요한 recomposition을 유발합니다.

## 참고 문헌

모듈 구조 및 레이어링에 대해서는 skill: `android-clean-architecture`를 참조하세요.
Coroutine 및 Flow 패턴에 대해서는 skill: `kotlin-coroutines-flows`를 참조하세요.

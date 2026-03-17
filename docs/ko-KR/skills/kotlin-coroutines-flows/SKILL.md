---
name: kotlin-coroutines-flows
description: Android 및 KMP를 위한 Kotlin Coroutines와 Flow 패턴 — 구조적 동시성, Flow 연산자, StateFlow, 에러 처리, 테스트.
origin: ECC
---

# Kotlin Coroutines & Flows

Android 및 Kotlin Multiplatform 프로젝트에서 구조적 동시성, Flow 기반 리액티브 스트림, 코루틴 테스트를 위한 패턴.

## 활성화 시점

- Kotlin 코루틴으로 비동기 코드 작성 시
- 리액티브 데이터를 위해 Flow, StateFlow, SharedFlow 사용 시
- 동시 작업 처리 시 (병렬 로딩, 디바운스, 재시도)
- 코루틴과 Flow 테스트 시
- 코루틴 스코프 및 취소 관리 시

## 구조적 동시성

### 스코프 계층

```
Application
  └── viewModelScope (ViewModel)
        └── coroutineScope { } (구조적 자식)
              ├── async { } (동시 작업)
              └── async { } (동시 작업)
```

구조적 동시성을 항상 사용 — `GlobalScope` 절대 사용 금지:

```kotlin
// 나쁜 예
GlobalScope.launch { fetchData() }

// 좋은 예 — ViewModel 수명 주기에 스코프됨
viewModelScope.launch { fetchData() }

// 좋은 예 — 컴포저블 수명 주기에 스코프됨
LaunchedEffect(key) { fetchData() }
```

### 병렬 분해

병렬 작업에는 `coroutineScope` + `async` 사용:

```kotlin
suspend fun loadDashboard(): Dashboard = coroutineScope {
    val items = async { itemRepository.getRecent() }
    val stats = async { statsRepository.getToday() }
    val profile = async { userRepository.getCurrent() }
    Dashboard(
        items = items.await(),
        stats = stats.await(),
        profile = profile.await()
    )
}
```

### SupervisorScope

자식 실패가 형제를 취소해서는 안 될 때 `supervisorScope` 사용:

```kotlin
suspend fun syncAll() = supervisorScope {
    launch { syncItems() }       // 여기서 실패해도 syncStats는 취소되지 않음
    launch { syncStats() }
    launch { syncSettings() }
}
```

## Flow 패턴

### 콜드 Flow — 일회성에서 스트림으로 변환

```kotlin
fun observeItems(): Flow<List<Item>> = flow {
    // 데이터베이스가 변경될 때마다 재방출
    itemDao.observeAll()
        .map { entities -> entities.map { it.toDomain() } }
        .collect { emit(it) }
}
```

### UI 상태를 위한 StateFlow

```kotlin
class DashboardViewModel(
    observeProgress: ObserveUserProgressUseCase
) : ViewModel() {
    val progress: StateFlow<UserProgress> = observeProgress()
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = UserProgress.EMPTY
        )
}
```

`WhileSubscribed(5_000)`은 마지막 구독자가 떠난 후 5초 동안 업스트림을 활성 상태로 유지 — 재시작 없이 구성 변경에서 살아남습니다.

### 여러 Flow 결합

```kotlin
val uiState: StateFlow<HomeState> = combine(
    itemRepository.observeItems(),
    settingsRepository.observeTheme(),
    userRepository.observeProfile()
) { items, theme, profile ->
    HomeState(items = items, theme = theme, profile = profile)
}.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), HomeState())
```

### Flow 연산자

```kotlin
// 검색 입력 디바운스
searchQuery
    .debounce(300)
    .distinctUntilChanged()
    .flatMapLatest { query -> repository.search(query) }
    .catch { emit(emptyList()) }
    .collect { results -> _state.update { it.copy(results = results) } }

// 지수 백오프로 재시도
fun fetchWithRetry(): Flow<Data> = flow { emit(api.fetch()) }
    .retryWhen { cause, attempt ->
        if (cause is IOException && attempt < 3) {
            delay(1000L * (1 shl attempt.toInt()))
            true
        } else {
            false
        }
    }
```

### 일회성 이벤트를 위한 SharedFlow

```kotlin
class ItemListViewModel : ViewModel() {
    private val _effects = MutableSharedFlow<Effect>()
    val effects: SharedFlow<Effect> = _effects.asSharedFlow()

    sealed interface Effect {
        data class ShowSnackbar(val message: String) : Effect
        data class NavigateTo(val route: String) : Effect
    }

    private fun deleteItem(id: String) {
        viewModelScope.launch {
            repository.delete(id)
            _effects.emit(Effect.ShowSnackbar("Item deleted"))
        }
    }
}

// 컴포저블에서 수집
LaunchedEffect(Unit) {
    viewModel.effects.collect { effect ->
        when (effect) {
            is Effect.ShowSnackbar -> snackbarHostState.showSnackbar(effect.message)
            is Effect.NavigateTo -> navController.navigate(effect.route)
        }
    }
}
```

## Dispatcher

```kotlin
// CPU 집약적 작업
withContext(Dispatchers.Default) { parseJson(largePayload) }

// IO 바운드 작업
withContext(Dispatchers.IO) { database.query() }

// 메인 스레드 (UI) — viewModelScope의 기본값
withContext(Dispatchers.Main) { updateUi() }
```

KMP에서는 `Dispatchers.Default`와 `Dispatchers.Main`을 사용 (모든 플랫폼에서 사용 가능). `Dispatchers.IO`는 JVM/Android 전용 — 다른 플랫폼에서는 `Dispatchers.Default`를 사용하거나 DI를 통해 제공.

## 취소

### 협력적 취소

장기 실행 루프는 취소를 확인해야 함:

```kotlin
suspend fun processItems(items: List<Item>) = coroutineScope {
    for (item in items) {
        ensureActive()  // 취소되면 CancellationException 발생
        process(item)
    }
}
```

### try/finally를 이용한 정리

```kotlin
viewModelScope.launch {
    try {
        _state.update { it.copy(isLoading = true) }
        val data = repository.fetch()
        _state.update { it.copy(data = data) }
    } finally {
        _state.update { it.copy(isLoading = false) }  // 취소 시에도 항상 실행
    }
}
```

## 테스트

### Turbine으로 StateFlow 테스트

```kotlin
@Test
fun `search updates item list`() = runTest {
    val fakeRepository = FakeItemRepository().apply { emit(testItems) }
    val viewModel = ItemListViewModel(GetItemsUseCase(fakeRepository))

    viewModel.state.test {
        assertEquals(ItemListState(), awaitItem())  // 초기값

        viewModel.onSearch("query")
        val loading = awaitItem()
        assertTrue(loading.isLoading)

        val loaded = awaitItem()
        assertFalse(loaded.isLoading)
        assertEquals(1, loaded.items.size)
    }
}
```

### TestDispatcher로 테스트

```kotlin
@Test
fun `parallel load completes correctly`() = runTest {
    val viewModel = DashboardViewModel(
        itemRepo = FakeItemRepo(),
        statsRepo = FakeStatsRepo()
    )

    viewModel.load()
    advanceUntilIdle()

    val state = viewModel.state.value
    assertNotNull(state.items)
    assertNotNull(state.stats)
}
```

### Flow 페이킹

```kotlin
class FakeItemRepository : ItemRepository {
    private val _items = MutableStateFlow<List<Item>>(emptyList())

    override fun observeItems(): Flow<List<Item>> = _items

    fun emit(items: List<Item>) { _items.value = items }

    override suspend fun getItemsByCategory(category: String): Result<List<Item>> {
        return Result.success(_items.value.filter { it.category == category })
    }
}
```

## 피해야 할 안티패턴

- `GlobalScope` 사용 — 코루틴 누수, 구조적 취소 없음
- 스코프 없이 `init {}` 에서 Flow 수집 — `viewModelScope.launch` 사용
- 가변 컬렉션으로 `MutableStateFlow` 사용 — 항상 불변 복사본 사용: `_state.update { it.copy(list = it.list + newItem) }`
- `CancellationException` 잡기 — 올바른 취소를 위해 전파시키기
- 수집에 `flowOn(Dispatchers.Main)` 사용 — 수집 디스패처는 호출자의 디스패처
- `remember` 없이 `@Composable`에서 `Flow` 생성 — 매 리컴포지션마다 Flow 재생성

## 참고

Flow의 UI 소비에 대해서는 스킬: `compose-multiplatform-patterns` 참고.
레이어에서 코루틴이 적합한 위치에 대해서는 스킬: `android-clean-architecture` 참고.

---
paths:
  - "**/*.kt"
  - "**/*.kts"
---
# Kotlin 테스팅

> 이 파일은 [common/testing.md](../common/testing.md)를 확장하여 Kotlin 및 Android/KMP 특화 내용을 다룹니다.

## 테스트 프레임워크

- 멀티플랫폼(KMP)에는 **kotlin.test** — `@Test`, `assertEquals`, `assertTrue`
- Android 특화 테스트에는 **JUnit 4/5**
- Flow와 StateFlow 테스트에는 **Turbine**
- 코루틴 테스트에는 **kotlinx-coroutines-test** (`runTest`, `TestDispatcher`)

## Turbine을 활용한 ViewModel 테스팅

```kotlin
@Test
fun `loading state emitted then data`() = runTest {
    val repo = FakeItemRepository()
    repo.addItem(testItem)
    val viewModel = ItemListViewModel(GetItemsUseCase(repo))

    viewModel.state.test {
        assertEquals(ItemListState(), awaitItem())     // initial state
        viewModel.onEvent(ItemListEvent.Load)
        assertTrue(awaitItem().isLoading)               // loading
        assertEquals(listOf(testItem), awaitItem().items) // loaded
    }
}
```

## Mock보다 Fake 선호

Mock 프레임워크보다 직접 작성한 fake를 선호:

```kotlin
class FakeItemRepository : ItemRepository {
    private val items = mutableListOf<Item>()
    var fetchError: Throwable? = null

    override suspend fun getAll(): Result<List<Item>> {
        fetchError?.let { return Result.failure(it) }
        return Result.success(items.toList())
    }

    override fun observeAll(): Flow<List<Item>> = flowOf(items.toList())

    fun addItem(item: Item) { items.add(item) }
}
```

## 코루틴 테스팅

```kotlin
@Test
fun `parallel operations complete`() = runTest {
    val repo = FakeRepository()
    val result = loadDashboard(repo)
    advanceUntilIdle()
    assertNotNull(result.items)
    assertNotNull(result.stats)
}
```

`runTest` 사용 — 가상 시간을 자동으로 진행하고 `TestScope`를 제공합니다.

## Ktor MockEngine

```kotlin
val mockEngine = MockEngine { request ->
    when (request.url.encodedPath) {
        "/api/items" -> respond(
            content = Json.encodeToString(testItems),
            headers = headersOf(HttpHeaders.ContentType, ContentType.Application.Json.toString())
        )
        else -> respondError(HttpStatusCode.NotFound)
    }
}

val client = HttpClient(mockEngine) {
    install(ContentNegotiation) { json() }
}
```

## Room/SQLDelight 테스팅

- Room: 인메모리 테스트에 `Room.inMemoryDatabaseBuilder()` 사용
- SQLDelight: JVM 테스트에 `JdbcSqliteDriver(JdbcSqliteDriver.IN_MEMORY)` 사용

```kotlin
@Test
fun `insert and query items`() = runTest {
    val driver = JdbcSqliteDriver(JdbcSqliteDriver.IN_MEMORY)
    Database.Schema.create(driver)
    val db = Database(driver)

    db.itemQueries.insert("1", "Sample Item", "description")
    val items = db.itemQueries.getAll().executeAsList()
    assertEquals(1, items.size)
}
```

## 테스트 네이밍

백틱으로 감싼 설명적인 이름 사용:

```kotlin
@Test
fun `search with empty query returns all items`() = runTest { }

@Test
fun `delete item emits updated list without deleted item`() = runTest { }
```

## 테스트 구성

```
src/
├── commonTest/kotlin/     # Shared tests (ViewModel, UseCase, Repository)
├── androidUnitTest/kotlin/ # Android unit tests (JUnit)
├── androidInstrumentedTest/kotlin/  # Instrumented tests (Room, UI)
└── iosTest/kotlin/        # iOS-specific tests
```

최소 테스트 커버리지: 모든 기능에 대한 ViewModel + UseCase.

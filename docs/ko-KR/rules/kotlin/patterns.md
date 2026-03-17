---
paths:
  - "**/*.kt"
  - "**/*.kts"
---
# Kotlin 패턴

> 이 파일은 [common/patterns.md](../common/patterns.md)를 확장하여 Kotlin 및 Android/KMP 특화 내용을 다룹니다.

## 의존성 주입

생성자 주입을 선호합니다. KMP에는 Koin, Android 전용에는 Hilt 사용:

```kotlin
// Koin — 모듈 선언
val dataModule = module {
    single<ItemRepository> { ItemRepositoryImpl(get(), get()) }
    factory { GetItemsUseCase(get()) }
    viewModelOf(::ItemListViewModel)
}

// Hilt — 어노테이션
@HiltViewModel
class ItemListViewModel @Inject constructor(
    private val getItems: GetItemsUseCase
) : ViewModel()
```

## ViewModel 패턴

단일 상태 객체, 이벤트 싱크, 단방향 데이터 흐름:

```kotlin
data class ScreenState(
    val items: List<Item> = emptyList(),
    val isLoading: Boolean = false
)

class ScreenViewModel(private val useCase: GetItemsUseCase) : ViewModel() {
    private val _state = MutableStateFlow(ScreenState())
    val state = _state.asStateFlow()

    fun onEvent(event: ScreenEvent) {
        when (event) {
            is ScreenEvent.Load -> load()
            is ScreenEvent.Delete -> delete(event.id)
        }
    }
}
```

## Repository 패턴

- `suspend` 함수는 `Result<T>` 또는 커스텀 에러 타입 반환
- 반응형 스트림에는 `Flow` 사용
- 로컬 + 원격 데이터 소스 조율

```kotlin
interface ItemRepository {
    suspend fun getById(id: String): Result<Item>
    suspend fun getAll(): Result<List<Item>>
    fun observeAll(): Flow<List<Item>>
}
```

## UseCase 패턴

단일 책임, `operator fun invoke`:

```kotlin
class GetItemUseCase(private val repository: ItemRepository) {
    suspend operator fun invoke(id: String): Result<Item> {
        return repository.getById(id)
    }
}

class GetItemsUseCase(private val repository: ItemRepository) {
    suspend operator fun invoke(): Result<List<Item>> {
        return repository.getAll()
    }
}
```

## expect/actual (KMP)

플랫폼별 구현에 사용:

```kotlin
// commonMain
expect fun platformName(): String
expect class SecureStorage {
    fun save(key: String, value: String)
    fun get(key: String): String?
}

// androidMain
actual fun platformName(): String = "Android"
actual class SecureStorage {
    actual fun save(key: String, value: String) { /* EncryptedSharedPreferences */ }
    actual fun get(key: String): String? = null /* ... */
}

// iosMain
actual fun platformName(): String = "iOS"
actual class SecureStorage {
    actual fun save(key: String, value: String) { /* Keychain */ }
    actual fun get(key: String): String? = null /* ... */
}
```

## 코루틴 패턴

- ViewModel에서는 `viewModelScope`, 구조화된 하위 작업에는 `coroutineScope` 사용
- cold Flow에서 StateFlow로 변환 시 `stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), initialValue)` 사용
- 자식 실패가 독립적이어야 할 때 `supervisorScope` 사용

## DSL을 활용한 Builder 패턴

```kotlin
class HttpClientConfig {
    var baseUrl: String = ""
    var timeout: Long = 30_000
    private val interceptors = mutableListOf<Interceptor>()

    fun interceptor(block: () -> Interceptor) {
        interceptors.add(block())
    }
}

fun httpClient(block: HttpClientConfig.() -> Unit): HttpClient {
    val config = HttpClientConfig().apply(block)
    return HttpClient(config)
}

// 사용 예
val client = httpClient {
    baseUrl = "https://api.example.com"
    timeout = 15_000
    interceptor { AuthInterceptor(tokenProvider) }
}
```

## 참고

상세한 코루틴 패턴은 skill: `kotlin-coroutines-flows`를 참조하세요.
모듈 및 레이어 패턴은 skill: `android-clean-architecture`를 참조하세요.

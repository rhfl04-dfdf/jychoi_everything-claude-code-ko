---
name: android-clean-architecture
description: Android 및 Kotlin Multiplatform 프로젝트를 위한 Clean Architecture 패턴 — 모듈 구조, 의존성 규칙, UseCase, Repository, 데이터 레이어 패턴.
origin: ECC
---

# Android Clean Architecture

Android 및 KMP 프로젝트를 위한 Clean Architecture 패턴. 모듈 경계, 의존성 역전, UseCase/Repository 패턴, Room, SQLDelight, Ktor를 사용한 데이터 레이어 설계를 다룹니다.

## 활성화 시점

- Android 또는 KMP 프로젝트 모듈 구조화
- UseCase, Repository, 또는 DataSource 구현
- 레이어(도메인, 데이터, 프레젠테이션) 간 데이터 흐름 설계
- Koin 또는 Hilt를 이용한 의존성 주입 설정
- 레이어드 아키텍처에서 Room, SQLDelight, 또는 Ktor 작업

## 모듈 구조

### 권장 레이아웃

```
project/
├── app/                  # Android entry point, DI wiring, Application class
├── core/                 # Shared utilities, base classes, error types
├── domain/               # UseCases, domain models, repository interfaces (pure Kotlin)
├── data/                 # Repository implementations, DataSources, DB, network
├── presentation/         # Screens, ViewModels, UI models, navigation
├── design-system/        # Reusable Compose components, theme, typography
└── feature/              # Feature modules (optional, for larger projects)
    ├── auth/
    ├── settings/
    └── profile/
```

### 의존성 규칙

```
app → presentation, domain, data, core
presentation → domain, design-system, core
data → domain, core
domain → core (or no dependencies)
core → (nothing)
```

**중요**: `domain`은 `data`, `presentation`, 또는 어떤 프레임워크에도 절대 의존해서는 안 됩니다. 순수 Kotlin만 포함합니다.

## 도메인 레이어

### UseCase 패턴

각 UseCase는 하나의 비즈니스 동작을 나타냅니다. 깔끔한 호출 사이트를 위해 `operator fun invoke`를 사용하세요:

```kotlin
class GetItemsByCategoryUseCase(
    private val repository: ItemRepository
) {
    suspend operator fun invoke(category: String): Result<List<Item>> {
        return repository.getItemsByCategory(category)
    }
}

// Flow-based UseCase for reactive streams
class ObserveUserProgressUseCase(
    private val repository: UserRepository
) {
    operator fun invoke(userId: String): Flow<UserProgress> {
        return repository.observeProgress(userId)
    }
}
```

### 도메인 모델

도메인 모델은 순수 Kotlin data class입니다 — 프레임워크 어노테이션 없음:

```kotlin
data class Item(
    val id: String,
    val title: String,
    val description: String,
    val tags: List<String>,
    val status: Status,
    val category: String
)

enum class Status { DRAFT, ACTIVE, ARCHIVED }
```

### Repository 인터페이스

도메인에서 정의하고, 데이터에서 구현합니다:

```kotlin
interface ItemRepository {
    suspend fun getItemsByCategory(category: String): Result<List<Item>>
    suspend fun saveItem(item: Item): Result<Unit>
    fun observeItems(): Flow<List<Item>>
}
```

## 데이터 레이어

### Repository 구현체

로컬과 원격 데이터 소스 간의 조율:

```kotlin
class ItemRepositoryImpl(
    private val localDataSource: ItemLocalDataSource,
    private val remoteDataSource: ItemRemoteDataSource
) : ItemRepository {

    override suspend fun getItemsByCategory(category: String): Result<List<Item>> {
        return runCatching {
            val remote = remoteDataSource.fetchItems(category)
            localDataSource.insertItems(remote.map { it.toEntity() })
            localDataSource.getItemsByCategory(category).map { it.toDomain() }
        }
    }

    override suspend fun saveItem(item: Item): Result<Unit> {
        return runCatching {
            localDataSource.insertItems(listOf(item.toEntity()))
        }
    }

    override fun observeItems(): Flow<List<Item>> {
        return localDataSource.observeAll().map { entities ->
            entities.map { it.toDomain() }
        }
    }
}
```

### 매퍼 패턴

데이터 모델 근처에 확장 함수로 매퍼를 유지하세요:

```kotlin
// In data layer
fun ItemEntity.toDomain() = Item(
    id = id,
    title = title,
    description = description,
    tags = tags.split("|"),
    status = Status.valueOf(status),
    category = category
)

fun ItemDto.toEntity() = ItemEntity(
    id = id,
    title = title,
    description = description,
    tags = tags.joinToString("|"),
    status = status,
    category = category
)
```

### Room 데이터베이스 (Android)

```kotlin
@Entity(tableName = "items")
data class ItemEntity(
    @PrimaryKey val id: String,
    val title: String,
    val description: String,
    val tags: String,
    val status: String,
    val category: String
)

@Dao
interface ItemDao {
    @Query("SELECT * FROM items WHERE category = :category")
    suspend fun getByCategory(category: String): List<ItemEntity>

    @Upsert
    suspend fun upsert(items: List<ItemEntity>)

    @Query("SELECT * FROM items")
    fun observeAll(): Flow<List<ItemEntity>>
}
```

### SQLDelight (KMP)

```sql
-- Item.sq
CREATE TABLE ItemEntity (
    id TEXT NOT NULL PRIMARY KEY,
    title TEXT NOT NULL,
    description TEXT NOT NULL,
    tags TEXT NOT NULL,
    status TEXT NOT NULL,
    category TEXT NOT NULL
);

getByCategory:
SELECT * FROM ItemEntity WHERE category = ?;

upsert:
INSERT OR REPLACE INTO ItemEntity (id, title, description, tags, status, category)
VALUES (?, ?, ?, ?, ?, ?);

observeAll:
SELECT * FROM ItemEntity;
```

### Ktor 네트워크 클라이언트 (KMP)

```kotlin
class ItemRemoteDataSource(private val client: HttpClient) {

    suspend fun fetchItems(category: String): List<ItemDto> {
        return client.get("api/items") {
            parameter("category", category)
        }.body()
    }
}

// HttpClient setup with content negotiation
val httpClient = HttpClient {
    install(ContentNegotiation) { json(Json { ignoreUnknownKeys = true }) }
    install(Logging) { level = LogLevel.HEADERS }
    defaultRequest { url("https://api.example.com/") }
}
```

## 의존성 주입

### Koin (KMP 친화적)

```kotlin
// Domain module
val domainModule = module {
    factory { GetItemsByCategoryUseCase(get()) }
    factory { ObserveUserProgressUseCase(get()) }
}

// Data module
val dataModule = module {
    single<ItemRepository> { ItemRepositoryImpl(get(), get()) }
    single { ItemLocalDataSource(get()) }
    single { ItemRemoteDataSource(get()) }
}

// Presentation module
val presentationModule = module {
    viewModelOf(::ItemListViewModel)
    viewModelOf(::DashboardViewModel)
}
```

### Hilt (Android 전용)

```kotlin
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    @Binds
    abstract fun bindItemRepository(impl: ItemRepositoryImpl): ItemRepository
}

@HiltViewModel
class ItemListViewModel @Inject constructor(
    private val getItems: GetItemsByCategoryUseCase
) : ViewModel()
```

## 에러 처리

### Result/Try 패턴

에러 전파를 위해 `Result<T>` 또는 커스텀 sealed 타입을 사용하세요:

```kotlin
sealed interface Try<out T> {
    data class Success<T>(val value: T) : Try<T>
    data class Failure(val error: AppError) : Try<Nothing>
}

sealed interface AppError {
    data class Network(val message: String) : AppError
    data class Database(val message: String) : AppError
    data object Unauthorized : AppError
}

// In ViewModel — map to UI state
viewModelScope.launch {
    when (val result = getItems(category)) {
        is Try.Success -> _state.update { it.copy(items = result.value, isLoading = false) }
        is Try.Failure -> _state.update { it.copy(error = result.error.toMessage(), isLoading = false) }
    }
}
```

## Convention Plugins (Gradle)

KMP 프로젝트의 경우, 빌드 파일 중복을 줄이기 위해 convention plugins를 사용하세요:

```kotlin
// build-logic/src/main/kotlin/kmp-library.gradle.kts
plugins {
    id("org.jetbrains.kotlin.multiplatform")
}

kotlin {
    androidTarget()
    iosX64(); iosArm64(); iosSimulatorArm64()
    sourceSets {
        commonMain.dependencies { /* shared deps */ }
        commonTest.dependencies { implementation(kotlin("test")) }
    }
}
```

모듈에서 적용:

```kotlin
// domain/build.gradle.kts
plugins { id("kmp-library") }
```

## 피해야 할 안티패턴

- `domain`에서 Android 프레임워크 클래스 임포트 — 순수 Kotlin 유지
- UI 레이어에 데이터베이스 엔티티나 DTO 노출 — 항상 도메인 모델로 매핑
- ViewModel에 비즈니스 로직 배치 — UseCase로 추출
- `GlobalScope` 또는 비구조적 코루틴 사용 — `viewModelScope` 또는 구조적 동시성 사용
- 거대한 Repository 구현체 — 집중된 DataSource로 분리
- 순환 모듈 의존성 — A가 B에 의존한다면 B는 A에 의존해서는 안 됨

## 참고

스킬: `compose-multiplatform-patterns` — UI 패턴 참고.
스킬: `kotlin-coroutines-flows` — 비동기 패턴 참고.

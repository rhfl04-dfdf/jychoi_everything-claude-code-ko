---
name: kotlin-patterns
description: 코루틴, null safety, DSL 빌더를 사용하여 견고하고 효율적이며 유지보수가 용이한 Kotlin 애플리케이션을 구축하기 위한 관용적인 Kotlin 패턴, best practices 및 컨벤션입니다.
origin: ECC
---

# Kotlin 개발 패턴

견고하고 효율적이며 유지보수가 용이한 애플리케이션을 구축하기 위한 관용적인 Kotlin 패턴 및 best practices입니다.

## 사용 시기

- 새로운 Kotlin 코드 작성 시
- Kotlin 코드 리뷰 시
- 기존 Kotlin 코드 refactor 시
- Kotlin 모듈 또는 라이브러리 설계 시
- Gradle Kotlin DSL 빌드 구성 시

## 작동 방식

이 skill은 일곱 가지 핵심 영역에서 관용적인 Kotlin 컨벤션을 적용합니다: 타입 시스템과 safe-call 연산자를 사용한 null safety, 데이터 클래스의 `val` 및 `copy()`를 통한 불변성(immutability), 철저한 타입 계층 구조를 위한 sealed 클래스 및 인터페이스, 코루틴과 `Flow`를 사용한 structured concurrency, 상속 없이 동작을 추가하기 위한 extension functions, `@DslMarker`와 람다 수신 객체를 사용한 타입 안전 DSL 빌더, 그리고 빌드 구성을 위한 Gradle Kotlin DSL입니다.

## 예시

**Elvis 연산자를 사용한 null safety:**
```kotlin
fun getUserEmail(userId: String): String {
    val user = userRepository.findById(userId)
    return user?.email ?: "unknown@example.com"
}
```

**철저한 결과를 위한 Sealed 클래스:**
```kotlin
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Failure(val error: AppError) : Result<Nothing>()
    data object Loading : Result<Nothing>()
}
```

**async/await를 사용한 structured concurrency:**
```kotlin
suspend fun fetchUserWithPosts(userId: String): UserProfile =
    coroutineScope {
        val user = async { userService.getUser(userId) }
        val posts = async { postService.getUserPosts(userId) }
        UserProfile(user = user.await(), posts = posts.await())
    }
```

## 핵심 원칙

### 1. Null Safety

Kotlin의 타입 시스템은 nullable과 non-nullable 타입을 구분합니다. 이를 충분히 활용하세요.

```kotlin
// Good: 기본적으로 non-nullable 타입 사용
fun getUser(id: String): User {
    return userRepository.findById(id)
        ?: throw UserNotFoundException("User $id not found")
}

// Good: Safe call 및 Elvis 연산자
fun getUserEmail(userId: String): String {
    val user = userRepository.findById(userId)
    return user?.email ?: "unknown@example.com"
}

// Bad: Nullable 타입 강제 언래핑
fun getUserEmail(userId: String): String {
    val user = userRepository.findById(userId)
    return user!!.email // null일 경우 NPE 발생
}
```

### 2. 기본적으로 불변성(Immutability) 유지

`var`보다 `val`을, 가변 컬렉션보다 불변 컬렉션을 선호하세요.

```kotlin
// Good: 불변 데이터
data class User(
    val id: String,
    val name: String,
    val email: String,
)

// Good: copy()를 사용한 변환
fun updateEmail(user: User, newEmail: String): User =
    user.copy(email = newEmail)

// Good: 불변 컬렉션
val users: List<User> = listOf(user1, user2)
val filtered = users.filter { it.email.isNotBlank() }

// Bad: 가변 상태
var currentUser: User? = null // 가변 전역 상태를 피하세요
val mutableUsers = mutableListOf<User>() // 정말 필요한 경우가 아니면 피하세요
```

### 3. 표현식 본문 및 단일 표현식 함수

간결하고 가독성 좋은 함수를 위해 표현식 본문을 사용하세요.

```kotlin
// Good: 표현식 본문
fun isAdult(age: Int): Boolean = age >= 18

fun formatFullName(first: String, last: String): String =
    "$first $last".trim()

fun User.displayName(): String =
    name.ifBlank { email.substringBefore('@') }

// Good: 표현식으로서의 when
fun statusMessage(code: Int): String = when (code) {
    200 -> "OK"
    404 -> "Not Found"
    500 -> "Internal Server Error"
    else -> "Unknown status: $code"
}

// Bad: 불필요한 블록 본문
fun isAdult(age: Int): Boolean {
    return age >= 18
}
```

### 4. Value Object를 위한 데이터 클래스

주로 데이터를 보유하는 타입에는 데이터 클래스를 사용하세요.

```kotlin
// Good: copy, equals, hashCode, toString이 포함된 데이터 클래스
data class CreateUserRequest(
    val name: String,
    val email: String,
    val role: Role = Role.USER,
)

// Good: 타입 안전성을 위한 value 클래스 (런타임 오버헤드 없음)
@JvmInline
value class UserId(val value: String) {
    init {
        require(value.isNotBlank()) { "UserId cannot be blank" }
    }
}

@JvmInline
value class Email(val value: String) {
    init {
        require('@' in value) { "Invalid email: $value" }
    }
}

fun getUser(id: UserId): User = userRepository.findById(id)
```

## Sealed 클래스 및 인터페이스

### 제한된 계층 구조 모델링

```kotlin
// Good: 철저한 when 처리를 위한 Sealed 클래스
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Failure(val error: AppError) : Result<Nothing>()
    data object Loading : Result<Nothing>()
}

fun <T> Result<T>.getOrNull(): T? = when (this) {
    is Result.Success -> data
    is Result.Failure -> null
    is Result.Loading -> null
}

fun <T> Result<T>.getOrThrow(): T = when (this) {
    is Result.Success -> data
    is Result.Failure -> throw error.toException()
    is Result.Loading -> throw IllegalStateException("Still loading")
}
```

### API 응답을 위한 Sealed 인터페이스

```kotlin
sealed interface ApiError {
    val message: String

    data class NotFound(override val message: String) : ApiError
    data class Unauthorized(override val message: String) : ApiError
    data class Validation(
        override val message: String,
        val field: String,
    ) : ApiError
    data class Internal(
        override val message: String,
        val cause: Throwable? = null,
    ) : ApiError
}

fun ApiError.toStatusCode(): Int = when (this) {
    is ApiError.NotFound -> 404
    is ApiError.Unauthorized -> 401
    is ApiError.Validation -> 422
    is ApiError.Internal -> 500
}
```

## Scope Functions

### 각 함수별 사용 시기

```kotlin
// let: nullable 또는 스코프 결과를 변환
val length: Int? = name?.let { it.trim().length }

// apply: 객체 구성 (객체 자신을 반환)
val user = User().apply {
    name = "Alice"
    email = "alice@example.com"
}

// also: 부수 효과(side effects) (객체 자신을 반환)
val user = createUser(request).also { logger.info("Created user: ${it.id}") }

// run: 수신 객체와 함께 블록 실행 (결과 반환)
val result = connection.run {
    prepareStatement(sql)
    executeQuery()
}

// with: run의 비확장 함수 형태
val csv = with(StringBuilder()) {
    appendLine("name,email")
    users.forEach { appendLine("${it.name},${it.email}") }
    toString()
}
```

### Anti-Patterns

```kotlin
// Bad: 중첩된 scope functions
user?.let { u ->
    u.address?.let { addr ->
        addr.city?.let { city ->
            println(city) // 가독성이 떨어짐
        }
    }
}

// Good: 대신 safe call 체인 사용
val city = user?.address?.city
city?.let { println(it) }
```

## Extension Functions

### 상속 없이 기능 추가하기

```kotlin
// Good: 도메인 특화 확장
fun String.toSlug(): String =
    lowercase()
        .replace(Regex("[^a-z0-9\\s-]"), "")
        .replace(Regex("\\s+"), "-")
        .trim('-')

fun Instant.toLocalDate(zone: ZoneId = ZoneId.systemDefault()): LocalDate =
    atZone(zone).toLocalDate()

// Good: 컬렉션 확장
fun <T> List<T>.second(): T = this[1]

fun <T> List<T>.secondOrNull(): T? = getOrNull(1)

// Good: 스코프 확장 (전역 네임스페이스를 오염시키지 않음)
class UserService {
    private fun User.isActive(): Boolean =
        status == Status.ACTIVE && lastLogin.isAfter(Instant.now().minus(30, ChronoUnit.DAYS))

    fun getActiveUsers(): List<User> = userRepository.findAll().filter { it.isActive() }
}
```

## 코루틴

### Structured Concurrency

```kotlin
// Good: coroutineScope를 사용한 structured concurrency
suspend fun fetchUserWithPosts(userId: String): UserProfile =
    coroutineScope {
        val userDeferred = async { userService.getUser(userId) }
        val postsDeferred = async { postService.getUserPosts(userId) }

        UserProfile(
            user = userDeferred.await(),
            posts = postsDeferred.await(),
        )
    }

// Good: 자식 코루틴이 독립적으로 실패할 수 있을 때의 supervisorScope
suspend fun fetchDashboard(userId: String): Dashboard =
    supervisorScope {
        val user = async { userService.getUser(userId) }
        val notifications = async { notificationService.getRecent(userId) }
        val recommendations = async { recommendationService.getFor(userId) }

        Dashboard(
            user = user.await(),
            notifications = try {
                notifications.await()
            } catch (e: CancellationException) {
                throw e
            } catch (e: Exception) {
                emptyList()
            },
            recommendations = try {
                recommendations.await()
            } catch (e: CancellationException) {
                throw e
            } catch (e: Exception) {
                emptyList()
            },
        )
    }
```

### 리액티브 스트림을 위한 Flow

```kotlin
// Good: 적절한 에러 핸들링을 포함한 cold flow
fun observeUsers(): Flow<List<User>> = flow {
    while (currentCoroutineContext().isActive) {
        val users = userRepository.findAll()
        emit(users)
        delay(5.seconds)
    }
}.catch { e ->
    logger.error("Error observing users", e)
    emit(emptyList())
}

// Good: Flow 연산자
fun searchUsers(query: Flow<String>): Flow<List<User>> =
    query
        .debounce(300.milliseconds)
        .distinctUntilChanged()
        .filter { it.length >= 2 }
        .mapLatest { q -> userRepository.search(q) }
        .catch { emit(emptyList()) }
```

### 취소 및 정리(Cleanup)

```kotlin
// Good: 취소 존중
suspend fun processItems(items: List<Item>) {
    items.forEach { item ->
        ensureActive() // 비용이 많이 드는 작업 전에 취소 여부 확인
        processItem(item)
    }
}

// Good: try/finally를 사용한 정리
suspend fun acquireAndProcess() {
    val resource = acquireResource()
    try {
        resource.process()
    } finally {
        withContext(NonCancellable) {
            resource.release() // 취소 시에도 항상 해제
        }
    }
}
```

## 위임(Delegation)

### 프로퍼티 위임

```kotlin
// 지연 초기화
val expensiveData: List<User> by lazy {
    userRepository.findAll()
}

// 관찰 가능한 프로퍼티
var name: String by Delegates.observable("initial") { _, old, new ->
    logger.info("Name changed from '$old' to '$new'")
}

// Map 기반 프로퍼티
class Config(private val map: Map<String, Any?>) {
    val host: String by map
    val port: Int by map
    val debug: Boolean by map
}

val config = Config(mapOf("host" to "localhost", "port" to 8080, "debug" to true))
```

### 인터페이스 위임

```kotlin
// Good: 인터페이스 구현 위임
class LoggingUserRepository(
    private val delegate: UserRepository,
    private val logger: Logger,
) : UserRepository by delegate {
    // 로깅을 추가해야 하는 부분만 오버라이드
    override suspend fun findById(id: String): User? {
        logger.info("Finding user by id: $id")
        return delegate.findById(id).also {
            logger.info("Found user: ${it?.name ?: "null"}")
        }
    }
}
```

## DSL 빌더

### 타입 안전 빌더

```kotlin
// Good: @DslMarker를 사용한 DSL
@DslMarker
annotation class HtmlDsl

@HtmlDsl
class HTML {
    private val children = mutableListOf<Element>()

    fun head(init: Head.() -> Unit) {
        children += Head().apply(init)
    }

    fun body(init: Body.() -> Unit) {
        children += Body().apply(init)
    }

    override fun toString(): String = children.joinToString("\n")
}

fun html(init: HTML.() -> Unit): HTML = HTML().apply(init)

// 사용 예시
val page = html {
    head { title("My Page") }
    body {
        h1("Welcome")
        p("Hello, World!")
    }
}
```

### 구성(Configuration) DSL

```kotlin
data class ServerConfig(
    val host: String = "0.0.0.0",
    val port: Int = 8080,
    val ssl: SslConfig? = null,
    val database: DatabaseConfig? = null,
)

data class SslConfig(val certPath: String, val keyPath: String)
data class DatabaseConfig(val url: String, val maxPoolSize: Int = 10)

class ServerConfigBuilder {
    var host: String = "0.0.0.0"
    var port: Int = 8080
    private var ssl: SslConfig? = null
    private var database: DatabaseConfig? = null

    fun ssl(certPath: String, keyPath: String) {
        ssl = SslConfig(certPath, keyPath)
    }

    fun database(url: String, maxPoolSize: Int = 10) {
        database = DatabaseConfig(url, maxPoolSize)
    }

    fun build(): ServerConfig = ServerConfig(host, port, ssl, database)
}

fun serverConfig(init: ServerConfigBuilder.() -> Unit): ServerConfig =
    ServerConfigBuilder().apply(init).build()

// 사용 예시
val config = serverConfig {
    host = "0.0.0.0"
    port = 443
    ssl("/certs/cert.pem", "/certs/key.pem")
    database("jdbc:postgresql://localhost:5432/mydb", maxPoolSize = 20)
}
```

## 지연 평가를 위한 Sequences

```kotlin
// Good: 여러 연산이 있는 대규모 컬렉션에 sequence 사용
val result = users.asSequence()
    .filter { it.isActive }
    .map { it.email }
    .filter { it.endsWith("@company.com") }
    .take(10)
    .toList()

// Good: 무한 sequence 생성
val fibonacci: Sequence<Long> = sequence {
    var a = 0L
    var b = 1L
    while (true) {
        yield(a)
        val next = a + b
        a = b
        b = next
    }
}

val first20 = fibonacci.take(20).toList()
```

## Gradle Kotlin DSL

### build.gradle.kts 구성

```kotlin
// 최신 버전 확인: https://kotlinlang.org/docs/releases.html
plugins {
    kotlin("jvm") version "2.3.10"
    kotlin("plugin.serialization") version "2.3.10"
    id("io.gitlab.arturbosch.detekt") version "1.23.8"
    id("io.ktor.plugin") version "3.4.0"
    id("org.jetbrains.kotlinx.kover") version "0.9.7"
}

group = "com.example"
version = "1.0.0"

kotlin {
    jvmToolchain(21)
}

dependencies {
    // Ktor
    implementation("io.ktor:ktor-server-core:3.4.0")
    implementation("io.ktor:ktor-server-netty:3.4.0")
    implementation("io.ktor:ktor-server-content-negotiation:3.4.0")
    implementation("io.ktor:ktor-serialization-kotlinx-json:3.4.0")

    // Exposed
    implementation("org.jetbrains.exposed:exposed-core:1.0.0")
    implementation("org.jetbrains.exposed:exposed-dao:1.0.0")
    implementation("org.jetbrains.exposed:exposed-jdbc:1.0.0")
    implementation("org.jetbrains.exposed:exposed-kotlin-datetime:1.0.0")

    // Koin
    implementation("io.insert-koin:koin-ktor:4.2.0")

    // Coroutines
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.10.2")

    // Testing
    testImplementation("io.kotest:kotest-runner-junit5:6.1.4")
    testImplementation("io.kotest:kotest-assertions-core:6.1.4")
    testImplementation("io.kotest:kotest-property:6.1.4")
    testImplementation("io.mockk:mockk:1.14.9")
    testImplementation("io.ktor:ktor-server-test-host:3.4.0")
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.10.2")
}

tasks.withType<Test> {
    useJUnitPlatform()
}

detekt {
    config.setFrom(files("config/detekt/detekt.yml"))
    buildUponDefaultConfig = true
}
```

## 에러 핸들링 패턴

### 도메인 작업을 위한 Result 타입

```kotlin
// Good: Kotlin의 Result 또는 커스텀 sealed 클래스 사용
suspend fun createUser(request: CreateUserRequest): Result<User> = runCatching {
    require(request.name.isNotBlank()) { "Name cannot be blank" }
    require('@' in request.email) { "Invalid email format" }

    val user = User(
        id = UserId(UUID.randomUUID().toString()),
        name = request.name,
        email = Email(request.email),
    )
    userRepository.save(user)
    user
}

// Good: 결과 체이닝
val displayName = createUser(request)
    .map { it.name }
    .getOrElse { "Unknown" }
```

### require, check, error

```kotlin
// Good: 명확한 메시지를 포함한 선행 조건(preconditions)
fun withdraw(account: Account, amount: Money): Account {
    require(amount.value > 0) { "Amount must be positive: $amount" }
    check(account.balance >= amount) { "Insufficient balance: ${account.balance} < $amount" }

    return account.copy(balance = account.balance - amount)
}
```

## 컬렉션 연산

### 관용적인 컬렉션 처리

```kotlin
// Good: 체이닝된 연산
val activeAdminEmails: List<String> = users
    .filter { it.role == Role.ADMIN && it.isActive }
    .sortedBy { it.name }
    .map { it.email }

// Good: 그룹화 및 집계
val usersByRole: Map<Role, List<User>> = users.groupBy { it.role }

val oldestByRole: Map<Role, User?> = users.groupBy { it.role }
    .mapValues { (_, users) -> users.minByOrNull { it.createdAt } }

// Good: 맵 생성을 위한 associate
val usersById: Map<UserId, User> = users.associateBy { it.id }

// Good: 분할을 위한 partition
val (active, inactive) = users.partition { it.isActive }
```

## 빠른 참조: Kotlin 관용구

| 관용구 | 설명 |
|-------|-------------|
| `val` over `var` | 불변 변수 선호 |
| `data class` | equals/hashCode/copy가 포함된 value objects용 |
| `sealed class/interface` | 제한된 타입 계층 구조용 |
| `value class` | 오버헤드가 없는 타입 안전 래퍼용 |
| 표현식 `when` | 철저한 패턴 매칭 |
| Safe call `?.` | null 안전 멤버 액세스 |
| Elvis `?:` | nullable을 위한 기본값 |
| `let`/`apply`/`also`/`run`/`with` | 깔끔한 코드를 위한 Scope functions |
| Extension functions | 상속 없이 동작 추가 |
| `copy()` | 데이터 클래스의 불변 업데이트 |
| `require`/`check` | 선행 조건 단언(assertions) |
| Coroutine `async`/`await` | Structured concurrent 실행 |
| `Flow` | Cold 리액티브 스트림 |
| `sequence` | 지연 평가 |
| 위임 `by` | 상속 없이 구현 재사용 |

## 피해야 할 Anti-Patterns

```kotlin
// Bad: nullable 타입 강제 언래핑
val name = user!!.name

// Bad: Java로부터의 플랫폼 타입 누출
fun getLength(s: String) = s.length // 안전함
fun getLength(s: String?) = s?.length ?: 0 // Java에서 넘어온 null 처리

// Bad: 가변 데이터 클래스
data class MutableUser(var name: String, var email: String)

// Bad: 제어 흐름을 위해 예외 사용
try {
    val user = findUser(id)
} catch (e: NotFoundException) {
    // 예상되는 상황에 예외를 사용하지 마세요
}

// Good: Nullable 반환 또는 Result 사용
val user: User? = findUserOrNull(id)

// Bad: 코루틴 스코프 무시
GlobalScope.launch { /* GlobalScope를 피하세요 */ }

// Good: structured concurrency 사용
coroutineScope {
    launch { /* 적절한 스코프 내에서 실행 */ }
}

// Bad: 깊게 중첩된 scope functions
user?.let { u ->
    u.address?.let { a ->
        a.city?.let { c -> process(c) }
    }
}

// Good: 직접적인 null 안전 체인
user?.address?.city?.let { process(it) }
```

**기억하세요**: Kotlin 코드는 간결하면서도 가독성이 있어야 합니다. 안전성을 위해 타입 시스템을 활용하고, 불변성을 선호하며, 동시성을 위해 코루틴을 사용하세요. 의심스러울 때는 컴파일러의 도움을 받으세요.

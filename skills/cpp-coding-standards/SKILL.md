---
name: cpp-coding-standards
description: C++ Core Guidelines(isocpp.github.io)에 기반한 C++ 코딩 표준. C++ 코드 작성, 검토, 리팩토링 시 현대적이고 안전하며 관용적인 방법을 적용할 때 사용합니다.
origin: ECC
---

# C++ 코딩 표준 (C++ Core Guidelines)

[C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines)에서 파생된 현대 C++(C++17/20/23)의 포괄적 코딩 표준. 타입 안전성, 리소스 안전성, 불변성, 명확성을 강제합니다.

## 사용 시점

- 새 C++ 코드 작성 (클래스, 함수, 템플릿)
- 기존 C++ 코드 검토 또는 리팩토링
- C++ 프로젝트에서 아키텍처 결정 시
- C++ 코드베이스 전반에 일관된 스타일 적용 시
- 언어 기능 선택 시 (예: `enum` vs `enum class`, 원시 포인터 vs 스마트 포인터)

### 사용하지 않을 때

- C++ 외 프로젝트
- 현대 C++ 기능을 도입할 수 없는 레거시 C 코드베이스
- 특정 가이드라인이 하드웨어 제약과 충돌하는 임베디드/베어 메탈 환경 (선택적으로 적용)

## 횡단 관심사 원칙

이 주제들은 전체 가이드라인에 걸쳐 반복되며 기반을 형성합니다:

1. **어디서나 RAII** (P.8, R.1, E.6, CP.20): 리소스 수명을 객체 수명에 바인딩
2. **기본적으로 불변성** (P.10, Con.1-5, ES.25): `const`/`constexpr`로 시작; 가변성은 예외
3. **타입 안전성** (P.4, I.4, ES.46-49, Enum.3): 타입 시스템을 사용하여 컴파일 타임에 오류 방지
4. **의도 표현** (P.3, F.1, NL.1-2, T.10): 이름, 타입, 개념이 목적을 전달해야 함
5. **복잡성 최소화** (F.2-3, ES.5, Per.4-5): 단순한 코드가 올바른 코드
6. **포인터 시맨틱보다 값 시맨틱** (C.10, R.3-5, F.20, CP.31): 값 반환과 스코프 객체 선호

## 철학 & 인터페이스 (P.*, I.*)

### 주요 규칙

| 규칙 | 요약 |
|------|---------|
| **P.1** | 코드에서 직접 아이디어를 표현 |
| **P.3** | 의도를 표현 |
| **P.4** | 이상적으로, 프로그램은 정적 타입 안전해야 함 |
| **P.5** | 런타임 검사보다 컴파일 타임 검사 선호 |
| **P.8** | 리소스를 누출하지 않음 |
| **P.10** | 가변 데이터보다 불변 데이터 선호 |
| **I.1** | 인터페이스를 명시적으로 만들기 |
| **I.2** | 비const 전역 변수 피하기 |
| **I.4** | 인터페이스를 정밀하고 강하게 타입화 |
| **I.11** | 원시 포인터나 참조로 소유권 이전 금지 |
| **I.23** | 함수 인수 수를 낮게 유지 |

### 권장 사항

```cpp
// P.10 + I.4: Immutable, strongly typed interface
struct Temperature {
    double kelvin;
};

Temperature boil(const Temperature& water);
```

### 금지 사항

```cpp
// Weak interface: unclear ownership, unclear units
double boil(double* temp);

// Non-const global variable
int g_counter = 0;  // I.2 violation
```

## 함수 (F.*)

### 주요 규칙

| 규칙 | 요약 |
|------|---------|
| **F.1** | 의미 있는 연산을 신중하게 명명된 함수로 패키징 |
| **F.2** | 함수는 단일 논리적 연산 수행 |
| **F.3** | 함수를 짧고 단순하게 유지 |
| **F.4** | 컴파일 타임에 평가될 수 있으면 `constexpr` 선언 |
| **F.6** | throw하지 않아야 한다면 `noexcept` 선언 |
| **F.8** | 순수 함수 선호 |
| **F.16** | "in" 매개변수는 저렴하게 복사되는 타입은 값으로, 나머지는 `const&`로 전달 |
| **F.20** | "out" 값은 출력 매개변수보다 반환값 선호 |
| **F.21** | 여러 "out" 값 반환 시 struct 반환 선호 |
| **F.43** | 로컬 객체에 대한 포인터나 참조를 절대 반환하지 않기 |

### 매개변수 전달

```cpp
// F.16: Cheap types by value, others by const&
void print(int x);                           // cheap: by value
void analyze(const std::string& data);       // expensive: by const&
void transform(std::string s);               // sink: by value (will move)

// F.20 + F.21: Return values, not output parameters
struct ParseResult {
    std::string token;
    int position;
};

ParseResult parse(std::string_view input);   // GOOD: return struct

// BAD: output parameters
void parse(std::string_view input,
           std::string& token, int& pos);    // avoid this
```

### 순수 함수와 constexpr

```cpp
// F.4 + F.8: Pure, constexpr where possible
constexpr int factorial(int n) noexcept {
    return (n <= 1) ? 1 : n * factorial(n - 1);
}

static_assert(factorial(5) == 120);
```

### 안티패턴

- 함수에서 `T&&` 반환 (F.45)
- `va_arg` / C 스타일 가변 인자 사용 (F.55)
- 다른 스레드로 전달되는 람다에서 참조로 캡처 (F.53)
- 이동 시맨틱을 방해하는 `const T` 반환 (F.49)

## 클래스 & 클래스 계층 (C.*)

### 주요 규칙

| 규칙 | 요약 |
|------|---------|
| **C.2** | 불변식이 있으면 `class` 사용; 데이터 멤버가 독립적으로 변하면 `struct` |
| **C.9** | 멤버 노출 최소화 |
| **C.20** | 기본 연산을 정의하지 않아도 된다면 그렇게 하기 (영의 규칙) |
| **C.21** | 복사/이동/소멸자 중 하나라도 정의하거나 `=delete`하면 모두 처리 (다섯의 규칙) |
| **C.35** | 기반 클래스 소멸자: public virtual 또는 protected non-virtual |
| **C.41** | 생성자는 완전히 초기화된 객체를 생성해야 함 |
| **C.46** | 단일 인수 생성자는 `explicit` 선언 |
| **C.67** | 다형 클래스는 public 복사/이동을 억제해야 함 |
| **C.128** | 가상 함수: `virtual`, `override`, `final` 중 정확히 하나만 지정 |

### 영의 규칙

```cpp
// C.20: Let the compiler generate special members
struct Employee {
    std::string name;
    std::string department;
    int id;
    // No destructor, copy/move constructors, or assignment operators needed
};
```

### 다섯의 규칙

```cpp
// C.21: If you must manage a resource, define all five
class Buffer {
public:
    explicit Buffer(std::size_t size)
        : data_(std::make_unique<char[]>(size)), size_(size) {}

    ~Buffer() = default;

    Buffer(const Buffer& other)
        : data_(std::make_unique<char[]>(other.size_)), size_(other.size_) {
        std::copy_n(other.data_.get(), size_, data_.get());
    }

    Buffer& operator=(const Buffer& other) {
        if (this != &other) {
            auto new_data = std::make_unique<char[]>(other.size_);
            std::copy_n(other.data_.get(), other.size_, new_data.get());
            data_ = std::move(new_data);
            size_ = other.size_;
        }
        return *this;
    }

    Buffer(Buffer&&) noexcept = default;
    Buffer& operator=(Buffer&&) noexcept = default;

private:
    std::unique_ptr<char[]> data_;
    std::size_t size_;
};
```

### 클래스 계층

```cpp
// C.35 + C.128: Virtual destructor, use override
class Shape {
public:
    virtual ~Shape() = default;
    virtual double area() const = 0;  // C.121: pure interface
};

class Circle : public Shape {
public:
    explicit Circle(double r) : radius_(r) {}
    double area() const override { return 3.14159 * radius_ * radius_; }

private:
    double radius_;
};
```

### 안티패턴

- 생성자/소멸자에서 가상 함수 호출 (C.82)
- 비자명 타입에 `memset`/`memcpy` 사용 (C.90)
- 가상 함수와 오버라이더에 다른 기본 인수 제공 (C.140)
- 이동/복사를 억제하는 `const` 또는 참조 데이터 멤버 (C.12)

## 리소스 관리 (R.*)

### 주요 규칙

| 규칙 | 요약 |
|------|---------|
| **R.1** | RAII를 사용하여 자동으로 리소스 관리 |
| **R.3** | 원시 포인터(`T*`)는 소유하지 않음 |
| **R.5** | 스코프 객체 선호; 불필요하게 힙 할당하지 않기 |
| **R.10** | `malloc()`/`free()` 피하기 |
| **R.11** | 명시적으로 `new`와 `delete` 호출 피하기 |
| **R.20** | `unique_ptr` 또는 `shared_ptr`를 사용하여 소유권 표현 |
| **R.21** | 소유권 공유가 아니라면 `shared_ptr`보다 `unique_ptr` 선호 |
| **R.22** | `make_shared()`를 사용하여 `shared_ptr` 생성 |

### 스마트 포인터 사용

```cpp
// R.11 + R.20 + R.21: RAII with smart pointers
auto widget = std::make_unique<Widget>("config");  // unique ownership
auto cache  = std::make_shared<Cache>(1024);        // shared ownership

// R.3: Raw pointer = non-owning observer
void render(const Widget* w) {  // does NOT own w
    if (w) w->draw();
}

render(widget.get());
```

### RAII 패턴

```cpp
// R.1: Resource acquisition is initialization
class FileHandle {
public:
    explicit FileHandle(const std::string& path)
        : handle_(std::fopen(path.c_str(), "r")) {
        if (!handle_) throw std::runtime_error("Failed to open: " + path);
    }

    ~FileHandle() {
        if (handle_) std::fclose(handle_);
    }

    FileHandle(const FileHandle&) = delete;
    FileHandle& operator=(const FileHandle&) = delete;
    FileHandle(FileHandle&& other) noexcept
        : handle_(std::exchange(other.handle_, nullptr)) {}
    FileHandle& operator=(FileHandle&& other) noexcept {
        if (this != &other) {
            if (handle_) std::fclose(handle_);
            handle_ = std::exchange(other.handle_, nullptr);
        }
        return *this;
    }

private:
    std::FILE* handle_;
};
```

### 안티패턴

- 벌거벗은 `new`/`delete` (R.11)
- C++ 코드에서 `malloc()`/`free()` (R.10)
- 단일 표현식에서 여러 리소스 할당 (R.13 -- 예외 안전성 위험)
- `unique_ptr`으로 충분한 곳에 `shared_ptr` (R.21)

## 표현식 & 문 (ES.*)

### 주요 규칙

| 규칙 | 요약 |
|------|---------|
| **ES.5** | 스코프를 작게 유지 |
| **ES.20** | 객체를 항상 초기화 |
| **ES.23** | `{}` 초기화 문법 선호 |
| **ES.25** | 수정이 의도되지 않으면 `const` 또는 `constexpr`로 선언 |
| **ES.28** | `const` 변수의 복잡한 초기화에 람다 사용 |
| **ES.45** | 매직 상수 피하기; 기호 상수 사용 |
| **ES.46** | 좁히기/손실 산술 변환 피하기 |
| **ES.47** | `0`이나 `NULL` 대신 `nullptr` 사용 |
| **ES.48** | 캐스트 피하기 |
| **ES.50** | `const`를 캐스트로 제거하지 않기 |

### 초기화

```cpp
// ES.20 + ES.23 + ES.25: Always initialize, prefer {}, default to const
const int max_retries{3};
const std::string name{"widget"};
const std::vector<int> primes{2, 3, 5, 7, 11};

// ES.28: Lambda for complex const initialization
const auto config = [&] {
    Config c;
    c.timeout = std::chrono::seconds{30};
    c.retries = max_retries;
    c.verbose = debug_mode;
    return c;
}();
```

### 안티패턴

- 초기화되지 않은 변수 (ES.20)
- 포인터로 `0`이나 `NULL` 사용 (ES.47 -- `nullptr` 사용)
- C 스타일 캐스트 (ES.48 -- `static_cast`, `const_cast` 등 사용)
- `const` 제거 캐스트 (ES.50)
- 명명된 상수 없는 매직 숫자 (ES.45)
- 부호 있는 정수와 부호 없는 정수 혼합 산술 (ES.100)
- 중첩 스코프에서 이름 재사용 (ES.12)

## 오류 처리 (E.*)

### 주요 규칙

| 규칙 | 요약 |
|------|---------|
| **E.1** | 설계 초기에 오류 처리 전략 개발 |
| **E.2** | 함수가 할당된 작업을 수행할 수 없음을 알리기 위해 예외 throw |
| **E.6** | RAII를 사용하여 누출 방지 |
| **E.12** | throw가 불가능하거나 허용되지 않으면 `noexcept` 사용 |
| **E.14** | 목적에 맞게 설계된 사용자 정의 타입을 예외로 사용 |
| **E.15** | 값으로 throw하고, 참조로 catch |
| **E.16** | 소멸자, 해제, swap은 절대 실패해서는 안 됨 |
| **E.17** | 모든 함수에서 모든 예외를 catch하려 하지 않기 |

### 예외 계층

```cpp
// E.14 + E.15: Custom exception types, throw by value, catch by reference
class AppError : public std::runtime_error {
public:
    using std::runtime_error::runtime_error;
};

class NetworkError : public AppError {
public:
    NetworkError(const std::string& msg, int code)
        : AppError(msg), status_code(code) {}
    int status_code;
};

void fetch_data(const std::string& url) {
    // E.2: Throw to signal failure
    throw NetworkError("connection refused", 503);
}

void run() {
    try {
        fetch_data("https://api.example.com");
    } catch (const NetworkError& e) {
        log_error(e.what(), e.status_code);
    } catch (const AppError& e) {
        log_error(e.what());
    }
    // E.17: Don't catch everything here -- let unexpected errors propagate
}
```

### 안티패턴

- `int`나 문자열 리터럴 같은 내장 타입 throw (E.14)
- 값으로 catch (슬라이싱 위험) (E.15)
- 오류를 조용히 삼키는 빈 catch 블록
- 흐름 제어에 예외 사용 (E.3)
- `errno` 같은 전역 상태 기반 오류 처리 (E.28)

## 상수 & 불변성 (Con.*)

### 모든 규칙

| 규칙 | 요약 |
|------|---------|
| **Con.1** | 기본적으로 객체를 불변으로 만들기 |
| **Con.2** | 기본적으로 멤버 함수를 `const`로 만들기 |
| **Con.3** | 기본적으로 포인터와 참조를 `const`로 전달 |
| **Con.4** | 생성 후 변경되지 않는 값에 `const` 사용 |
| **Con.5** | 컴파일 타임에 계산 가능한 값에 `constexpr` 사용 |

```cpp
// Con.1 through Con.5: Immutability by default
class Sensor {
public:
    explicit Sensor(std::string id) : id_(std::move(id)) {}

    // Con.2: const member functions by default
    const std::string& id() const { return id_; }
    double last_reading() const { return reading_; }

    // Only non-const when mutation is required
    void record(double value) { reading_ = value; }

private:
    const std::string id_;  // Con.4: never changes after construction
    double reading_{0.0};
};

// Con.3: Pass by const reference
void display(const Sensor& s) {
    std::cout << s.id() << ": " << s.last_reading() << '\n';
}

// Con.5: Compile-time constants
constexpr double PI = 3.14159265358979;
constexpr int MAX_SENSORS = 256;
```

## 동시성 & 병렬성 (CP.*)

### 주요 규칙

| 규칙 | 요약 |
|------|---------|
| **CP.2** | 데이터 레이스 피하기 |
| **CP.3** | 쓰기 가능한 데이터의 명시적 공유 최소화 |
| **CP.4** | 스레드가 아닌 태스크 기준으로 생각하기 |
| **CP.8** | 동기화에 `volatile` 사용하지 않기 |
| **CP.20** | RAII 사용, 절대 `lock()`/`unlock()` 직접 호출하지 않기 |
| **CP.21** | 여러 뮤텍스 획득 시 `std::scoped_lock` 사용 |
| **CP.22** | 락을 보유한 채 알 수 없는 코드를 호출하지 않기 |
| **CP.42** | 조건 없이 대기하지 않기 |
| **CP.44** | `lock_guard`와 `unique_lock`에 이름 붙이기 |
| **CP.100** | 반드시 필요한 경우가 아니면 락 프리 프로그래밍 사용하지 않기 |

### 안전한 락킹

```cpp
// CP.20 + CP.44: RAII locks, always named
class ThreadSafeQueue {
public:
    void push(int value) {
        std::lock_guard<std::mutex> lock(mutex_);  // CP.44: named!
        queue_.push(value);
        cv_.notify_one();
    }

    int pop() {
        std::unique_lock<std::mutex> lock(mutex_);
        // CP.42: Always wait with a condition
        cv_.wait(lock, [this] { return !queue_.empty(); });
        const int value = queue_.front();
        queue_.pop();
        return value;
    }

private:
    std::mutex mutex_;             // CP.50: mutex with its data
    std::condition_variable cv_;
    std::queue<int> queue_;
};
```

### 여러 뮤텍스

```cpp
// CP.21: std::scoped_lock for multiple mutexes (deadlock-free)
void transfer(Account& from, Account& to, double amount) {
    std::scoped_lock lock(from.mutex_, to.mutex_);
    from.balance_ -= amount;
    to.balance_ += amount;
}
```

### 안티패턴

- 동기화에 `volatile` (CP.8 -- 하드웨어 I/O용)
- 스레드 분리 (CP.26 -- 수명 관리가 거의 불가능해짐)
- 이름 없는 락 가드: `std::lock_guard<std::mutex>(m);`는 즉시 파괴 (CP.44)
- 콜백 호출 중 락 보유 (CP.22 -- 교착 상태 위험)
- 깊은 전문 지식 없는 락 프리 프로그래밍 (CP.100)

## 템플릿 & 제네릭 프로그래밍 (T.*)

### 주요 규칙

| 규칙 | 요약 |
|------|---------|
| **T.1** | 추상화 수준을 높이기 위해 템플릿 사용 |
| **T.2** | 여러 인수 타입에 대한 알고리즘을 표현하기 위해 템플릿 사용 |
| **T.10** | 모든 템플릿 인수에 concept 지정 |
| **T.11** | 가능하면 표준 concept 사용 |
| **T.13** | 단순한 concept에는 단축 표기법 선호 |
| **T.43** | `typedef` 대신 `using` 선호 |
| **T.120** | 정말 필요할 때만 템플릿 메타프로그래밍 사용 |
| **T.144** | 함수 템플릿 특수화 금지 (오버로딩 대신) |

### Concepts (C++20)

```cpp
#include <concepts>

// T.10 + T.11: Constrain templates with standard concepts
template<std::integral T>
T gcd(T a, T b) {
    while (b != 0) {
        a = std::exchange(b, a % b);
    }
    return a;
}

// T.13: Shorthand concept syntax
void sort(std::ranges::random_access_range auto& range) {
    std::ranges::sort(range);
}

// Custom concept for domain-specific constraints
template<typename T>
concept Serializable = requires(const T& t) {
    { t.serialize() } -> std::convertible_to<std::string>;
};

template<Serializable T>
void save(const T& obj, const std::string& path);
```

### 안티패턴

- 가시 네임스페이스에서 제약 없는 템플릿 (T.47)
- 오버로딩 대신 함수 템플릿 특수화 (T.144)
- `constexpr`로 충분한 곳에 템플릿 메타프로그래밍 (T.120)
- `using` 대신 `typedef` (T.43)

## 표준 라이브러리 (SL.*)

### 주요 규칙

| 규칙 | 요약 |
|------|---------|
| **SL.1** | 가능하면 라이브러리 사용 |
| **SL.2** | 다른 라이브러리보다 표준 라이브러리 선호 |
| **SL.con.1** | C 배열보다 `std::array` 또는 `std::vector` 선호 |
| **SL.con.2** | 기본적으로 `std::vector` 선호 |
| **SL.str.1** | 문자 시퀀스 소유에 `std::string` 사용 |
| **SL.str.2** | 문자 시퀀스 참조에 `std::string_view` 사용 |
| **SL.io.50** | `endl` 피하기 (`'\n'` 사용 -- `endl`은 플러시를 강제) |

```cpp
// SL.con.1 + SL.con.2: Prefer vector/array over C arrays
const std::array<int, 4> fixed_data{1, 2, 3, 4};
std::vector<std::string> dynamic_data;

// SL.str.1 + SL.str.2: string owns, string_view observes
std::string build_greeting(std::string_view name) {
    return "Hello, " + std::string(name) + "!";
}

// SL.io.50: Use '\n' not endl
std::cout << "result: " << value << '\n';
```

## 열거형 (Enum.*)

### 주요 규칙

| 규칙 | 요약 |
|------|---------|
| **Enum.1** | 매크로보다 열거형 선호 |
| **Enum.3** | 일반 `enum`보다 `enum class` 선호 |
| **Enum.5** | 열거자에 ALL_CAPS 사용하지 않기 |
| **Enum.6** | 이름 없는 열거형 피하기 |

```cpp
// Enum.3 + Enum.5: Scoped enum, no ALL_CAPS
enum class Color { red, green, blue };
enum class LogLevel { debug, info, warning, error };

// BAD: plain enum leaks names, ALL_CAPS clashes with macros
enum { RED, GREEN, BLUE };           // Enum.3 + Enum.5 + Enum.6 violation
#define MAX_SIZE 100                  // Enum.1 violation -- use constexpr
```

## 소스 파일 & 명명 (SF.*, NL.*)

### 주요 규칙

| 규칙 | 요약 |
|------|---------|
| **SF.1** | 코드 파일에는 `.cpp`, 인터페이스 파일에는 `.h` 사용 |
| **SF.7** | 헤더의 전역 스코프에서 `using namespace` 작성하지 않기 |
| **SF.8** | 모든 `.h` 파일에 `#include` 가드 사용 |
| **SF.11** | 헤더 파일은 자기 완결적이어야 함 |
| **NL.5** | 이름에 타입 정보 인코딩 피하기 (헝가리안 표기법 금지) |
| **NL.8** | 일관된 명명 스타일 사용 |
| **NL.9** | 매크로 이름에만 ALL_CAPS 사용 |
| **NL.10** | `underscore_style` 이름 선호 |

### 헤더 가드

```cpp
// SF.8: Include guard (or #pragma once)
#ifndef PROJECT_MODULE_WIDGET_H
#define PROJECT_MODULE_WIDGET_H

// SF.11: Self-contained -- include everything this header needs
#include <string>
#include <vector>

namespace project::module {

class Widget {
public:
    explicit Widget(std::string name);
    const std::string& name() const;

private:
    std::string name_;
};

}  // namespace project::module

#endif  // PROJECT_MODULE_WIDGET_H
```

### 명명 규칙

```cpp
// NL.8 + NL.10: Consistent underscore_style
namespace my_project {

constexpr int max_buffer_size = 4096;  // NL.9: not ALL_CAPS (it's not a macro)

class tcp_connection {                 // underscore_style class
public:
    void send_message(std::string_view msg);
    bool is_connected() const;

private:
    std::string host_;                 // trailing underscore for members
    int port_;
};

}  // namespace my_project
```

### 안티패턴

- 헤더의 전역 스코프에서 `using namespace std;` (SF.7)
- 인클루드 순서에 의존하는 헤더 (SF.10, SF.11)
- `strName`, `iCount` 같은 헝가리안 표기법 (NL.5)
- 매크로 외에 ALL_CAPS (NL.9)

## 성능 (Per.*)

### 주요 규칙

| 규칙 | 요약 |
|------|---------|
| **Per.1** | 이유 없이 최적화하지 않기 |
| **Per.2** | 조기 최적화하지 않기 |
| **Per.6** | 측정 없이 성능 주장하지 않기 |
| **Per.7** | 최적화를 가능하게 하는 설계 |
| **Per.10** | 정적 타입 시스템에 의존하기 |
| **Per.11** | 계산을 런타임에서 컴파일 타임으로 이동 |
| **Per.19** | 메모리를 예측 가능하게 접근 |

### 가이드라인

```cpp
// Per.11: Compile-time computation where possible
constexpr auto lookup_table = [] {
    std::array<int, 256> table{};
    for (int i = 0; i < 256; ++i) {
        table[i] = i * i;
    }
    return table;
}();

// Per.19: Prefer contiguous data for cache-friendliness
std::vector<Point> points;           // GOOD: contiguous
std::vector<std::unique_ptr<Point>> indirect_points; // BAD: pointer chasing
```

### 안티패턴

- 프로파일링 데이터 없이 최적화 (Per.1, Per.6)
- 명확한 추상화보다 "영리한" 저수준 코드 선택 (Per.4, Per.5)
- 데이터 레이아웃과 캐시 동작 무시 (Per.19)

## 빠른 참조 체크리스트

C++ 작업 완료 전 확인:

- [ ] 원시 `new`/`delete` 없음 -- 스마트 포인터 또는 RAII 사용 (R.11)
- [ ] 선언 시 객체 초기화 (ES.20)
- [ ] 변수는 기본적으로 `const`/`constexpr` (Con.1, ES.25)
- [ ] 가능하면 멤버 함수를 `const`로 (Con.2)
- [ ] 일반 `enum` 대신 `enum class` (Enum.3)
- [ ] `0`/`NULL` 대신 `nullptr` (ES.47)
- [ ] 좁히기 변환 없음 (ES.46)
- [ ] C 스타일 캐스트 없음 (ES.48)
- [ ] 단일 인수 생성자는 `explicit` (C.46)
- [ ] 영의 규칙 또는 다섯의 규칙 적용 (C.20, C.21)
- [ ] 기반 클래스 소멸자는 public virtual 또는 protected non-virtual (C.35)
- [ ] 템플릿은 concept으로 제한 (T.10)
- [ ] 헤더의 전역 스코프에서 `using namespace` 없음 (SF.7)
- [ ] 헤더는 인클루드 가드가 있고 자기 완결적 (SF.8, SF.11)
- [ ] 락은 RAII (`scoped_lock`/`lock_guard`) 사용 (CP.20)
- [ ] 예외는 커스텀 타입, 값으로 throw, 참조로 catch (E.14, E.15)
- [ ] `std::endl` 대신 `'\n'` (SL.io.50)
- [ ] 매직 숫자 없음 (ES.45)

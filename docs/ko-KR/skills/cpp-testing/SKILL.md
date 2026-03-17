---
name: cpp-testing
description: C++ 테스트 작성/업데이트/수정, GoogleTest/CTest 설정, 실패하거나 불안정한 테스트 진단, 커버리지/새니타이저 추가 시에만 사용합니다.
origin: ECC
---

# C++ 테스팅 (에이전트 스킬)

CMake/CTest와 함께 GoogleTest/GoogleMock을 사용하는 현대 C++(C++17/20)의 에이전트 중심 테스팅 워크플로우.

## 사용 시점

- 새 C++ 테스트 작성 또는 기존 테스트 수정
- C++ 컴포넌트의 단위/통합 테스트 커버리지 설계
- 테스트 커버리지, CI 게이팅, 회귀 보호 추가
- 일관된 실행을 위한 CMake/CTest 워크플로우 설정
- 테스트 실패 또는 불안정한 동작 조사
- 메모리/레이스 진단을 위한 새니타이저 활성화

### 사용하지 않을 때

- 테스트 변경 없이 새 제품 기능 구현
- 테스트 커버리지나 실패와 무관한 대규모 리팩토링
- 검증할 테스트 회귀 없는 성능 튜닝
- C++ 외 프로젝트 또는 비테스트 작업

## 핵심 개념

- **TDD 루프**: red → green → refactor (테스트 먼저, 최소 수정, 그 다음 정리)
- **격리**: 전역 상태보다 의존성 주입과 fake 선호
- **테스트 레이아웃**: `tests/unit`, `tests/integration`, `tests/testdata`
- **Mock vs Fake**: 상호작용에는 mock, 상태 있는 동작에는 fake
- **CTest 발견**: 안정적인 테스트 발견을 위해 `gtest_discover_tests()` 사용
- **CI 신호**: 먼저 부분 집합 실행, 그 다음 `--output-on-failure`로 전체 스위트 실행

## TDD 워크플로우

RED → GREEN → REFACTOR 루프를 따릅니다:

1. **RED**: 새 동작을 캡처하는 실패 테스트 작성
2. **GREEN**: 통과를 위한 최소한의 변경 구현
3. **REFACTOR**: 테스트를 통과한 상태로 유지하면서 정리

```cpp
// tests/add_test.cpp
#include <gtest/gtest.h>

int Add(int a, int b); // 프로덕션 코드에서 제공.

TEST(AddTest, AddsTwoNumbers) { // RED
  EXPECT_EQ(Add(2, 3), 5);
}

// src/add.cpp
int Add(int a, int b) { // GREEN
  return a + b;
}

// REFACTOR: 테스트가 통과되면 단순화/이름 변경
```

## 코드 예제

### 기본 단위 테스트 (gtest)

```cpp
// tests/calculator_test.cpp
#include <gtest/gtest.h>

int Add(int a, int b); // 프로덕션 코드에서 제공.

TEST(CalculatorTest, AddsTwoNumbers) {
    EXPECT_EQ(Add(2, 3), 5);
}
```

### 픽스처 (gtest)

```cpp
// tests/user_store_test.cpp
// 의사코드 스텁: UserStore/User를 프로젝트 타입으로 교체하세요.
#include <gtest/gtest.h>
#include <memory>
#include <optional>
#include <string>

struct User { std::string name; };
class UserStore {
public:
    explicit UserStore(std::string /*path*/) {}
    void Seed(std::initializer_list<User> /*users*/) {}
    std::optional<User> Find(const std::string &/*name*/) { return User{"alice"}; }
};

class UserStoreTest : public ::testing::Test {
protected:
    void SetUp() override {
        store = std::make_unique<UserStore>(":memory:");
        store->Seed({{"alice"}, {"bob"}});
    }

    std::unique_ptr<UserStore> store;
};

TEST_F(UserStoreTest, FindsExistingUser) {
    auto user = store->Find("alice");
    ASSERT_TRUE(user.has_value());
    EXPECT_EQ(user->name, "alice");
}
```

### Mock (gmock)

```cpp
// tests/notifier_test.cpp
#include <gmock/gmock.h>
#include <gtest/gtest.h>
#include <string>

class Notifier {
public:
    virtual ~Notifier() = default;
    virtual void Send(const std::string &message) = 0;
};

class MockNotifier : public Notifier {
public:
    MOCK_METHOD(void, Send, (const std::string &message), (override));
};

class Service {
public:
    explicit Service(Notifier &notifier) : notifier_(notifier) {}
    void Publish(const std::string &message) { notifier_.Send(message); }

private:
    Notifier &notifier_;
};

TEST(ServiceTest, SendsNotifications) {
    MockNotifier notifier;
    Service service(notifier);

    EXPECT_CALL(notifier, Send("hello")).Times(1);
    service.Publish("hello");
}
```

### CMake/CTest 빠른 시작

```cmake
# CMakeLists.txt (발췌)
cmake_minimum_required(VERSION 3.20)
project(example LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

include(FetchContent)
# 프로젝트 고정 버전 선호. 태그 사용 시, 프로젝트 정책에 따라 고정 버전 사용.
set(GTEST_VERSION v1.17.0) # 프로젝트 정책에 맞게 조정.
FetchContent_Declare(
  googletest
  # Google Test 프레임워크 (공식 저장소)
  URL https://github.com/google/googletest/archive/refs/tags/${GTEST_VERSION}.zip
)
FetchContent_MakeAvailable(googletest)

add_executable(example_tests
  tests/calculator_test.cpp
  src/calculator.cpp
)
target_link_libraries(example_tests GTest::gtest GTest::gmock GTest::gtest_main)

enable_testing()
include(GoogleTest)
gtest_discover_tests(example_tests)
```

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build -j
ctest --test-dir build --output-on-failure
```

## 테스트 실행

```bash
ctest --test-dir build --output-on-failure
ctest --test-dir build -R ClampTest
ctest --test-dir build -R "UserStoreTest.*" --output-on-failure
```

```bash
./build/example_tests --gtest_filter=ClampTest.*
./build/example_tests --gtest_filter=UserStoreTest.FindsExistingUser
```

## 실패 디버깅

1. gtest 필터로 단일 실패 테스트 재실행
2. 실패 어설션 주변에 스코프 로깅 추가
3. 새니타이저를 활성화하여 재실행
4. 근본 원인이 수정되면 전체 스위트로 확장

## 커버리지

전역 플래그 대신 타겟 수준 설정을 선호합니다.

```cmake
option(ENABLE_COVERAGE "Enable coverage flags" OFF)

if(ENABLE_COVERAGE)
  if(CMAKE_CXX_COMPILER_ID MATCHES "GNU")
    target_compile_options(example_tests PRIVATE --coverage)
    target_link_options(example_tests PRIVATE --coverage)
  elseif(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    target_compile_options(example_tests PRIVATE -fprofile-instr-generate -fcoverage-mapping)
    target_link_options(example_tests PRIVATE -fprofile-instr-generate)
  endif()
endif()
```

GCC + gcov + lcov:

```bash
cmake -S . -B build-cov -DENABLE_COVERAGE=ON
cmake --build build-cov -j
ctest --test-dir build-cov
lcov --capture --directory build-cov --output-file coverage.info
lcov --remove coverage.info '/usr/*' --output-file coverage.info
genhtml coverage.info --output-directory coverage
```

Clang + llvm-cov:

```bash
cmake -S . -B build-llvm -DENABLE_COVERAGE=ON -DCMAKE_CXX_COMPILER=clang++
cmake --build build-llvm -j
LLVM_PROFILE_FILE="build-llvm/default.profraw" ctest --test-dir build-llvm
llvm-profdata merge -sparse build-llvm/default.profraw -o build-llvm/default.profdata
llvm-cov report build-llvm/example_tests -instr-profile=build-llvm/default.profdata
```

## 새니타이저

```cmake
option(ENABLE_ASAN "Enable AddressSanitizer" OFF)
option(ENABLE_UBSAN "Enable UndefinedBehaviorSanitizer" OFF)
option(ENABLE_TSAN "Enable ThreadSanitizer" OFF)

if(ENABLE_ASAN)
  add_compile_options(-fsanitize=address -fno-omit-frame-pointer)
  add_link_options(-fsanitize=address)
endif()
if(ENABLE_UBSAN)
  add_compile_options(-fsanitize=undefined -fno-omit-frame-pointer)
  add_link_options(-fsanitize=undefined)
endif()
if(ENABLE_TSAN)
  add_compile_options(-fsanitize=thread)
  add_link_options(-fsanitize=thread)
endif()
```

## 불안정한 테스트 방지 가이드라인

- 동기화에 `sleep` 사용 금지; 조건 변수 또는 래치 사용
- 임시 디렉토리를 테스트별로 고유하게 만들고 항상 정리
- 단위 테스트에서 실제 시간, 네트워크, 파일시스템 의존성 피하기
- 무작위 입력에 결정론적 시드 사용

## 모범 사례

### 권장 사항

- 테스트를 결정론적이고 격리된 상태로 유지
- 전역 변수보다 의존성 주입 선호
- 전제 조건에는 `ASSERT_*`, 여러 검사에는 `EXPECT_*` 사용
- CTest 레이블 또는 디렉토리로 단위 테스트와 통합 테스트 분리
- 메모리와 레이스 감지를 위해 CI에서 새니타이저 실행

### 금지 사항

- 단위 테스트에서 실제 시간이나 네트워크에 의존하지 않기
- 조건 변수를 사용할 수 있을 때 sleep으로 동기화하지 않기
- 단순한 값 객체에 과도한 mock 사용하지 않기
- 중요하지 않은 로그에 취약한 문자열 매칭 사용하지 않기

### 일반적인 함정

- **고정된 임시 경로 사용** → 테스트별 고유 임시 디렉토리를 생성하고 정리
- **벽시계 시간에 의존** → 클럭을 주입하거나 가짜 시간 소스 사용
- **불안정한 동시성 테스트** → 조건 변수/래치와 제한된 대기 사용
- **숨겨진 전역 상태** → 픽스처에서 전역 상태 초기화 또는 전역 변수 제거
- **과도한 mock** → 상태 있는 동작에는 fake를 선호하고 상호작용만 mock
- **새니타이저 실행 누락** → CI에 ASan/UBSan/TSan 빌드 추가
- **디버그 전용 빌드에서 커버리지** → 커버리지 타겟이 일관된 플래그를 사용하도록 보장

## 선택적 부록: 퍼징 / 속성 테스팅

프로젝트가 이미 LLVM/libFuzzer 또는 속성 테스팅 라이브러리를 지원하는 경우에만 사용합니다.

- **libFuzzer**: 최소한의 I/O를 가진 순수 함수에 최적
- **RapidCheck**: 불변식을 검증하는 속성 기반 테스트

최소 libFuzzer 하니스 (의사코드: ParseConfig를 교체하세요):

```cpp
#include <cstddef>
#include <cstdint>
#include <string>

extern "C" int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size) {
    std::string input(reinterpret_cast<const char *>(data), size);
    // ParseConfig(input); // 프로젝트 함수
    return 0;
}
```

## GoogleTest 대안

- **Catch2**: 헤더 전용, 표현적인 매처
- **doctest**: 경량, 최소한의 컴파일 오버헤드

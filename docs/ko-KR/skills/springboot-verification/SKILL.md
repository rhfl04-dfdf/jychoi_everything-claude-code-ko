---
name: springboot-verification
description: "Spring Boot 프로젝트를 위한 검증 loop: 릴리스 또는 PR 전 빌드, static analysis, coverage를 포함한 테스트, security scan 및 diff review."
origin: ECC
---

# Spring Boot Verification Loop

PR 전, 주요 변경 사항 이후, 그리고 배포 전 실행하세요.

## 활성화 시점

- Spring Boot 서비스에 대한 pull request를 열기 전
- 주요 refactoring 또는 dependency 업그레이드 이후
- 스테이징 또는 프로덕션 배포 전 검증
- 전체 빌드 → lint → 테스트 → security scan 파이프라인 실행 시
- 테스트 coverage가 임계값을 충족하는지 확인 시

## Phase 1: 빌드

```bash
mvn -T 4 clean verify -DskipTests
# 또는
./gradlew clean assemble -x test
```

빌드에 실패하면 중단하고 수정하세요.

## Phase 2: Static Analysis

Maven (공통 플러그인):
```bash
mvn -T 4 spotbugs:check pmd:check checkstyle:check
```

Gradle (설정된 경우):
```bash
./gradlew checkstyleMain pmdMain spotbugsMain
```

## Phase 3: 테스트 + Coverage

```bash
mvn -T 4 test
mvn jacoco:report   # 80% 이상의 coverage 확인
# 또는
./gradlew test jacocoTestReport
```

리포트:
- 전체 테스트 수, 통과/실패
- Coverage % (라인/브랜치)

### Unit Tests

mock된 dependency를 사용하여 서비스 로직을 격리하여 테스트하세요:

```java
 @ExtendWith(MockitoExtension.class)
class UserServiceTest {

  @Mock private UserRepository userRepository;
  @InjectMocks private UserService userService;

  @everything-claude-code/docs/ko-KR/rules/typescript/testing.md
  void createUser_validInput_returnsUser() {
    var dto = new CreateUserDto("Alice", "alice @example.com");
    var expected = new User(1L, "Alice", "alice@example.com");
    when(userRepository.save(any(User.class))).thenReturn(expected);

    var result = userService.create(dto);

    assertThat(result.name()).isEqualTo("Alice");
    verify(userRepository).save(any(User.class));
  }

  @Test
  void createUser_duplicateEmail_throwsException() {
    var dto = new CreateUserDto("Alice", "existing@example.com");
    when(userRepository.existsByEmail(dto.email())).thenReturn(true);

    assertThatThrownBy(() -> userService.create(dto))
        .isInstanceOf(DuplicateEmailException.class);
  }
}
```

### Testcontainers를 사용한 Integration Tests

H2 대신 실제 데이터베이스를 대상으로 테스트하세요:

```java
@SpringBootTest
@Testcontainers
class UserRepositoryIntegrationTest {

  @Container
  static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
      .withDatabaseName("testdb");

  @DynamicPropertySource
  static void configureProperties(DynamicPropertyRegistry registry) {
    registry.add("spring.datasource.url", postgres::getJdbcUrl);
    registry.add("spring.datasource.username", postgres::getUsername);
    registry.add("spring.datasource.password", postgres::getPassword);
  }

  @Autowired private UserRepository userRepository;

  @Test
  void findByEmail_existingUser_returnsUser() {
    userRepository.save(new User("Alice", "alice@example.com"));

    var found = userRepository.findByEmail("alice@example.com");

    assertThat(found).isPresent();
    assertThat(found.get().getName()).isEqualTo("Alice");
  }
}
```

### MockMvc를 사용한 API Tests

전체 Spring context를 사용하여 controller 레이어를 테스트하세요:

```java
@WebMvcTest(UserController.class)
class UserControllerTest {

  @Autowired private MockMvc mockMvc;
  @MockBean private UserService userService;

  @Test
  void createUser_validInput_returns201() throws Exception {
    var user = new UserDto(1L, "Alice", "alice@example.com");
    when(userService.create(any())).thenReturn(user);

    mockMvc.perform(post("/api/users")
            .contentType(MediaType.APPLICATION_JSON)
            .content("""
                {"name": "Alice", "email": "alice @example.com"}
                """))
        .andExpect(status().isCreated())
        .andExpect(jsonPath("$.name").value("Alice"));
  }

  @everything-claude-code/docs/ko-KR/rules/typescript/testing.md
  void createUser_invalidEmail_returns400() throws Exception {
    mockMvc.perform(post("/api/users")
            .contentType(MediaType.APPLICATION_JSON)
            .content("""
                {"name": "Alice", "email": "not-an-email"}
                """))
        .andExpect(status().isBadRequest());
  }
}
```

## Phase 4: Security Scan

```bash
# Dependency CVEs
mvn org.owasp:dependency-check-maven:check
# 또는
./gradlew dependencyCheckAnalyze

# 소스 내 Secrets
grep -rn "password\s*=\s*\"" src/ --include="*.java" --include="*.yml" --include="*.properties"
grep -rn "sk-\|api_key\|secret" src/ --include="*.java" --include="*.yml"

# Secrets (git 히스토리)
git secrets --scan  # 설정된 경우
```

### 일반적인 Security 발견 사항

```
# System.out.println 확인 (대신 logger 사용)
grep -rn "System\.out\.print" src/main/ --include="*.java"

# 응답에 원시 예외 메시지가 포함되어 있는지 확인
grep -rn "e\.getMessage()" src/main/ --include="*.java"

# 와일드카드 CORS 확인
grep -rn "allowedOrigins.*\*" src/main/ --include="*.java"
```

## Phase 5: Lint/Format (선택적 gate)

```bash
mvn spotless:apply   # Spotless 플러그인을 사용하는 경우
./gradlew spotlessApply
```

## Phase 6: Diff Review

```bash
git diff --stat
git diff
```

체크리스트:
- 디버깅 로그가 남아 있지 않음 (`System.out`, guard 없는 `log.debug`)
- 의미 있는 에러 및 HTTP 상태 코드
- 필요한 곳에 transaction 및 validation이 존재함
- 설정 변경 사항이 문서화됨

## 출력 템플릿

```
검증 리포트 (VERIFICATION REPORT)
===================
빌드:      [PASS/FAIL]
Static:    [PASS/FAIL] (spotbugs/pmd/checkstyle)
테스트:    [PASS/FAIL] (X/Y 통과, Z% coverage)
Security:  [PASS/FAIL] (CVE 발견: N)
Diff:      [X개 파일 변경됨]

종합:      [READY / NOT READY]

수정할 사항:
1. ...
2. ...
```

## 지속 모드 (Continuous Mode)

- 중요한 변경 사항이 있거나 긴 세션 중에는 30~60분마다 단계를 재실행하세요.
- 빠른 피드백을 위해 짧은 loop를 유지하세요: `mvn -T 4 test` + spotbugs

**주의**: 빠른 피드백은 나중에 발생하는 문제를 방지합니다. Gate를 엄격하게 유지하세요. 프로덕션 시스템에서는 경고(warning)도 결함(defect)으로 취급하세요.

--- Content from referenced files ---
Content from @everything-claude-code/docs/ko-KR/rules/typescript/testing.md:
---
paths:
  - "**/*.ts"
  - "**/*.tsx"
  - "**/*.js"
  - "**/*.jsx"
---
# TypeScript/JavaScript 테스트

> 이 파일은 [common/testing.md](../common/testing.md)을 TypeScript/JavaScript 전용 내용으로 확장합니다.

## E2E 테스트

주요 user flows를 위한 E2E 테스트 framework로 **Playwright**를 사용하세요.

## Agent 지원

- **e2e-runner** - Playwright E2E 테스트 specialist
--- End of content ---

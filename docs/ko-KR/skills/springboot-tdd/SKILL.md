---
name: springboot-tdd
description: JUnit 5, Mockito, MockMvc, Testcontainers 및 JaCoCo를 사용한 Spring Boot용 Test-driven development입니다. 기능 추가, 버그 수정 또는 refactoring 시 사용하세요.
origin: ECC
---

# Spring Boot TDD Workflow

80% 이상의 coverage(unit + integration)를 갖는 Spring Boot 서비스를 위한 TDD 가이드입니다.

## 사용 시점

- 새로운 기능 또는 엔드포인트 추가
- 버그 수정 또는 refactor
- 데이터 액세스 로직 또는 보안 규칙 추가

## Workflow

1) 테스트를 먼저 작성합니다 (실패해야 함)
2) 통과를 위한 최소한의 코드를 구현합니다
3) 테스트가 통과(green)된 상태에서 refactor를 수행합니다
4) coverage를 강제합니다 (JaCoCo)

## Unit Tests (JUnit 5 + Mockito)

```java
 @ExtendWith(MockitoExtension.class)
class MarketServiceTest {
  @Mock MarketRepository repo;
  @InjectMocks MarketService service;

  @everything-claude-code/docs/ko-KR/rules/typescript/testing.md
  void createsMarket() {
    CreateMarketRequest req = new CreateMarketRequest("name", "desc", Instant.now(), List.of("cat"));
    when(repo.save(any())).thenAnswer(inv -> inv.getArgument(0));

    Market result = service.create(req);

    assertThat(result.name()).isEqualTo("name");
    verify(repo).save(any());
  }
}
```

패턴:
- Arrange-Act-Assert
- 부분 mock을 피하고 명시적인 stubbing을 선호하세요
- 다양한 케이스를 위해 `@ParameterizedTest`를 사용하세요

## Web Layer Tests (MockMvc)

```java
 @WebMvcTest(MarketController.class)
class MarketControllerTest {
  @Autowired MockMvc mockMvc;
  @MockBean MarketService marketService;

  @everything-claude-code/docs/ko-KR/rules/typescript/testing.md
  void returnsMarkets() throws Exception {
    when(marketService.list(any())).thenReturn(Page.empty());

    mockMvc.perform(get("/api/markets"))
        .andExpect(status().isOk())
        .andExpect(jsonPath("$.content").isArray());
  }
}
```

## Integration Tests (SpringBootTest)

```java
 @SpringBootTest @AutoConfigureMockMvc @ActiveProfiles("test")
class MarketIntegrationTest {
  @Autowired MockMvc mockMvc;

  @everything-claude-code/docs/ko-KR/rules/typescript/testing.md
  void createsMarket() throws Exception {
    mockMvc.perform(post("/api/markets")
        .contentType(MediaType.APPLICATION_JSON)
        .content("""
          {"name":"Test","description":"Desc","endDate":"2030-01-01T00:00:00Z","categories":["general"]}
        """))
      .andExpect(status().isCreated());
  }
}
```

## Persistence Tests (DataJpaTest)

```java
 @DataJpaTest @AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
 @everything-claude-code/docs/ko-KR/commands/instinct-import.md(TestContainersConfig.class)
class MarketRepositoryTest {
  @Autowired MarketRepository repo;

  @everything-claude-code/docs/ko-KR/rules/typescript/testing.md
  void savesAndFinds() {
    MarketEntity entity = new MarketEntity();
    entity.setName("Test");
    repo.save(entity);

    Optional<MarketEntity> found = repo.findByName("Test");
    assertThat(found).isPresent();
  }
}
```

## Testcontainers

- 프로덕션 환경을 반영하기 위해 Postgres/Redis용 재사용 가능한 컨테이너를 사용하세요
- JDBC URL을 Spring context에 주입하기 위해 `@DynamicPropertySource`를 통해 연결하세요

## Coverage (JaCoCo)

Maven 스니펫:
```xml
<plugin>
  <groupId>org.jacoco</groupId>
  <artifactId>jacoco-maven-plugin</artifactId>
  <version>0.8.14</version>
  <executions>
    <execution>
      <goals><goal>prepare-agent</goal></goals>
    </execution>
    <execution>
      <id>report</id>
      <phase>verify</phase>
      <goals><goal>report</goal></goals>
    </execution>
  </executions>
</plugin>
```

## Assertions

- 가독성을 위해 AssertJ(`assertThat`)를 선호하세요
- JSON 응답에는 `jsonPath`를 사용하세요
- 예외 발생 시: `assertThatThrownBy(...)`

## Test Data Builders

```java
class MarketBuilder {
  private String name = "Test";
  MarketBuilder withName(String name) { this.name = name; return this; }
  Market build() { return new Market(null, name, MarketStatus.ACTIVE); }
}
```

## CI 커맨드

- Maven: `mvn -T 4 test` 또는 `mvn verify`
- Gradle: `./gradlew test jacocoTestReport`

**주의**: 테스트를 빠르고, 격리되며, 결정론적으로 유지하세요. 구현 세부 사항이 아닌 동작을 테스트하세요.
--- 참조 파일 내용 ---
@everything-claude-code/docs/ko-KR/commands/instinct-import.md 내용:
---
name: instinct-import
description: 파일 또는 URL에서 프로젝트/글로벌 범위로 instinct를 가져옵니다
command: true
---

# Instinct Import 커맨드

## 구현

플러그인 루트 경로를 사용하여 instinct CLI를 실행합니다:

```bash
python3 "${CLAUDE_PLUGIN_ROOT}/skills/continuous-learning-v2/scripts/instinct-cli.py" import <file-or-url> [--dry-run] [--force] [--min-confidence 0.7] [--scope project|global]
```

또는 `CLAUDE_PLUGIN_ROOT`가 설정되지 않은 경우 (수동 설치):

```bash
python3 ~/.claude/skills/continuous-learning-v2/scripts/instinct-cli.py import <file-or-url>
```

로컬 파일 경로 또는 HTTP(S) URL에서 instinct를 가져옵니다.

## 사용법

```
/instinct-import team-instincts.yaml
/instinct-import https://github.com/org/repo/instincts.yaml
/instinct-import team-instincts.yaml --dry-run
/instinct-import team-instincts.yaml --scope global --force
```

## 수행할 작업

1. instinct 파일 가져오기 (로컬 경로 또는 URL)
2. 형식 파싱 및 유효성 검사
3. 기존 instinct와 중복 확인
4. 새 instinct 병합 또는 추가
5. inherited instinct 디렉토리에 저장:
   - 프로젝트 범위: `~/.claude/homunculus/projects/<project-id>/instincts/inherited/`
   - 글로벌 범위: `~/.claude/homunculus/instincts/inherited/`

## 가져오기 프로세스

```
📥 Importing instincts from: team-instincts.yaml
================================================

Found 12 instincts to import.

Analyzing conflicts...

## New Instincts (8)
These will be added:
  ✓ use-zod-validation (confidence: 0.7)
  ✓ prefer-named-exports (confidence: 0.65)
  ✓ test-async-functions (confidence: 0.8)
  ...

## Duplicate Instincts (3)
Already have similar instincts:
  ⚠️ prefer-functional-style
     Local: 0.8 confidence, 12 observations
     Import: 0.7 confidence
     → Keep local (higher confidence)

  ⚠️ test-first-workflow
     Local: 0.75 confidence
     Import: 0.9 confidence
     → Update to import (higher confidence)

Import 8 new, update 1?
```

## 병합 동작

기존 ID를 가진 instinct를 가져올 때:
- 더 높은 신뢰도의 import는 업데이트 후보가 됩니다
- 같거나 낮은 신뢰도의 import는 건너뜁니다
- `--force`를 사용하지 않는 한 사용자가 확인합니다

## 소스 추적

가져온 instinct는 다음과 같이 표시됩니다:
```yaml
source: inherited
scope: project
imported_from: "team-instincts.yaml"
project_id: "a1b2c3d4e5f6"
project_name: "my-project"
```

## 플래그

- `--dry-run`: 가져오기 없이 미리 보기
- `--force`: 확인 프롬프트 건너뜀
- `--min-confidence <n>`: 임계값 이상의 instinct만 가져오기
- `--scope <project|global>`: 대상 범위 선택 (기본값: `project`)

## 출력

가져오기 후:
```
✅ Import complete!

Added: 8 instincts
Updated: 1 instinct
Skipped: 3 instincts (equal/higher confidence already exists)

New instincts saved to: ~/.claude/homunculus/instincts/inherited/

Run /instinct-status to see all instincts.
```
@everything-claude-code/docs/ko-KR/rules/typescript/testing.md 내용:
---
paths:
  - "**/*.ts"
  - "**/*.tsx"
  - "**/*.js"
  - "**/*.jsx"
---
# TypeScript/JavaScript Testing

> 이 파일은 [common/testing.md](../common/testing.md)을 TypeScript/JavaScript 전용 내용으로 확장합니다.

## E2E Testing

주요 user flows를 위한 E2E testing framework로 **Playwright**를 사용하세요.

## Agent Support

- **e2e-runner** - Playwright E2E testing specialist
--- 내용 끝 ---

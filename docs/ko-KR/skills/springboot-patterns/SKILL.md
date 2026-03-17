---
name: springboot-patterns
description: Spring Boot 아키텍처 패턴, REST API 디자인, 계층화된 서비스, 데이터 액세스, 캐싱, 비동기 처리 및 로깅. Java Spring Boot backend 작업 시 사용하세요.
origin: ECC
---

# Spring Boot 개발 패턴

확장 가능하고 상용 수준의 서비스를 위한 Spring Boot 아키텍처 및 API 패턴.

## 활성화 시점

- Spring MVC 또는 WebFlux로 REST API를 구축할 때
- controller → service → repository 계층을 구조화할 때
- Spring Data JPA, 캐싱 또는 비동기 처리를 설정할 때
- 검증, 예외 처리 또는 페이지네이션을 추가할 때
- dev/staging/production 환경을 위한 프로필을 설정할 때
- Spring Events 또는 Kafka를 사용하여 이벤트 기반 패턴을 구현할 때

## REST API 구조

```java
 @RestController @RequestMapping("/api/markets")
 @Validated
class MarketController {
  private final MarketService marketService;

  MarketController(MarketService marketService) {
    this.marketService = marketService;
  }

  @GetMapping
  ResponseEntity<Page<MarketResponse>> list(
      @RequestParam(defaultValue = "0") int page,
      @RequestParam(defaultValue = "20") int size) {
    Page<Market> markets = marketService.list(PageRequest.of(page, size));
    return ResponseEntity.ok(markets.map(MarketResponse::from));
  }

  @PostMapping
  ResponseEntity<MarketResponse> create( @everything-claude-code/tests/ci/validators.test.js @RequestBody CreateMarketRequest request) {
    Market market = marketService.create(request);
    return ResponseEntity.status(HttpStatus.CREATED).body(MarketResponse.from(market));
  }
}
```

## Repository 패턴 (Spring Data JPA)

```java
public interface MarketRepository extends JpaRepository<MarketEntity, Long> {
  @Query("select m from MarketEntity m where m.status = :status order by m.volume desc")
  List<MarketEntity> findActive( @Param("status") MarketStatus status, Pageable pageable);
}
```

## 트랜잭션을 포함한 Service 계층

```java
 @everything-claude-code/examples/go-microservice-CLAUDE.md
public class MarketService {
  private final MarketRepository repo;

  public MarketService(MarketRepository repo) {
    this.repo = repo;
  }

  @Transactional
  public Market create(CreateMarketRequest request) {
    MarketEntity entity = MarketEntity.from(request);
    MarketEntity saved = repo.save(entity);
    return Market.from(saved);
  }
}
```

## DTO 및 검증

```java
public record CreateMarketRequest(
    @NotBlank @Size(max = 200) String name,
    @NotBlank @Size(max = 2000) String description,
    @NotNull @FutureOrPresent Instant endDate,
    @NotEmpty List< @NotBlank String> categories) {}

public record MarketResponse(Long id, String name, MarketStatus status) {
  static MarketResponse from(Market market) {
    return new MarketResponse(market.id(), market.name(), market.status());
  }
}
```

## 예외 처리

```java
 @ControllerAdvice
class GlobalExceptionHandler {
  @ExceptionHandler(MethodArgumentNotValidException.class)
  ResponseEntity<ApiError> handleValidation(MethodArgumentNotValidException ex) {
    String message = ex.getBindingResult().getFieldErrors().stream()
        .map(e -> e.getField() + ": " + e.getDefaultMessage())
        .collect(Collectors.joining(", "));
    return ResponseEntity.badRequest().body(ApiError.validation(message));
  }

  @ExceptionHandler(AccessDeniedException.class)
  ResponseEntity<ApiError> handleAccessDenied() {
    return ResponseEntity.status(HttpStatus.FORBIDDEN).body(ApiError.of("Forbidden"));
  }

  @ExceptionHandler(Exception.class)
  ResponseEntity<ApiError> handleGeneric(Exception ex) {
    // Log unexpected errors with stack traces
    return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
        .body(ApiError.of("Internal server error"));
  }
}
```

## 캐싱

설정 클래스에 `@EnableCaching`이 필요합니다.

```java
 @everything-claude-code/examples/go-microservice-CLAUDE.md
public class MarketCacheService {
  private final MarketRepository repo;

  public MarketCacheService(MarketRepository repo) {
    this.repo = repo;
  }

  @Cacheable(value = "market", key = "#id")
  public Market getById(Long id) {
    return repo.findById(id)
        .map(Market::from)
        .orElseThrow(() -> new EntityNotFoundException("Market not found"));
  }

  @CacheEvict(value = "market", key = "#id")
  public void evict(Long id) {}
}
```

## 비동기 처리

설정 클래스에 `@EnableAsync`가 필요합니다.

```java
 @everything-claude-code/examples/go-microservice-CLAUDE.md
public class NotificationService {
  @Async
  public CompletableFuture<Void> sendAsync(Notification notification) {
    // send email/SMS
    return CompletableFuture.completedFuture(null);
  }
}
```

## 로깅 (SLF4J)

```java
 @everything-claude-code/examples/go-microservice-CLAUDE.md
public class ReportService {
  private static final Logger log = LoggerFactory.getLogger(ReportService.class);

  public Report generate(Long marketId) {
    log.info("generate_report marketId={}", marketId);
    try {
      // logic
    } catch (Exception ex) {
      log.error("generate_report_failed marketId={}", marketId, ex);
      throw ex;
    }
    return new Report();
  }
}
```

## 미들웨어 / 필터

```java
 @everything-claude-code/schemas/install-components.schema.json
public class RequestLoggingFilter extends OncePerRequestFilter {
  private static final Logger log = LoggerFactory.getLogger(RequestLoggingFilter.class);

  @Override
  protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
      FilterChain filterChain) throws ServletException, IOException {
    long start = System.currentTimeMillis();
    try {
      filterChain.doFilter(request, response);
    } finally {
      long duration = System.currentTimeMillis() - start;
      log.info("req method={} uri={} status={} durationMs={}",
          request.getMethod(), request.getRequestURI(), response.getStatus(), duration);
    }
  }
}
```

## 페이지네이션 및 정렬

```java
PageRequest page = PageRequest.of(pageNumber, pageSize, Sort.by("createdAt").descending());
Page<Market> results = marketService.list(page);
```

## 오류 회복력이 있는 외부 호출

```java
public <T> T withRetry(Supplier<T> supplier, int maxRetries) {
  int attempts = 0;
  while (true) {
    try {
      return supplier.get();
    } catch (Exception ex) {
      attempts++;
      if (attempts >= maxRetries) {
        throw ex;
      }
      try {
        Thread.sleep((long) Math.pow(2, attempts) * 100L);
      } catch (InterruptedException ie) {
        Thread.currentThread().interrupt();
        throw ex;
      }
    }
  }
}
```

## Rate Limiting (필터 + Bucket4j)

**보안 참고**: `X-Forwarded-For` 헤더는 클라이언트가 속일 수 있으므로 기본적으로 신뢰할 수 없습니다.
다음과 같은 경우에만 전달된 헤더를 사용하세요:
1. 애플리케이션이 신뢰할 수 있는 리버스 프록시(nginx, AWS ALB 등) 뒤에 있는 경우
2. `ForwardedHeaderFilter`를 빈으로 등록한 경우
3. 애플리케이션 속성에서 `server.forward-headers-strategy=NATIVE` 또는 `FRAMEWORK`로 설정한 경우
4. 프록시가 `X-Forwarded-For` 헤더를 (추가하는 것이 아니라) 덮어쓰도록 설정된 경우

`ForwardedHeaderFilter`가 올바르게 설정되면 `request.getRemoteAddr()`는 전달된 헤더에서 올바른 클라이언트 IP를 자동으로 반환합니다. 이 설정이 없으면 `request.getRemoteAddr()`를 직접 사용하세요. 이는 즉각적인 연결 IP를 반환하며, 이것이 유일하게 신뢰할 수 있는 값입니다.

```java
 @everything-claude-code/schemas/install-components.schema.json
public class RateLimitFilter extends OncePerRequestFilter {
  private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();

  /*
   * SECURITY: This filter uses request.getRemoteAddr() to identify clients for rate limiting.
   *
   * If your application is behind a reverse proxy (nginx, AWS ALB, etc.), you MUST configure
   * Spring to handle forwarded headers properly for accurate client IP detection:
   *
   * 1. Set server.forward-headers-strategy=NATIVE (for cloud platforms) or FRAMEWORK in
   *    application.properties/yaml
   * 2. If using FRAMEWORK strategy, register ForwardedHeaderFilter:
   *
   *    @Bean
   *    ForwardedHeaderFilter forwardedHeaderFilter() {
   *        return new ForwardedHeaderFilter();
   *    }
   *
   * 3. Ensure your proxy overwrites (not appends) the X-Forwarded-For header to prevent spoofing
   * 4. Configure server.tomcat.remoteip.trusted-proxies or equivalent for your container
   *
   * Without this configuration, request.getRemoteAddr() returns the proxy IP, not the client IP.
   * Do NOT read X-Forwarded-For directly—it is trivially spoofable without trusted proxy handling.
   */
  @Override
  protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
      FilterChain filterChain) throws ServletException, IOException {
    // Use getRemoteAddr() which returns the correct client IP when ForwardedHeaderFilter
    // is configured, or the direct connection IP otherwise. Never trust X-Forwarded-For
    // headers directly without proper proxy configuration.
    String clientIp = request.getRemoteAddr();

    Bucket bucket = buckets.computeIfAbsent(clientIp,
        k -> Bucket.builder()
            .addLimit(Bandwidth.classic(100, Refill.greedy(100, Duration.ofMinutes(1))))
            .build());

    if (bucket.tryConsume(1)) {
      filterChain.doFilter(request, response);
    } else {
      response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
    }
  }
}
```

## 백그라운드 작업

Spring의 `@Scheduled`를 사용하거나 큐(예: Kafka, SQS, RabbitMQ)와 통합하세요. 핸들러를 멱등성(idempotent) 있게 유지하고 모니터링 가능하도록 하세요.

## 관측 가능성 (Observability)

- Logback 인코더를 통한 구조화된 로깅 (JSON)
- 메트릭: Micrometer + Prometheus/OTel
- 트레이싱: OpenTelemetry 또는 Brave 백엔드를 사용한 Micrometer Tracing

## 운영 환경 기본값

- 생성자 주입을 선호하고 필드 주입을 피하세요.
- RFC 7807 오류를 위해 `spring.mvc.problemdetails.enabled=true`를 활성화하세요 (Spring Boot 3+).
- 워크로드에 맞게 HikariCP 풀 크기를 구성하고 타임아웃을 설정하세요.
- 조회 쿼리에는 `@Transactional(readOnly = true)`를 사용하세요.
- `@NonNull` 및 적절한 경우 `Optional`을 통해 null-safety를 강화하세요.

**기억하세요**: controller는 얇게, service는 집중되게, repository는 단순하게 유지하고, 예외 처리는 중앙에서 관리하세요. 유지보수성과 테스트 가능성을 위해 최적화하세요.

--- Content from referenced files ---
Content from @everything-claude-code/examples/go-microservice-CLAUDE.md:
# Go Microservice — Project CLAUDE.md

> PostgreSQL, gRPC 및 Docker를 사용하는 Go microservice의 실제 예시입니다.
> 이 내용을 프로젝트 루트에 복사하고 서비스에 맞게 수정하세요.

## 프로젝트 개요

**Stack:** Go 1.22+, PostgreSQL, gRPC + REST (grpc-gateway), Docker, sqlc (type-safe SQL), Wire (dependency injection)

**아키텍처:** domain, repository, service, handler 계층을 포함한 Clean architecture입니다. 기본 전송 방식으로 gRPC를 사용하며 외부 클라이언트를 위해 REST gateway를 제공합니다.

## 핵심 규칙

### Go 컨벤션

- Effective Go 및 Go Code Review Comments 가이드를 따르세요.
- 오류 래핑 시 `errors.New` / `fmt.Errorf`와 `%w`를 사용하세요. 오류 메시지에 문자열 매칭을 사용하지 마세요.
- `init()` 함수를 사용하지 마세요. `main()` 또는 생성자에서 명시적으로 초기화하세요.
- 전역 가변 상태를 두지 마세요. 생성자를 통해 의존성을 전달하세요.
- Context는 반드시 첫 번째 매개변수여야 하며 모든 계층에 전달되어야 합니다.

### 데이터베이스

- 모든 쿼리는 `queries/`에 일반 SQL로 작성하세요. sqlc가 type-safe한 Go 코드를 생성합니다.
- 마이그레이션은 golang-migrate를 사용하여 `migrations/`에 작성하세요. 데이터베이스를 직접 수정하지 마세요.
- 다단계 작업에는 `pgx.Tx`를 통한 트랜잭션을 사용하세요.
- 모든 쿼리는 파라미터화된 플레이스홀더(`$1`, `$2`)를 사용해야 합니다. 문자열 포매팅을 사용하지 마세요.

### 예외 처리

- panic하지 말고 오류를 반환하세요. panic은 정말로 복구 불가능한 상황에만 사용합니다.
- `fmt.Errorf("creating user: %w", err)`와 같이 컨텍스트와 함께 오류를 래핑하세요.
- 비즈니스 로직을 위해 `domain/errors.go`에 sentinel 오류를 정의하세요.
- handler 계층에서 domain 오류를 gRPC 상태 코드로 매핑하세요.

```go
// Domain layer — sentinel errors
var (
    ErrUserNotFound  = errors.New("user not found")
    ErrEmailTaken    = errors.New("email already registered")
)

// Handler layer — map to gRPC status
func toGRPCError(err error) error {
    switch {
    case errors.Is(err, domain.ErrUserNotFound):
        return status.Error(codes.NotFound, err.Error())
    case errors.Is(err, domain.ErrEmailTaken):
        return status.Error(codes.AlreadyExists, err.Error())
    default:
        return status.Error(codes.Internal, "internal error")
    }
}
```

### 코드 스타일

- 코드나 주석에 이모지를 사용하지 마세요.
- 노출된(Exported) 타입과 함수에는 반드시 doc 주석을 작성하세요.
- 함수는 50라인 이하로 유지하세요. 필요시 헬퍼 함수를 추출하세요.
- 모든 로직에 대해 여러 케이스를 포함하는 table-driven 테스트를 사용하세요.
- signal 채널에는 `bool`이 아닌 `struct{}`를 선호하세요.

## 파일 구조

```
cmd/
  server/
    main.go              # Entrypoint, Wire injection, graceful shutdown
internal/
  domain/                # Business types and interfaces
    user.go              # User entity and repository interface
    errors.go            # Sentinel errors
  service/               # Business logic
    user_service.go
    user_service_test.go
  repository/            # Data access (sqlc-generated + custom)
    postgres/
      user_repo.go
      user_repo_test.go  # Integration tests with testcontainers
  handler/               # gRPC + REST handlers
    grpc/
      user_handler.go
    rest/
      user_handler.go
  config/                # Configuration loading
    config.go
proto/                   # Protobuf definitions
  user/v1/
    user.proto
queries/                 # SQL queries for sqlc
  user.sql
migrations/              # Database migrations
  001_create_users.up.sql
  001_create_users.down.sql
```

## 주요 패턴

### Repository 인터페이스

```go
type UserRepository interface {
    Create(ctx context.Context, user *User) error
    FindByID(ctx context.Context, id uuid.UUID) (*User, error)
    FindByEmail(ctx context.Context, email string) (*User, error)
    Update(ctx context.Context, user *User) error
    Delete(ctx context.Context, id uuid.UUID) error
}
```

### Dependency Injection을 사용한 Service

```go
type UserService struct {
    repo   domain.UserRepository
    hasher PasswordHasher
    logger *slog.Logger
}

func NewUserService(repo domain.UserRepository, hasher PasswordHasher, logger *slog.Logger) *UserService {
    return &UserService{repo: repo, hasher: hasher, logger: logger}
}

func (s *UserService) Create(ctx context.Context, req CreateUserRequest) (*domain.User, error) {
    existing, err := s.repo.FindByEmail(ctx, req.Email)
    if err != nil && !errors.Is(err, domain.ErrUserNotFound) {
        return nil, fmt.Errorf("checking email: %w", err)
    }
    if existing != nil {
        return nil, domain.ErrEmailTaken
    }

    hashed, err := s.hasher.Hash(req.Password)
    if err != nil {
        return nil, fmt.Errorf("hashing password: %w", err)
    }

    user := &domain.User{
        ID:       uuid.New(),
        Name:     req.Name,
        Email:    req.Email,
        Password: hashed,
    }
    if err := s.repo.Create(ctx, user); err != nil {
        return nil, fmt.Errorf("creating user: %w", err)
    }
    return user, nil
}
```

### Table-Driven 테스트

```go
func TestUserService_Create(t *testing.T) {
    tests := []struct {
        name    string
        req     CreateUserRequest
        setup   func(*MockUserRepo)
        wantErr error
    }{
        {
            name: "valid user",
            req:  CreateUserRequest{Name: "Alice", Email: "alice@example.com", Password: "secure123"},
            setup: func(m *MockUserRepo) {
                m.On("FindByEmail", mock.Anything, "alice@example.com").Return(nil, domain.ErrUserNotFound)
                m.On("Create", mock.Anything, mock.Anything).Return(nil)
            },
            wantErr: nil,
        },
        {
            name: "duplicate email",
            req:  CreateUserRequest{Name: "Alice", Email: "taken@example.com", Password: "secure123"},
            setup: func(m *MockUserRepo) {
                m.On("FindByEmail", mock.Anything, "taken@example.com").Return(&domain.User{}, nil)
            },
            wantErr: domain.ErrEmailTaken,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            repo := new(MockUserRepo)
            tt.setup(repo)
            svc := NewUserService(repo, &bcryptHasher{}, slog.Default())

            _, err := svc.Create(context.Background(), tt.req)

            if tt.wantErr != nil {
                assert.ErrorIs(t, err, tt.wantErr)
            } else {
                assert.NoError(t, err)
            }
        })
    }
}
```

## 환경 변수

```bash
# Database
DATABASE_URL=postgres://user:pass@localhost:5432/myservice?sslmode=disable

# gRPC
GRPC_PORT=50051
REST_PORT=8080

# Auth
JWT_SECRET=           # Load from vault in production
TOKEN_EXPIRY=24h

# Observability
LOG_LEVEL=info        # debug, info, warn, error
OTEL_ENDPOINT=        # OpenTelemetry collector
```

## 테스트 전략

```bash
/go-test             # Go를 위한 TDD workflow
/go-review           # Go 전용 code review
/go-build            # 빌드 오류 수정
```

### 테스트 명령

```bash
# Unit tests (fast, no external deps)
go test ./internal/... -short -count=1

# Integration tests (requires Docker for testcontainers)
go test ./internal/repository/... -count=1 -timeout 120s

# All tests with coverage
go test ./... -coverprofile=coverage.out -count=1
go tool cover -func=coverage.out  # summary
go tool cover -html=coverage.out  # browser

# Race detector
go test ./... -race -count=1
```

## ECC Workflow

```bash
# Planning
/plan "Add rate limiting to user endpoints"

# Development
/go-test                  # Go 전용 패턴을 사용한 TDD

# Review
/go-review                # Go idioms, 예외 처리, 동시성
/security-scan            # Secrets 및 취약점 점검

# Before merge
go vet ./...
staticcheck ./...
```

## Git Workflow

- `feat:` 새로운 기능, `fix:` 버그 수정, `refactor:` 코드 변경
- `main`에서 기능 브랜치 생성, PR 필수
- CI: `go vet`, `staticcheck`, `go test -race`, `golangci-lint`
- 배포: CI에서 빌드된 Docker 이미지를 Kubernetes에 배포
Content from @everything-claude-code/schemas/install-components.schema.json:
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "ECC Install Components",
  "type": "object",
  "additionalProperties": false,
  "required": [
    "version",
    "components"
  ],
  "properties": {
    "version": {
      "type": "integer",
      "minimum": 1
    },
    "components": {
      "type": "array",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": [
          "id",
          "family",
          "description",
          "modules"
        ],
        "properties": {
          "id": {
            "type": "string",
            "pattern": "^(baseline|lang|framework|capability):[a-z0-9-]+$"
          },
          "family": {
            "type": "string",
            "enum": [
              "baseline",
              "language",
              "framework",
              "capability"
            ]
          },
          "description": {
            "type": "string",
            "minLength": 1
          },
          "modules": {
            "type": "array",
            "minItems": 1,
            "items": {
              "type": "string",
              "pattern": "^[a-z0-9-]+$"
            }
          }
        }
      }
    }
  }
}
Content from @everything-claude-code/tests/ci/validators.test.js:
[WARNING: This file was truncated. To view the full content, use the 'read_file' tool on this specific file.]

/**
 * CI validator 스크립트를 위한 테스트
 *
 * 성공 경로(실제 프로젝트 대상)와 오류 경로(래퍼 스크립트를 통한 
 * 임시 fixture 디렉토리 대상)를 모두 테스트합니다.
 *
 * 실행: node tests/ci/validators.test.js
 */

const assert = require('assert');
const path = require('path');
const fs = require('fs');
const os = require('os');
const { execFileSync } = require('child_process');

const validatorsDir = path.join(__dirname, '..', '..', 'scripts', 'ci');
const repoRoot = path.join(__dirname, '..', '..');
const modulesSchemaPath = path.join(repoRoot, 'schemas', 'install-modules.schema.json');
const profilesSchemaPath = path.join(repoRoot, 'schemas', 'install-profiles.schema.json');
const componentsSchemaPath = path.join(repoRoot, 'schemas', 'install-components.schema.json');

// Test helpers
function test(name, fn) {
  try {
    fn();
    console.log(`  \u2713 ${name}`);
    return true;
  } catch (err) {
    console.log(`  \u2717 ${name}`);
    console.log(`    Error: ${err.message}`);
    return false;
  }
}

function createTestDir() {
  return fs.mkdtempSync(path.join(os.tmpdir(), 'ci-validator-test-'));
}

function cleanupTestDir(testDir) {
  fs.rmSync(testDir, { recursive: true, force: true });
}

function writeJson(filePath, value) {
  fs.mkdirSync(path.dirname(filePath), { recursive: true });
  fs.writeFileSync(filePath, JSON.stringify(value, null, 2));
}

function writeInstallComponentsManifest(testDir, components) {
  writeJson(path.join(testDir, 'manifests', 'install-components.json'), {
    version: 1,
    components,
  });
}

/**
 * 디렉토리 상수를 재정의하는 래퍼를 통해 validator 스크립트를 실행합니다.
 * 이를 통해 실제 프로젝트 파일을 수정하지 않고도 오류 사례를 테스트할 수 있습니다.
 *
 * @param {string} validatorName - 예: 'validate-agents'
 * @param {string} dirConstant - 재정의할 상수 이름 (예: 'AGENTS_DIR')
 * @param {string} overridePath - 사용할 임시 디렉토리
 * @returns {{code: number, stdout: string, stderr: string}}
 */
function runValidatorWithDir(validatorName, dirConstant, overridePath) {
  const validatorPath = path.join(validatorsDir, `${validatorName}.js`);

  // validator 소스를 읽고 디렉토리 상수를 교체한 후 래퍼로 실행합니다.
  let source = fs.readFileSync(validatorPath, 'utf8');

  // shebang 라인 제거
  source = source.replace(/^#!.*\n/, '');

  // 디렉토리 상수를 재정의 경로로 교체
  const dirRegex = new RegExp(`const ${dirConstant} = .*?;`);
  source = source.replace(dirRegex, `const ${dirConstant} = ${JSON.stringify(overridePath)};`);

  try {
    const stdout = execFileSync('node', ['-e', source], {
      encoding: 'utf8',
      stdio: ['pipe', 'pipe', 'pipe'],
      timeout: 10000,
    });
    return { code: 0, stdout, stderr: '' };
  } catch (err) {
    return {
      code: err.status || 1,
      stdout: err.stdout || '',
      stderr: err.stderr || '',
    };
  }
}

/**
 * 여러 디렉토리 재정의와 함께 validator 스크립트를 실행합니다.
 * @param {string} validatorName
 * @param {Record<string, string>} overrides - 상수 이름과 경로의 맵
 */
function runValidatorWithDirs(validatorName, overrides) {
  const validatorPath = path.join(validatorsDir, `${validatorName}.js`);
  let source = fs.readFileSync(validatorPath, 'utf8');
  source = source.replace(/^#!.*\n/, '');
  for (const [constant, overridePath] of Object.entries(overrides)) {
    const dirRegex = new RegExp(`const ${constant} = .*?;`);
    source = source.replace(dirRegex, `const ${constant} = ${JSON.stringify(overridePath)};`);
  }
  try {
    const stdout = execFileSync('node', ['-e', source], {
      encoding: 'utf8',
      stdio: ['pipe', 'pipe', 'pipe'],
      timeout: 10000,
    });
    return { code: 0, stdout, stderr: '' };
  } catch (err) {
    return {
      code: err.status || 1,
      stdout: err.stdout || '',
      stderr: err.stderr || '',
    };
  }
}

/**
 * validator 스크립트를 직접 실행합니다 (실제 프로젝트 테스트용).
 */
function runValidator(validatorName) {
  const validatorPath = path.join(validatorsDir, `${validatorName}.js`);
  try {
    const stdout = execFileSync('node', [validatorPath], {
      encoding: 'utf8',
      stdio: ['pipe', 'pipe', 'pipe'],
      timeout: 15000,
    });
    return { code: 0, stdout, stderr: '' };
  } catch (err) {
    return {
      code: err.status || 1,
      stdout: err.stdout || '',
      stderr: err.stderr || '',
    };
  }
}

function runTests() {
  console.log('\n=== Testing CI Validators ===\n');

  let passed = 0;
  let failed = 0;

  // ==========================================
  // validate-agents.js
  // ==========================================
  console.log('validate-agents.js:');

  if (test('passes on real project agents', () => {
    const result = runValidator('validate-agents');
    assert.strictEqual(result.code, 0, `Should pass, got stderr: ${result.stderr}`);
    assert.ok(result.stdout.includes('Validated'), 'Should output validation count');
  })) passed++; else failed++;

  if (test('fails on agent without frontmatter', () => {
    const testDir = createTestDir();
    fs.writeFileSync(path.join(testDir, 'bad-agent.md'), '# No frontmatter here\nJust content.');

    const result = runValidatorWithDir('validate-agents', 'AGENTS_DIR', testDir);
    assert.strictEqual(result.code, 1, 'Should exit 1 for missing frontmatter');
    assert.ok(result.stderr.includes('Missing frontmatter'), 'Should report missing frontmatter');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('fails on agent missing required model field', () => {
    const testDir = createTestDir();
    fs.writeFileSync(path.join(testDir, 'no-model.md'), '---\ntools: Read, Write\n---\n# Agent');

    const result = runValidatorWithDir('validate-agents', 'AGENTS_DIR', testDir);
    assert.strictEqual(result.code, 1, 'Should exit 1 for missing model');
    assert.ok(result.stderr.includes('model'), 'Should report missing model field');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('fails on agent missing required tools field', () => {
    const testDir = createTestDir();
    fs.writeFileSync(path.join(testDir, 'no-tools.md'), '---\nmodel: sonnet\n---\n# Agent');

    const result = runValidatorWithDir('validate-agents', 'AGENTS_DIR', testDir);
    assert.strictEqual(result.code, 1, 'Should exit 1 for missing tools');
    assert.ok(result.stderr.includes('tools'), 'Should report missing tools field');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('passes on valid agent with all required fields', () => {
    const testDir = createTestDir();
    fs.writeFileSync(path.join(testDir, 'good-agent.md'), '---\nmodel: sonnet\ntools: Read, Write\n---\n# Agent');

    const result = runValidatorWithDir('validate-agents', 'AGENTS_DIR', testDir);
    assert.strictEqual(result.code, 0, 'Should pass for valid agent');
    assert.ok(result.stdout.includes('Validated 1'), 'Should report 1 validated');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('handles frontmatter with BOM and CRLF', () => {
    const testDir = createTestDir();
    const content = '\uFEFF---\r\nmodel: sonnet\r\ntools: Read, Write\r\n---\r\n# Agent';
    fs.writeFileSync(path.join(testDir, 'bom-agent.md'), content);

    const result = runValidatorWithDir('validate-agents', 'AGENTS_DIR', testDir);
    assert.strictEqual(result.code, 0, 'Should handle BOM and CRLF');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('handles frontmatter with colons in values', () => {
    const testDir = createTestDir();
    fs.writeFileSync(path.join(testDir, 'colon-agent.md'), '---\nmodel: sonnet\ntools: Read, Write, Bash\ndescription: Run this: always check: everything\n---\n# Agent');

    const result = runValidatorWithDir('validate-agents', 'AGENTS_DIR', testDir);
    assert.strictEqual(result.code, 0, 'Should handle colons in values');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('skips non-md files', () => {
    const testDir = createTestDir();
    fs.writeFileSync(path.join(testDir, 'readme.txt'), 'Not an agent');
    fs.writeFileSync(path.join(testDir, 'valid.md'), '---\nmodel: sonnet\ntools: Read\n---\n# Agent');

    const result = runValidatorWithDir('validate-agents', 'AGENTS_DIR', testDir);
    assert.strictEqual(result.code, 0, 'Should only validate .md files');
    assert.ok(result.stdout.includes('Validated 1'), 'Should count only .md files');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('exits 0 when directory does not exist', () => {
    const result = runValidatorWithDir('validate-agents', 'AGENTS_DIR', '/nonexistent/dir');
    assert.strictEqual(result.code, 0, 'Should skip when no agents dir');
    assert.ok(result.stdout.includes('skipping'), 'Should say skipping');
  })) passed++; else failed++;

  if (test('rejects agent with empty model value', () => {
    const testDir = createTestDir();
    fs.writeFileSync(path.join(testDir, 'empty.md'), '---\nmodel:\ntools: Read, Write\n---\n# Empty model');
    const result = runValidatorWithDir('validate-agents', 'AGENTS_DIR', testDir);
    assert.strictEqual(result.code, 1, 'Should reject empty model');
    assert.ok(result.stderr.includes('model'), 'Should mention model field');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('rejects agent with empty tools value', () => {
    const testDir = createTestDir();
    fs.writeFileSync(path.join(testDir, 'empty.md'), '---\nmodel: claude-sonnet-4-5-20250929\ntools:\n---\n# Empty tools');
    const result = runValidatorWithDir('validate-agents', 'AGENTS_DIR', testDir);
    assert.strictEqual(result.code, 1, 'Should reject empty tools');
    assert.ok(result.stderr.includes('tools'), 'Should mention tools field');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  // ==========================================
  // validate-hooks.js
  // ==========================================
  console.log('\nvalidate-hooks.js:');

  if (test('passes on real project hooks.json', () => {
    const result = runValidator('validate-hooks');
    assert.strictEqual(result.code, 0, `Should pass, got stderr: ${result.stderr}`);
    assert.ok(result.stdout.includes('Validated'), 'Should output validation count');
  })) passed++; else failed++;

  if (test('exits 0 when hooks.json does not exist', () => {
    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', '/nonexistent/hooks.json');
    assert.strictEqual(result.code, 0, 'Should skip when no hooks.json');
  })) passed++; else failed++;

  if (test('fails on invalid JSON', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    fs.writeFileSync(hooksFile, '{ not valid json }}}');

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 1, 'Should fail on invalid JSON');
    assert.ok(result.stderr.includes('Invalid JSON'), 'Should report invalid JSON');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('fails on invalid event type', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    fs.writeFileSync(hooksFile, JSON.stringify({
      hooks: {
        InvalidEventType: [{ matcher: 'test', hooks: [{ type: 'command', command: 'echo hi' }] }]
      }
    }));

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 1, 'Should fail on invalid event type');
    assert.ok(result.stderr.includes('Invalid event type'), 'Should report invalid event type');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('fails on hook entry missing type field', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    fs.writeFileSync(hooksFile, JSON.stringify({
      hooks: {
        PreToolUse: [{ matcher: 'test', hooks: [{ command: 'echo hi' }] }]
      }
    }));

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 1, 'Should fail on missing type');
    assert.ok(result.stderr.includes('type'), 'Should report missing type');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('fails on hook entry missing command field', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    fs.writeFileSync(hooksFile, JSON.stringify({
      hooks: {
        PreToolUse: [{ matcher: 'test', hooks: [{ type: 'command' }] }]
      }
    }));

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 1, 'Should fail on missing command');
    assert.ok(result.stderr.includes('command'), 'Should report missing command');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('fails on invalid async field type', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    fs.writeFileSync(hooksFile, JSON.stringify({
      hooks: {
        PreToolUse: [{ matcher: 'test', hooks: [{ type: 'command', command: 'echo', async: 'yes' }] }]
      }
    }));

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 1, 'Should fail on non-boolean async');
    assert.ok(result.stderr.includes('async'), 'Should report async type error');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('fails on negative timeout', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    fs.writeFileSync(hooksFile, JSON.stringify({
      hooks: {
        PreToolUse: [{ matcher: 'test', hooks: [{ type: 'command', command: 'echo', timeout: -5 }] }]
      }
    }));

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 1, 'Should fail on negative timeout');
    assert.ok(result.stderr.includes('timeout'), 'Should report timeout error');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('fails on invalid inline JS syntax', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    fs.writeFileSync(hooksFile, JSON.stringify({
      hooks: {
        PreToolUse: [{ matcher: 'test', hooks: [{ type: 'command', command: 'node -e "function {"' }] }]
      }
    }));

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 1, 'Should fail on invalid inline JS');
    assert.ok(result.stderr.includes('invalid inline JS'), 'Should report JS syntax error');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('passes valid inline JS commands', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    fs.writeFileSync(hooksFile, JSON.stringify({
      hooks: {
        PreToolUse: [{ matcher: 'test', hooks: [{ type: 'command', command: 'node -e "console.log(1+2)"' }] }]
      }
    }));

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 0, 'Should pass valid inline JS');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('validates array command format', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    fs.writeFileSync(hooksFile, JSON.stringify({
      hooks: {
        PreToolUse: [{ matcher: 'test', hooks: [{ type: 'command', command: ['node', '-e', 'console.log(1)'] }] }]
      }
    }));

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 0, 'Should accept array command format');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('validates legacy array format', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    fs.writeFileSync(hooksFile, JSON.stringify([
      { matcher: 'test', hooks: [{ type: 'command', command: 'echo ok' }] }
    ]));

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 0, 'Should accept legacy array format');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('fails on matcher missing hooks array', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    fs.writeFileSync(hooksFile, JSON.stringify({
      hooks: {
        PreToolUse: [{ matcher: 'test' }]
      }
    }));

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 1, 'Should fail on missing hooks array');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  // ==========================================
  // validate-skills.js
  // ==========================================
  console.log('\nvalidate-skills.js:');

  if (test('passes on real project skills', () => {
    const result = runValidator('validate-skills');
    assert.strictEqual(result.code, 0, `Should pass, got stderr: ${result.stderr}`);
    assert.ok(result.stdout.includes('Validated'), 'Should output validation count');
  })) passed++; else failed++;

  if (test('exits 0 when directory does not exist', () => {
    const result = runValidatorWithDir('validate-skills', 'SKILLS_DIR', '/nonexistent/dir');
    assert.strictEqual(result.code, 0, 'Should skip when no skills dir');
  })) passed++; else failed++;

  if (test('fails on skill directory without SKILL.md', () => {
    const testDir = createTestDir();
    fs.mkdirSync(path.join(testDir, 'broken-skill'));
    // SKILL.md 파일 없음

    const result = runValidatorWithDir('validate-skills', 'SKILLS_DIR', testDir);
    assert.strictEqual(result.code, 1, 'Should fail on missing SKILL.md');
    assert.ok(result.stderr.includes('Missing SKILL.md'), 'Should report missing SKILL.md');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('fails on empty SKILL.md', () => {
    const testDir = createTestDir();
    const skillDir = path.join(testDir, 'empty-skill');
    fs.mkdirSync(skillDir);
    fs.writeFileSync(path.join(skillDir, 'SKILL.md'), '');

    const result = runValidatorWithDir('validate-skills', 'SKILLS_DIR', testDir);
    assert.strictEqual(result.code, 1, 'Should fail on empty SKILL.md');
    assert.ok(result.stderr.includes('Empty'), 'Should report empty file');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('passes on valid skill directory', () => {
    const testDir = createTestDir();
    const skillDir = path.join(testDir, 'good-skill');
    fs.mkdirSync(skillDir);
    fs.writeFileSync(path.join(skillDir, 'SKILL.md'), '# My Skill\nDescription here.');

    const result = runValidatorWithDir('validate-skills', 'SKILLS_DIR', testDir);
    assert.strictEqual(result.code, 0, 'Should pass for valid skill');
    assert.ok(result.stdout.includes('Validated 1'), 'Should report 1 validated');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('ignores non-directory entries', () => {
    const testDir = createTestDir();
    fs.writeFileSync(path.join(testDir, 'not-a-skill.md'), '# README');
    const skillDir = path.join(testDir, 'real-skill');
    fs.mkdirSync(skillDir);
    fs.writeFileSync(path.join(skillDir, 'SKILL.md'), '# Skill');

    const result = runValidatorWithDir('validate-skills', 'SKILLS_DIR', testDir);
    assert.strictEqual(result.code, 0, 'Should ignore non-directory entries');
    assert.ok(result.stdout.includes('Validated 1'), 'Should count only directories');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('fails on whitespace-only SKILL.md', () => {
    const testDir = createTestDir();
    const skillDir = path.join(testDir, 'blank-skill');
    fs.mkdirSync(skillDir);
    fs.writeFileSync(path.join(skillDir, 'SKILL.md'), '   \n\t\n  ');

    const result = runValidatorWithDir('validate-skills', 'SKILLS_DIR', testDir);
    assert.strictEqual(result.code, 1, 'Should reject whitespace-only SKILL.md');
    assert.ok(result.stderr.includes('Empty file'), 'Should report empty file');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  // ==========================================
  // validate-commands.js
  // ==========================================
  console.log('\nvalidate-commands.js:');

  if (test('passes on real project commands', () => {
    const result = runValidator('validate-commands');
    assert.strictEqual(result.code, 0, `Should pass, got stderr: ${result.stderr}`);
    assert.ok(result.stdout.includes('Validated'), 'Should output validation count');
  })) passed++; else failed++;

  if (test('exits 0 when directory does not exist', () => {
    const result = runValidatorWithDir('validate-commands', 'COMMANDS_DIR', '/nonexistent/dir');
    assert.strictEqual(result.code, 0, 'Should skip when no commands dir');
  })) passed++; else failed++;

  if (test('fails on empty command file', () => {
    const testDir = createTestDir();
    fs.writeFileSync(path.join(testDir, 'empty.md'), '');

    const result = runValidatorWithDir('validate-commands', 'COMMANDS_DIR', testDir);
    assert.strictEqual(result.code, 1, 'Should fail on empty file');
    assert.ok(result.stderr.includes('Empty'), 'Should report empty file');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('passes on valid command files', () => {
    const testDir = createTestDir();
    fs.writeFileSync(path.join(testDir, 'deploy.md'), '# Deploy\nDeploy the application.');
    fs.writeFileSync(path.join(testDir, 'test.md'), '# Test\nRun all tests.');

    const result = runValidatorWithDir('validate-commands', 'COMMANDS_DIR', testDir);
    assert.strictEqual(result.code, 0, 'Should pass for valid commands');
    assert.ok(result.stdout.includes('Validated 2'), 'Should report 2 validated');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('ignores non-md files', () => {
    const testDir = createTestDir();
    fs.writeFileSync(path.join(testDir, 'script.js'), 'console.log(1)');
    fs.writeFileSync(path.join(testDir, 'valid.md'), '# Command');

    const result = runValidatorWithDir('validate-commands', 'COMMANDS_DIR', testDir);
    assert.strictEqual(result.code, 0, 'Should ignore non-md files');
    assert.ok(result.stdout.includes('Validated 1'), 'Should count only .md files');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('detects broken command cross-reference', () => {
    const testDir = createTestDir();
    const agentsDir = createTestDir();
    const skillsDir = createTestDir();
    fs.writeFileSync(path.join(testDir, 'my-cmd.md'), '# Command\nUse `/nonexistent-cmd` to do things.');

    const result = runValidatorWithDirs('validate-commands', {
      COMMANDS_DIR: testDir, AGENTS_DIR: agentsDir, SKILLS_DIR: skillsDir
    });
    assert.strictEqual(result.code, 1, 'Should fail on broken command ref');
    assert.ok(result.stderr.includes('nonexistent-cmd'), 'Should report broken command');
    cleanupTestDir(testDir); cleanupTestDir(agentsDir); cleanupTestDir(skillsDir);
  })) passed++; else failed++;

  if (test('detects broken agent path reference', () => {
    const testDir = createTestDir();
    const agentsDir = createTestDir();
    const skillsDir = createTestDir();
    fs.writeFileSync(path.join(testDir, 'cmd.md'), '# Command\nAgent: `agents/fake-agent.md`');

    const result = runValidatorWithDirs('validate-commands', {
      COMMANDS_DIR: testDir, AGENTS_DIR: agentsDir, SKILLS_DIR: skillsDir
    });
    assert.strictEqual(result.code, 1, 'Should fail on broken agent ref');
    assert.ok(result.stderr.includes('fake-agent'), 'Should report broken agent');
    cleanupTestDir(testDir); cleanupTestDir(agentsDir); cleanupTestDir(skillsDir);
  })) passed++; else failed++;

  if (test('skips references inside fenced code blocks', () => {
    const testDir = createTestDir();
    const agentsDir = createTestDir();
    const skillsDir = createTestDir();
    fs.writeFileSync(path.join(testDir, 'cmd.md'),
      '# Command\n\n```\nagents/example-agent.md\n`/example-cmd`\n```\n');

    const result = runValidatorWithDirs('validate-commands', {
      COMMANDS_DIR: testDir, AGENTS_DIR: agentsDir, SKILLS_DIR: skillsDir
    });
    assert.strictEqual(result.code, 0, 'Should skip refs inside code blocks');
    cleanupTestDir(testDir); cleanupTestDir(agentsDir); cleanupTestDir(skillsDir);
  })) passed++; else failed++;

  if (test('detects broken workflow agent reference', () => {
    const testDir = createTestDir();
    const agentsDir = createTestDir();
    const skillsDir = createTestDir();
    fs.writeFileSync(path.join(agentsDir, 'planner.md'), '---\nmodel: sonnet\ntools: Read\n---\n# A');
    fs.writeFileSync(path.join(testDir, 'cmd.md'), '# Command\nWorkflow:\nplanner -> ghost-agent');

    const result = runValidatorWithDirs('validate-commands', {
      COMMANDS_DIR: testDir, AGENTS_DIR: agentsDir, SKILLS_DIR: skillsDir
    });
    assert.strictEqual(result.code, 1, 'Should fail on broken workflow agent');
    assert.ok(result.stderr.includes('ghost-agent'), 'Should report broken workflow agent');
    cleanupTestDir(testDir); cleanupTestDir(agentsDir); cleanupTestDir(skillsDir);
  })) passed++; else failed++;

  if (test('skips command references on creates: lines', () => {
    const testDir = createTestDir();
    const agentsDir = createTestDir();
    const skillsDir = createTestDir();
    // "Creates: `/new-table`"는 /new-table을 깨진 참조로 플래그하지 않아야 함
    fs.writeFileSync(path.join(testDir, 'gen.md'),
      '# Generator\n\n→ Creates: `/new-table`\nWould create: `/new-endpoint`');

    const result = runValidatorWithDirs('validate-commands', {
      COMMANDS_DIR: testDir, AGENTS_DIR: agentsDir, SKILLS_DIR: skillsDir
    });
    assert.strictEqual(result.code, 0, 'Should skip creates: lines');
    cleanupTestDir(testDir); cleanupTestDir(agentsDir); cleanupTestDir(skillsDir);
  })) passed++; else failed++;

  if (test('accepts valid cross-reference between commands', () => {
    const testDir = createTestDir();
    const agentsDir = createTestDir();
    const skillsDir = createTestDir();
    fs.writeFileSync(path.join(testDir, 'build.md'), '# Build\nSee also `/deploy` for deployment.');
    fs.writeFileSync(path.join(testDir, 'deploy.md'), '# Deploy\nRun `/build` first.');

    const result = runValidatorWithDirs('validate-commands', {
      COMMANDS_DIR: testDir, AGENTS_DIR: agentsDir, SKILLS_DIR: skillsDir
    });
    assert.strictEqual(result.code, 0, 'Should accept valid cross-refs');
    assert.ok(result.stdout.includes('Validated 2'), 'Should validate both');
    cleanupTestDir(testDir); cleanupTestDir(agentsDir); cleanupTestDir(skillsDir);
  })) passed++; else failed++;

  if (test('checks references in unclosed code blocks', () => {
    const testDir = createTestDir();
    const agentsDir = createTestDir();
    const skillsDir = createTestDir();
    // 닫히지 않은 코드 블록: ``` regex가 이를 제거하지 않으므로 내부 참조를 확인합니다.
    fs.writeFileSync(path.join(testDir, 'bad.md'),
      '# Command\n\n```\n`/phantom-cmd`\nno closing block');

    const result = runValidatorWithDirs('validate-commands', {
      COMMANDS_DIR: testDir, AGENTS_DIR: agentsDir, SKILLS_DIR: skillsDir
    });
    // 닫히지 않은 코드 블록은 제거되지 않으므로 내부 참조가 검증됩니다.
    assert.strictEqual(result.code, 1, 'Should check refs in unclosed code blocks');
    assert.ok(result.stderr.includes('phantom-cmd'), 'Should report broken ref from unclosed block');
    cleanupTestDir(testDir); cleanupTestDir(agentsDir); cleanupTestDir(skillsDir);
  })) passed++; else failed++;

  if (test('captures ALL command references on a single line (multi-ref)', () => {
    const testDir = createTestDir();
    const agentsDir = createTestDir();
    const skillsDir = createTestDir();
    // 두 개의 명령 참조가 있는 라인 — 둘 다 감지되어야 함
    fs.writeFileSync(path.join(testDir, 'multi.md'),
      '# Multi\nUse `/ghost-a` and `/ghost-b` together.');

    const result = runValidatorWithDirs('validate-commands', {
      COMMANDS_DIR: testDir, AGENTS_DIR: agentsDir, SKILLS_DIR: skillsDir
    });
    assert.strictEqual(result.code, 1, 'Should fail on broken refs');
    // ghost-a와 ghost-b 모두 보고되어야 함 (greedy regex 버그 수정 확인)
    assert.ok(result.stderr.includes('ghost-a'), 'Should report first ref /ghost-a');
    assert.ok(result.stderr.includes('ghost-b'), 'Should report second ref /ghost-b');
    cleanupTestDir(testDir); cleanupTestDir(agentsDir); cleanupTestDir(skillsDir);
  })) passed++; else failed++;

  if (test('captures three command refs on one line', () => {
    const testDir = createTestDir();
    const agentsDir = createTestDir();
    const skillsDir = createTestDir();
    fs.writeFileSync(path.join(testDir, 'triple.md'),
      '# Triple\nChain `/alpha`, `/beta`, and `/gamma` in order.');

    const result = runValidatorWithDirs('validate-commands', {
      COMMANDS_DIR: testDir, AGENTS_DIR: agentsDir, SKILLS_DIR: skillsDir
    });
    assert.strictEqual(result.code, 1, 'Should fail on all three broken refs');
    assert.ok(result.stderr.includes('alpha'), 'Should report /alpha');
    assert.ok(result.stderr.includes('beta'), 'Should report /beta');
    assert.ok(result.stderr.includes('gamma'), 'Should report /gamma');
    cleanupTestDir(testDir); cleanupTestDir(agentsDir); cleanupTestDir(skillsDir);
  })) passed++; else failed++;

  if (test('multi-ref line with one valid and one invalid ref', () => {
    const testDir = createTestDir();
    const agentsDir = createTestDir();
    const skillsDir = createTestDir();
    // "real-cmd"는 존재, "fake-cmd"는 존재하지 않음
    fs.writeFileSync(path.join(testDir, 'real-cmd.md'), '# Real\nA real command.');
    fs.writeFileSync(path.join(testDir, 'mixed.md'),
      '# Mixed\nRun `/real-cmd` then `/fake-cmd`.');

    const result = runValidatorWithDirs('validate-commands', {
      COMMANDS_DIR: testDir, AGENTS_DIR: agentsDir, SKILLS_DIR: skillsDir
    });
    assert.strictEqual(result.code, 1, 'Should fail for the fake ref');
    assert.ok(result.stderr.includes('fake-cmd'), 'Should report /fake-cmd');
    // real-cmd는 오류에 나타나지 않아야 함
    assert.ok(!result.stderr.includes('real-cmd'), 'Should not report valid /real-cmd');
    cleanupTestDir(testDir); cleanupTestDir(agentsDir); cleanupTestDir(skillsDir);
  })) passed++; else failed++;

  if (test('creates: line with multiple refs skips entire line', () => {
    const testDir = createTestDir();
    const agentsDir = createTestDir();
    const skillsDir = createTestDir();
    // "Creates:" 라인의 두 참조 모두 건너뛰어야 함
    fs.writeFileSync(path.join(testDir, 'gen.md'),
      '# Generator\nCreates: `/new-a` and `/new-b`');

    const result = runValidatorWithDirs('validate-commands', {
      COMMANDS_DIR: testDir, AGENTS_DIR: agentsDir, SKILLS_DIR: skillsDir
    });
    assert.strictEqual(result.code, 0, 'Should skip all refs on creates: line');
    cleanupTestDir(testDir); cleanupTestDir(agentsDir); cleanupTestDir(skillsDir);
  })) passed++; else failed++;

  if (test('validates valid workflow diagram with known agents', () => {
    const testDir = createTestDir();
    const agentsDir = createTestDir();
    const skillsDir = createTestDir();
    fs.writeFileSync(path.join(agentsDir, 'planner.md'), '---\nmodel: sonnet\ntools: Read\n---\n# P');
    fs.writeFileSync(path.join(agentsDir, 'reviewer.md'), '---\nmodel: sonnet\ntools: Read\n---\n# R');
    fs.writeFileSync(path.join(testDir, 'flow.md'), '# Workflow\n\nplanner -> reviewer');

    const result = runValidatorWithDirs('validate-commands', {
      COMMANDS_DIR: testDir, AGENTS_DIR: agentsDir, SKILLS_DIR: skillsDir
    });
    assert.strictEqual(result.code, 0, 'Should pass on valid workflow');
    cleanupTestDir(testDir); cleanupTestDir(agentsDir); cleanupTestDir(skillsDir);
  })) passed++; else failed++;

  // ==========================================
  // validate-rules.js
  // ==========================================
  console.log('\nvalidate-rules.js:');

  if (test('passes on real project rules', () => {
    const result = runValidator('validate-rules');
    assert.strictEqual(result.code, 0, `Should pass, got stderr: ${result.stderr}`);
    assert.ok(result.stdout.includes('Validated'), 'Should output validation count');
  })) passed++; else failed++;

  if (test('exits 0 when directory does not exist', () => {
    const result = runValidatorWithDir('validate-rules', 'RULES_DIR', '/nonexistent/dir');
    assert.strictEqual(result.code, 0, 'Should skip when no rules dir');
  })) passed++; else failed++;

  if (test('fails on empty rule file', () => {
    const testDir = createTestDir();
    fs.writeFileSync(path.join(testDir, 'empty.md'), '');

    const result = runValidatorWithDir('validate-rules', 'RULES_DIR', testDir);
    assert.strictEqual(result.code, 1, 'Should fail on empty rule file');
    assert.ok(result.stderr.includes('Empty'), 'Should report empty file');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('passes on valid rule files', () => {
    const testDir = createTestDir();
    fs.writeFileSync(path.join(testDir, 'coding.md'), '# Coding Rules\nUse immutability.');

    const result = runValidatorWithDir('validate-rules', 'RULES_DIR', testDir);
    assert.strictEqual(result.code, 0, 'Should pass for valid rules');
    assert.ok(result.stdout.includes('Validated 1'), 'Should report 1 validated');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('fails on whitespace-only rule file', () => {
    const testDir = createTestDir();
    fs.writeFileSync(path.join(testDir, 'blank.md'), '   \n\t\n  ');

    const result = runValidatorWithDir('validate-rules', 'RULES_DIR', testDir);
    assert.strictEqual(result.code, 1, 'Should reject whitespace-only rule file');
    assert.ok(result.stderr.includes('Empty'), 'Should report empty file');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('validates rules in subdirectories recursively', () => {
    const testDir = createTestDir();
    const subDir = path.join(testDir, 'sub');
    fs.mkdirSync(subDir);
    fs.writeFileSync(path.join(testDir, 'top.md'), '# Top Level Rule');
    fs.writeFileSync(path.join(subDir, 'nested.md'), '# Nested Rule');

    const result = runValidatorWithDir('validate-rules', 'RULES_DIR', testDir);
    assert.strictEqual(result.code, 0, 'Should validate nested rules');
    assert.ok(result.stdout.includes('Validated 2'), 'Should find both rules');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  // ==========================================
  // Round 19: 공백 및 엣지 케이스 테스트
  // ==========================================

  // --- validate-hooks.js 공백/null 엣지 케이스 ---
  console.log('\nvalidate-hooks.js (whitespace edge cases):');

  if (test('rejects whitespace-only command string', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    fs.writeFileSync(hooksFile, JSON.stringify({
      hooks: {
        PreToolUse: [{ matcher: 'test', hooks: [{ type: 'command', command: '   \t  ' }] }]
      }
    }));

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 1, 'Should reject whitespace-only command');
    assert.ok(result.stderr.includes('command'), 'Should report command field error');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('rejects null command value', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    fs.writeFileSync(hooksFile, JSON.stringify({
      hooks: {
        PreToolUse: [{ matcher: 'test', hooks: [{ type: 'command', command: null }] }]
      }
    }));

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 1, 'Should reject null command');
    assert.ok(result.stderr.includes('command'), 'Should report command field error');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('rejects numeric command value', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    fs.writeFileSync(hooksFile, JSON.stringify({
      hooks: {
        PreToolUse: [{ matcher: 'test', hooks: [{ type: 'command', command: 42 }] }]
      }
    }));

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 1, 'Should reject numeric command');
    assert.ok(result.stderr.includes('command'), 'Should report command field error');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  // --- validate-agents.js 공백 엣지 케이스 ---
  console.log('\nvalidate-agents.js (whitespace edge cases):');

  if (test('rejects agent with whitespace-only model value', () => {
    const testDir = createTestDir();
    fs.writeFileSync(path.join(testDir, 'ws-model.md'), '---\nmodel:   \t  \ntools: Read, Write\n---\n# Agent');

    const result = runValidatorWithDir('validate-agents', 'AGENTS_DIR', testDir);
    assert.strictEqual(result.code, 1, 'Should reject whitespace-only model');
    assert.ok(result.stderr.includes('model'), 'Should report model field error');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('rejects agent with whitespace-only tools value', () => {
    const testDir = createTestDir();
    fs.writeFileSync(path.join(testDir, 'ws-tools.md'), '---\nmodel: sonnet\ntools:   \t  \n---\n# Agent');

    const result = runValidatorWithDir('validate-agents', 'AGENTS_DIR', testDir);
    assert.strictEqual(result.code, 1, 'Should reject whitespace-only tools');
    assert.ok(result.stderr.includes('tools'), 'Should report tools field error');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('accepts agent with extra unknown frontmatter fields', () => {
    const testDir = createTestDir();
    fs.writeFileSync(path.join(testDir, 'extra.md'), '---\nmodel: sonnet\ntools: Read, Write\ncustom_field: some value\nauthor: test\n---\n# Agent');

    const result = runValidatorWithDir('validate-agents', 'AGENTS_DIR', testDir);
    assert.strictEqual(result.code, 0, 'Should accept extra unknown fields');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('rejects agent with invalid model value', () => {
    const testDir = createTestDir();
    fs.writeFileSync(path.join(testDir, 'bad-model.md'), '---\nmodel: gpt-4\ntools: Read\n---\n# Agent');

    const result = runValidatorWithDir('validate-agents', 'AGENTS_DIR', testDir);
    assert.strictEqual(result.code, 1, 'Should reject invalid model');
    assert.ok(result.stderr.includes('Invalid model'), 'Should report invalid model');
    assert.ok(result.stderr.includes('gpt-4'), 'Should show the invalid value');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  // --- validate-commands.js 추가 엣지 케이스 ---
  console.log('\nvalidate-commands.js (additional edge cases):');

  if (test('reports all invalid agents in mixed agent references', () => {
    const testDir = createTestDir();
    const agentsDir = createTestDir();
    const skillsDir = createTestDir();
    fs.writeFileSync(path.join(agentsDir, 'real-agent.md'), '---\nmodel: sonnet\ntools: Read\n---\n# A');
    fs.writeFileSync(path.join(testDir, 'cmd.md'),
      '# Cmd\nSee agents/real-agent.md and agents/fake-one.md and agents/fake-two.md');

    const result = runValidatorWithDirs('validate-commands', {
      COMMANDS_DIR: testDir, AGENTS_DIR: agentsDir, SKILLS_DIR: skillsDir
    });
    assert.strictEqual(result.code, 1, 'Should fail on invalid agent refs');
    assert.ok(result.stderr.includes('fake-one'), 'Should report first invalid agent');
    assert.ok(result.stderr.includes('fake-two'), 'Should report second invalid agent');
    assert.ok(!result.stderr.includes('real-agent'), 'Should NOT report valid agent');
    cleanupTestDir(testDir); cleanupTestDir(agentsDir); cleanupTestDir(skillsDir);
  })) passed++; else failed++;

  if (test('validates workflow with hyphenated agent names', () => {
    const testDir = createTestDir();
    const agentsDir = createTestDir();
    const skillsDir = createTestDir();
    fs.writeFileSync(path.join(agentsDir, 'tdd-guide.md'), '---\nmodel: sonnet\ntools: Read\n---\n# T');
    fs.writeFileSync(path.join(agentsDir, 'code-reviewer.md'), '---\nmodel: sonnet\ntools: Read\n---\n# C');
    fs.writeFileSync(path.join(testDir, 'flow.md'), '# Workflow\n\ntdd-guide -> code-reviewer');

    const result = runValidatorWithDirs('validate-commands', {
      COMMANDS_DIR: testDir, AGENTS_DIR: agentsDir, SKILLS_DIR: skillsDir
    });
    assert.strictEqual(result.code, 0, 'Should pass on hyphenated agent names in workflow');
    cleanupTestDir(testDir); cleanupTestDir(agentsDir); cleanupTestDir(skillsDir);
  })) passed++; else failed++;

  if (test('detects skill directory reference warning', () => {
    const testDir = createTestDir();
    const agentsDir = createTestDir();
    const skillsDir = createTestDir();
    // 존재하지 않는 skill 디렉토리 참조
    fs.writeFileSync(path.join(testDir, 'cmd.md'),
      '# Command\nSee skills/nonexistent-skill/ for details.');

    const result = runValidatorWithDirs('validate-commands', {
      COMMANDS_DIR: testDir, AGENTS_DIR: agentsDir, SKILLS_DIR: skillsDir
    });
    // 통과해야 함 (경고는 exit 1을 유발하지 않음), 하지만 stderr에 경고가 있어야 함
    assert.strictEqual(result.code, 0, 'Skill warnings should not cause failure');
    assert.ok(result.stdout.includes('warning'), 'Should report warning count');
    cleanupTestDir(testDir); cleanupTestDir(agentsDir); cleanupTestDir(skillsDir);
  })) passed++; else failed++;

  // ==========================================
  // Round 22: Hook 스키마 엣지 케이스 및 빈 디렉토리 경로
  // ==========================================

  // --- validate-hooks.js: 스키마 엣지 케이스 ---
  console.log('\nvalidate-hooks.js (schema edge cases):');

  if (test('rejects event type value that is not an array', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    fs.writeFileSync(hooksFile, JSON.stringify({
      hooks: { PreToolUse: 'not-an-array' }
    }));

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 1, 'Should fail on non-array event type value');
    assert.ok(result.stderr.includes('must be an array'), 'Should report must be an array');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('rejects matcher entry that is null', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    fs.writeFileSync(hooksFile, JSON.stringify({
      hooks: { PreToolUse: [null] }
    }));

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 1, 'Should fail on null matcher entry');
    assert.ok(result.stderr.includes('is not an object'), 'Should report not an object');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('rejects matcher entry that is a string', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    fs.writeFileSync(hooksFile, JSON.stringify({
      hooks: { PreToolUse: ['just-a-string'] }
    }));

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 1, 'Should fail on string matcher entry');
    assert.ok(result.stderr.includes('is not an object'), 'Should report not an object');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('rejects top-level data that is a string', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    fs.writeFileSync(hooksFile, '"just a string"');

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 1, 'Should fail on string data');
    assert.ok(result.stderr.includes('must be an object or array'), 'Should report must be object or array');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('rejects top-level data that is a number', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    fs.writeFileSync(hooksFile, '42');

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 1, 'Should fail on numeric data');
    assert.ok(result.stderr.includes('must be an object or array'), 'Should report must be object or array');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('rejects empty string command', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    fs.writeFileSync(hooksFile, JSON.stringify({
      hooks: {
        PreToolUse: [{ matcher: 'test', hooks: [{ type: 'command', command: '' }] }]
      }
    }));

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 1, 'Should reject empty string command');
    assert.ok(result.stderr.includes('command'), 'Should report command field error');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('rejects empty array command', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    fs.writeFileSync(hooksFile, JSON.stringify({
      hooks: {
        PreToolUse: [{ matcher: 'test', hooks: [{ type: 'command', command: [] }] }]
      }
    }));

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 1, 'Should reject empty array command');
    assert.ok(result.stderr.includes('command'), 'Should report command field error');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('rejects array command with non-string elements', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    fs.writeFileSync(hooksFile, JSON.stringify({
      hooks: {
        PreToolUse: [{ matcher: 'test', hooks: [{ type: 'command', command: ['node', 123, null] }] }]
      }
    }));

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 1, 'Should reject non-string array elements');
    assert.ok(result.stderr.includes('command'), 'Should report command field error');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('rejects non-string type field', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    fs.writeFileSync(hooksFile, JSON.stringify({
      hooks: {
        PreToolUse: [{ matcher: 'test', hooks: [{ type: 42, command: 'echo hi' }] }]
      }
    }));

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 1, 'Should reject non-string type');
    assert.ok(result.stderr.includes('type'), 'Should report type field error');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('rejects non-number timeout type', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    fs.writeFileSync(hooksFile, JSON.stringify({
      hooks: {
        PreToolUse: [{ matcher: 'test', hooks: [{ type: 'command', command: 'echo', timeout: 'fast' }] }]
      }
    }));

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 1, 'Should reject string timeout');
    assert.ok(result.stderr.includes('timeout'), 'Should report timeout type error');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('accepts timeout of exactly 0', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    fs.writeFileSync(hooksFile, JSON.stringify({
      hooks: {
        PreToolUse: [{ matcher: 'test', hooks: [{ type: 'command', command: 'echo', timeout: 0 }] }]
      }
    }));

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 0, 'Should accept timeout of 0');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('validates object format without wrapping hooks key', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    // data.hooks가 undefined이므로 data 자체로 대체
    fs.writeFileSync(hooksFile, JSON.stringify({
      PreToolUse: [{ matcher: 'test', hooks: [{ type: 'command', command: 'echo ok' }] }]
    }));

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 0, 'Should accept object format without hooks wrapper');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  // --- validate-hooks.js: 레거시 포맷 오류 경로 ---
  console.log('\nvalidate-hooks.js (legacy format errors):');

  if (test('legacy format: rejects matcher missing matcher field', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    fs.writeFileSync(hooksFile, JSON.stringify([
      { hooks: [{ type: 'command', command: 'echo ok' }] }
    ]));

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 1, 'Should fail on missing matcher in legacy format');
    assert.ok(result.stderr.includes('matcher'), 'Should report missing matcher');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('legacy format: rejects matcher missing hooks array', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    fs.writeFileSync(hooksFile, JSON.stringify([
      { matcher: 'test' }
    ]));

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 1, 'Should fail on missing hooks array in legacy format');
    assert.ok(result.stderr.includes('hooks'), 'Should report missing hooks');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  // --- validate-agents.js: 빈 디렉토리 ---
  console.log('\nvalidate-agents.js (empty directory):');

  if (test('passes on empty agents directory', () => {
    const testDir = createTestDir();
    // .md 파일 없음, 그냥 빈 디렉토리

    const result = runValidatorWithDir('validate-agents', 'AGENTS_DIR', testDir);
    assert.strictEqual(result.code, 0, 'Should pass on empty directory');
    assert.ok(result.stdout.includes('Validated 0'), 'Should report 0 validated');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  // --- validate-commands.js: 공백만 있는 파일 ---
  console.log('\nvalidate-commands.js (whitespace edge cases):');

  if (test('fails on whitespace-only command file', () => {
    const testDir = createTestDir();
    fs.writeFileSync(path.join(testDir, 'blank.md'), '   \n\t\n  ');

    const result = runValidatorWithDir('validate-commands', 'COMMANDS_DIR', testDir);
    assert.strictEqual(result.code, 1, 'Should reject whitespace-only command file');
    assert.ok(result.stderr.includes('Empty'), 'Should report empty file');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('accepts valid skill directory reference', () => {
    const testDir = createTestDir();
    const agentsDir = createTestDir();
    const skillsDir = createTestDir();
    // 일치하는 skill 디렉토리 생성
    fs.mkdirSync(path.join(skillsDir, 'my-skill'));
    fs.writeFileSync(path.join(testDir, 'cmd.md'),
      '# Command\nSee skills/my-skill/ for details.');

    const result = runValidatorWithDirs('validate-commands', {
      COMMANDS_DIR: testDir, AGENTS_DIR: agentsDir, SKILLS_DIR: skillsDir
    });
    assert.strictEqual(result.code, 0, 'Should pass on valid skill reference');
    assert.ok(!result.stdout.includes('warning'), 'Should have no warnings');
    cleanupTestDir(testDir); cleanupTestDir(agentsDir); cleanupTestDir(skillsDir);
  })) passed++; else failed++;

  // --- validate-rules.js: 유효/유효하지 않은 파일 혼합 ---
  console.log('\nvalidate-rules.js (mixed files):');

  if (test('fails on mix of valid and empty rule files', () => {
    const testDir = createTestDir();
    fs.writeFileSync(path.join(testDir, 'good.md'), '# Good Rule\nContent here.');
    fs.writeFileSync(path.join(testDir, 'bad.md'), '');

    const result = runValidatorWithDir('validate-rules', 'RULES_DIR', testDir);
    assert.strictEqual(result.code, 1, 'Should fail when any rule is empty');
    assert.ok(result.stderr.includes('bad.md'), 'Should report the bad file');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  // ── Round 27: hook 검증 엣지 케이스 ──
  console.log('\nvalidate-hooks.js (Round 27 edge cases):');

  if (test('rejects array command with empty string element', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    fs.writeFileSync(hooksFile, JSON.stringify({
      hooks: {
        PreToolUse: [{ matcher: 'test', hooks: [{ type: 'command', command: ['node', '', 'script.js'] }] }]
      }
    }));

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 1, 'Should reject array with empty string element');
    assert.ok(result.stderr.includes('command'), 'Should report command field error');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('rejects negative timeout', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    fs.writeFileSync(hooksFile, JSON.stringify({
      hooks: {
        PreToolUse: [{ matcher: 'test', hooks: [{ type: 'command', command: 'echo hi', timeout: -5 }] }]
      }
    }));

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 1, 'Should reject negative timeout');
    assert.ok(result.stderr.includes('timeout'), 'Should report timeout error');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('rejects non-boolean async field', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    fs.writeFileSync(hooksFile, JSON.stringify({
      hooks: {
        PostToolUse: [{ matcher: 'test', hooks: [{ type: 'command', command: 'echo ok', async: 'yes' }] }]
      }
    }));

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 1, 'Should reject non-boolean async');
    assert.ok(result.stderr.includes('async'), 'Should report async type error');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('reports correct index for error in deeply nested hook', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    const manyHooks = [];
    for (let i = 0; i < 5; i++) {
      manyHooks.push({ type: 'command', command: 'echo ok' });
    }
    // 인덱스 5에 잘못된 hook 추가
    manyHooks.push({ type: 'command', command: '' });
    fs.writeFileSync(hooksFile, JSON.stringify({
      hooks: {
        PreToolUse: [{ matcher: 'test', hooks: manyHooks }]
      }
    }));

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 1, 'Should fail on invalid hook at high index');
    assert.ok(result.stderr.includes('hooks[5]'), 'Should report correct hook index 5');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('validates node -e with escaped quotes in inline JS', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    fs.writeFileSync(hooksFile, JSON.stringify({
      hooks: {
        PreToolUse: [{ matcher: 'test', hooks: [{ type: 'command', command: 'node -e "const x = 1 + 2; process.exit(0)"' }] }]
      }
    }));

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 0, 'Should pass valid multi-statement inline JS');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('accepts multiple valid event types in single hooks file', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    fs.writeFileSync(hooksFile, JSON.stringify({
      hooks: {
        PreToolUse: [{ matcher: 'test', hooks: [{ type: 'command', command: 'echo pre' }] }],
        PostToolUse: [{ matcher: 'test', hooks: [{ type: 'command', command: 'echo post' }] }],
        Stop: [{ matcher: 'test', hooks: [{ type: 'command', command: 'echo stop' }] }]
      }
    }));

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 0, 'Should accept multiple valid event types');
    assert.ok(result.stdout.includes('3'), 'Should report 3 matchers validated');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  // ── Round 27: command 검증 엣지 케이스 ──
  console.log('\nvalidate-commands.js (Round 27 edge cases):');

  if (test('validates multiple command refs on same non-creates line', () => {
    const testDir = createTestDir();
    const agentsDir = createTestDir();
    const skillsDir = createTestDir();
    // 두 개의 유효한 명령 생성
    fs.writeFileSync(path.join(testDir, 'cmd-a.md'), '# Command A\nBasic command.');
    fs.writeFileSync(path.join(testDir, 'cmd-b.md'), '# Command B\nBasic command.');
    // 한 라인에 두 명령을 모두 참조하는 세 번째 명령 생성
    fs.writeFileSync(path.join(testDir, 'cmd-c.md'),
      '# Command C\nUse `/cmd-a` and `/cmd-b` together.');

    const result = runValidatorWithDirs('validate-commands', {
      COMMANDS_DIR: testDir, AGENTS_DIR: agentsDir, SKILLS_DIR: skillsDir
    });
    assert.strictEqual(result.code, 0, 'Should pass when multiple refs on same line are all valid');
    cleanupTestDir(testDir); cleanupTestDir(agentsDir); cleanupTestDir(skillsDir);
  })) passed++; else failed++;

  if (test('fails when one of multiple refs on same line is invalid', () => {
    const testDir = createTestDir();
    const agentsDir = createTestDir();
    const skillsDir = createTestDir();
    // cmd-a만 존재
    fs.writeFileSync(path.join(testDir, 'cmd-a.md'), '# Command A\nBasic command.');
    // cmd-c가 cmd-a(유효)와 cmd-z(잘못됨)를 한 라인에서 참조
    fs.writeFileSync(path.join(testDir, 'cmd-c.md'),
      '# Command C\nUse `/cmd-a` and `/cmd-z` together.');

    const result = runValidatorWithDirs('validate-commands', {
      COMMANDS_DIR: testDir, AGENTS_DIR: agentsDir, SKILLS_DIR: skillsDir
    });
    assert.strictEqual(result.code, 1, 'Should fail when any ref is invalid');
    assert.ok(result.stderr.includes('cmd-z'), 'Should report the invalid reference');
    cleanupTestDir(testDir); cleanupTestDir(agentsDir); cleanupTestDir(skillsDir);
  })) passed++; else failed++;

  if (test('code blocks are stripped before checking references', () => {
    const testDir = createTestDir();
    const agentsDir = createTestDir();
    const skillsDir = createTestDir();
    // 코드 블록 내부의 참조는 검증되지 않아야 함
    fs.writeFileSync(path.join(testDir, 'cmd-x.md'),
      '# Command X\n```\n`/nonexistent-cmd` in code block\n```\nEnd.');

    const result = runValidatorWithDirs('validate-commands', {
      COMMANDS_DIR: testDir, AGENTS_DIR: agentsDir, SKILLS_DIR: skillsDir
    });
    assert.strictEqual(result.code, 0, 'Should ignore command refs inside code blocks');
    cleanupTestDir(testDir); cleanupTestDir(agentsDir); cleanupTestDir(skillsDir);
  })) passed++; else failed++;

  // --- validate-skills.js: 유효/유효하지 않은 디렉토리 혼합 ---
  console.log('\nvalidate-skills.js (mixed dirs):');

  if (test('fails on mix of valid and invalid skill directories', () => {
    const testDir = createTestDir();
    // 유효한 skill
    const goodSkill = path.join(testDir, 'good-skill');
    fs.mkdirSync(goodSkill);
    fs.writeFileSync(path.join(goodSkill, 'SKILL.md'), '# Good Skill');
    // SKILL.md 누락
    const badSkill = path.join(testDir, 'bad-skill');
    fs.mkdirSync(badSkill);
    // 빈 SKILL.md
    const emptySkill = path.join(testDir, 'empty-skill');
    fs.mkdirSync(emptySkill);
    fs.writeFileSync(path.join(emptySkill, 'SKILL.md'), '');

    const result = runValidatorWithDir('validate-skills', 'SKILLS_DIR', testDir);
    assert.strictEqual(result.code, 1, 'Should fail when any skill is invalid');
    assert.ok(result.stderr.includes('bad-skill'), 'Should report missing SKILL.md');
    assert.ok(result.stderr.includes('empty-skill'), 'Should report empty SKILL.md');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  // ── Round 30: validate-commands skill 경고 및 workflow 엣지 케이스 ──
  console.log('\nRound 30: validate-commands (skill warnings):');

  if (test('warns (not errors) when skill directory reference is not found', () => {
    const testDir = createTestDir();
    const agentsDir = createTestDir();
    const skillsDir = createTestDir();
    // 경로(skills/name/) 포맷을 통해 skill을 참조하는 명령 생성
    // 하지만 skill이 존재하지 않음 — 오류가 아닌 경고가 발생해야 함
    fs.writeFileSync(path.join(testDir, 'cmd-a.md'),
      '# Command A\nSee skills/nonexistent-skill/ for details.');

    const result = runValidatorWithDirs('validate-commands', {
      COMMANDS_DIR: testDir, AGENTS_DIR: agentsDir, SKILLS_DIR: skillsDir
    });
    // Skill 디렉토리 참조는 오류가 아닌 경고를 생성 — exit 0
    assert.strictEqual(result.code, 0, 'Skill path references should warn, not error');
    cleanupTestDir(testDir); cleanupTestDir(agentsDir); cleanupTestDir(skillsDir);
  })) passed++; else failed++;

  if (test('passes when command has no slash references at all', () => {
    const testDir = createTestDir();
    const agentsDir = createTestDir();
    const skillsDir = createTestDir();
    fs.writeFileSync(path.join(testDir, 'cmd-simple.md'),
      '# Simple Command\nThis command has no references to other commands.');

    const result = runValidatorWithDirs('validate-commands', {
      COMMANDS_DIR: testDir, AGENTS_DIR: agentsDir, SKILLS_DIR: skillsDir
    });
    assert.strictEqual(result.code, 0, 'Should pass with no references');
    cleanupTestDir(testDir); cleanupTestDir(agentsDir); cleanupTestDir(skillsDir);
  })) passed++; else failed++;

  console.log('\nRound 30: validate-agents (model validation):');

  if (test('rejects agent with unrecognized model value', () => {
    const testDir = createTestDir();
    fs.writeFileSync(path.join(testDir, 'bad-model.md'),
      '---\nmodel: gpt-4\ntools: Read, Write\n---\n# Bad Model Agent');

    const result = runValidatorWithDir('validate-agents', 'AGENTS_DIR', testDir);
    assert.strictEqual(result.code, 1, 'Should reject unrecognized model');
    assert.ok(result.stderr.includes('gpt-4'), 'Should mention the invalid model');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('accepts all valid model values (haiku, sonnet, opus)', () => {
    const testDir = createTestDir();
    fs.writeFileSync(path.join(testDir, 'haiku.md'),
      '---\nmodel: haiku\ntools: Read\n---\n# Haiku Agent');
    fs.writeFileSync(path.join(testDir, 'sonnet.md'),
      '---\nmodel: sonnet\ntools: Read, Write\n---\n# Sonnet Agent');
    fs.writeFileSync(path.join(testDir, 'opus.md'),
      '---\nmodel: opus\ntools: Read, Write, Bash\n---\n# Opus Agent');

    const result = runValidatorWithDir('validate-agents', 'AGENTS_DIR', testDir);
    assert.strictEqual(result.code, 0, 'All valid models should pass');
    assert.ok(result.stdout.includes('3'), 'Should validate 3 agent files');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  // ── Round 32: 빈 frontmatter 및 엣지 케이스 ──
  console.log('\nRound 32: validate-agents (empty frontmatter):');

  if (test('rejects agent with empty frontmatter block (no key-value pairs)', () => {
    const testDir = createTestDir();
    // --- 마커 사이의 빈 라인은 유효하지만 비어 있는 frontmatter 블록을 생성합니다.
    fs.writeFileSync(path.join(testDir, 'empty-fm.md'), '---\n\n---\n# Agent with empty frontmatter');

    const result = runValidatorWithDir('validate-agents', 'AGENTS_DIR', testDir);
    assert.strictEqual(result.code, 1, 'Should reject empty frontmatter');
    assert.ok(result.stderr.includes('model'), 'Should report missing model');
    assert.ok(result.stderr.includes('tools'), 'Should report missing tools');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('rejects agent with no content between --- markers (Missing frontmatter)', () => {
    const testDir = createTestDir();
    // 빈 라인 없는 ---\n--- → regex가 매칭되지 않음 → "Missing frontmatter"
    fs.writeFileSync(path.join(testDir, 'no-fm.md'), '---\n---\n# Agent');

    const result = runValidatorWithDir('validate-agents', 'AGENTS_DIR', testDir);
    assert.strictEqual(result.code, 1, 'Should reject missing frontmatter');
    assert.ok(result.stderr.includes('Missing frontmatter'), 'Should report missing frontmatter');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('rejects agent with partial frontmatter (only model, no tools)', () => {
    const testDir = createTestDir();
    fs.writeFileSync(path.join(testDir, 'partial.md'), '---\nmodel: haiku\n---\n# Partial agent');

    const result = runValidatorWithDir('validate-agents', 'AGENTS_DIR', testDir);
    assert.strictEqual(result.code, 1, 'Should reject partial frontmatter');
    assert.ok(result.stderr.includes('tools'), 'Should report missing tools');
    assert.ok(!result.stderr.includes('model'), 'Should NOT report model (it is present)');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('handles multiple agents where only one is invalid', () => {
    const testDir = createTestDir();
    fs.writeFileSync(path.join(testDir, 'good.md'), '---\nmodel: sonnet\ntools: Read\n---\n# Good');
    fs.writeFileSync(path.join(testDir, 'bad.md'), '---\nmodel: invalid-model\ntools: Read\n---\n# Bad');

    const result = runValidatorWithDir('validate-agents', 'AGENTS_DIR', testDir);
    assert.strictEqual(result.code, 1, 'Should fail when any agent is invalid');
    assert.ok(result.stderr.includes('bad.md'), 'Should identify the bad file');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  console.log('\nRound 32: validate-rules (non-file entries):');

  if (test('skips directory entries even if named with .md extension', () => {
    const testDir = createTestDir();
    // "tricky.md"라는 디렉토리 생성 — stat.isFile()에 의해 건너뛰어져야 함
    fs.mkdirSync(path.join(testDir, 'tricky.md'));
    fs.writeFileSync(path.join(testDir, 'real.md'), '# A real rule');

    const result = runValidatorWithDir('validate-rules', 'RULES_DIR', testDir);
    assert.strictEqual(result.code, 0, 'Should skip directory entries');
    assert.ok(result.stdout.includes('Validated 1'), 'Should count only the real file');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('handles deeply nested rule in subdirectory', () => {
    const testDir = createTestDir();
    const deepDir = path.join(testDir, 'cat1', 'sub1');
    fs.mkdirSync(deepDir, { recursive: true });
    fs.writeFileSync(path.join(deepDir, 'deep-rule.md'), '# Deep nested rule');

    const result = runValidatorWithDir('validate-rules', 'RULES_DIR', testDir);
    assert.strictEqual(result.code, 0, 'Should validate deeply nested rules');
    assert.ok(result.stdout.includes('Validated 1'), 'Should find the nested rule');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  console.log('\nRound 32: validate-commands (agent reference with valid workflow):');

  if (test('passes workflow with three chained agents', () => {
    const testDir = createTestDir();
    const agentsDir = createTestDir();
    const skillsDir = createTestDir();
    fs.writeFileSync(path.join(agentsDir, 'planner.md'), '---\nmodel: sonnet\ntools: Read\n---\n# P');
    fs.writeFileSync(path.join(agentsDir, 'tdd-guide.md'), '---\nmodel: sonnet\ntools: Read\n---\n# T');
    fs.writeFileSync(path.join(agentsDir, 'code-reviewer.md'), '---\nmodel: sonnet\ntools: Read\n---\n# C');
    fs.writeFileSync(path.join(testDir, 'flow.md'), '# Flow\n\nplanner -> tdd-guide -> code-reviewer');

    const result = runValidatorWithDirs('validate-commands', {
      COMMANDS_DIR: testDir, AGENTS_DIR: agentsDir, SKILLS_DIR: skillsDir
    });
    assert.strictEqual(result.code, 0, 'Should pass on valid 3-agent workflow');
    cleanupTestDir(testDir); cleanupTestDir(agentsDir); cleanupTestDir(skillsDir);
  })) passed++; else failed++;

  if (test('detects broken agent in middle of workflow chain', () => {
    const testDir = createTestDir();
    const agentsDir = createTestDir();
    const skillsDir = createTestDir();
    fs.writeFileSync(path.join(agentsDir, 'planner.md'), '---\nmodel: sonnet\ntools: Read\n---\n# P');
    fs.writeFileSync(path.join(agentsDir, 'code-reviewer.md'), '---\nmodel: sonnet\ntools: Read\n---\n# C');
    // missing-agent는 생성되지 않음
    fs.writeFileSync(path.join(testDir, 'flow.md'), '# Flow\n\nplanner -> missing-agent -> code-reviewer');

    const result = runValidatorWithDirs('validate-commands', {
      COMMANDS_DIR: testDir, AGENTS_DIR: agentsDir, SKILLS_DIR: skillsDir
    });
    assert.strictEqual(result.code, 1, 'Should detect broken agent in workflow chain');
    assert.ok(result.stderr.includes('missing-agent'), 'Should report the missing agent');
    cleanupTestDir(testDir); cleanupTestDir(agentsDir); cleanupTestDir(skillsDir);
  })) passed++; else failed++;

  // ── Round 42: 대소문자 구분, 콜론 앞 공백, 디렉토리 누락, 빈 matchers ──
  console.log('\nRound 42: validate-agents (case sensitivity):');

  if (test('rejects uppercase model value (case-sensitive check)', () => {
    const testDir = createTestDir();
    fs.writeFileSync(path.join(testDir, 'upper.md'), '---\nmodel: Haiku\ntools: Read\n---\n# Uppercase model');

    const result = runValidatorWithDir('validate-agents', 'AGENTS_DIR', testDir);
    assert.strictEqual(result.code, 1, 'Should reject capitalized model');
    assert.ok(result.stderr.includes('Invalid model'), 'Should report invalid model');
    assert.ok(result.stderr.includes('Haiku'), 'Should show the rejected value');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('handles space before colon in frontmatter key', () => {
    const testDir = createTestDir();
    // "model : sonnet" — 콜론 앞 공백. extractFrontmatter는 indexOf(':') + trim()을 사용합니다.
    fs.writeFileSync(path.join(testDir, 'space.md'), '---\nmodel : sonnet\ntools : Read, Write\n---\n# Agent with space-colon');

    const result = runValidatorWithDir('validate-agents', 'AGENTS_DIR', testDir);
    assert.strictEqual(result.code, 0, 'Should accept space before colon (trim handles it)');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  console.log('\nRound 42: validate-commands (missing agents dir):');

  if (test('flags agent path references when AGENTS_DIR does not exist', () => {
    const testDir = createTestDir();
    const skillsDir = createTestDir();
    // AGENTS_DIR가 존재하지 않는 경로를 가리킴 → validAgents 세트가 비어 있게 됨
    fs.writeFileSync(path.join(testDir, 'cmd.md'), '# Command\nSee agents/planner.md for details.');

    const result = runValidatorWithDirs('validate-commands', {
      COMMANDS_DIR: testDir, AGENTS_DIR: '/nonexistent/agents-dir', SKILLS_DIR: skillsDir
    });
    assert.strictEqual(result.code, 1, 'Should fail when agents dir missing but agent referenced');
    assert.ok(result.stderr.includes('planner'), 'Should report the unresolvable agent reference');
    cleanupTestDir(testDir); cleanupTestDir(skillsDir);
  })) passed++; else failed++;

  console.log('\nRound 42: validate-hooks (empty matchers array):');

  if (test('accepts event type with empty matchers array', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    fs.writeFileSync(hooksFile, JSON.stringify({
      hooks: {
        PreToolUse: []
      }
    }));

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 0, 'Should accept empty matchers array');
    assert.ok(result.stdout.includes('Validated 0'), 'Should report 0 matchers');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  // ── Round 47: 이스케이프 시퀀스 및 frontmatter 엣지 케이스 ──
  console.log('\nRound 47: validate-hooks (inline JS escape sequences):');

  if (test('validates inline JS with mixed escape sequences (newline + escaped quote)', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    // JSON 파싱 후의 Command 값: node -e "var a = \"ok\"\nconsole.log(a)"
    // Regex 캡처: var a = \"ok\"\nconsole.log(a)
    // unescape 체인 후: var a = "ok"\nconsole.log(a) (실제 개행) — 유효한 JS
    fs.writeFileSync(hooksFile, JSON.stringify({
      hooks: {
        PreToolUse: [{ matcher: 'test', hooks: [{ type: 'command',
          command: 'node -e "var a = \\"ok\\"\\nconsole.log(a)"' }] }]
      }
    }));

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 0, 'Should handle escaped quotes and newline separators');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('rejects inline JS with syntax error after unescaping', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    // unescape 후: var x = { — 닫는 중괄호 누락
    fs.writeFileSync(hooksFile, JSON.stringify({
      hooks: {
        PreToolUse: [{ matcher: 'test', hooks: [{ type: 'command',
          command: 'node -e "var x = {"' }] }]
      }
    }));

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 1, 'Should reject JS syntax error after unescaping');
    assert.ok(result.stderr.includes('invalid inline JS'), 'Should report inline JS error');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  console.log('\nRound 47: validate-agents (frontmatter lines without colon):');

  if (test('silently ignores frontmatter line without colon', () => {
    const testDir = createTestDir();
    // "just some text" 라인에는 콜론이 없음 — 건너뛰어야 하며 크래시를 유발하지 않아야 함
    fs.writeFileSync(path.join(testDir, 'mixed.md'),
      '---\nmodel: sonnet\njust some text without colon\ntools: Read\n---\n# Agent');

    const result = runValidatorWithDir('validate-agents', 'AGENTS_DIR', testDir);
    assert.strictEqual(result.code, 0, 'Should ignore lines without colon in frontmatter');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  // ── Round 52: 명령 인라인 backtick 참조, workflow 공백, 코드 전용 규칙 ──
  console.log('\nRound 52: validate-commands (inline backtick refs):');

  if (test('validates command refs inside inline backticks (not stripped by code block removal)', () => {
    const testDir = createTestDir();
    const agentsDir = createTestDir();
    const skillsDir = createTestDir();
    fs.writeFileSync(path.join(testDir, 'deploy.md'), '# Deploy\nDeploy the app.');
    // 인라인 backtick 참조 `/deploy`는 검증되어야 함 (fenced 블록만 제거됨)
    fs.writeFileSync(path.join(testDir, 'workflow.md'),
      '# Workflow\nFirst run `/deploy` to deploy the app.');

    const result = runValidatorWithDirs('validate-commands', {
      COMMANDS_DIR: testDir, AGENTS_DIR: agentsDir, SKILLS_DIR: skillsDir
    });
    assert.strictEqual(result.code, 0, 'Inline backtick command refs should be validated');
    cleanupTestDir(testDir); cleanupTestDir(agentsDir); cleanupTestDir(skillsDir);
  })) passed++; else failed++;

  console.log('\nRound 52: validate-commands (workflow whitespace):');

  if (test('validates workflow arrows with irregular whitespace', () => {
    const testDir = createTestDir();
    const agentsDir = createTestDir();
    const skillsDir = createTestDir();
    fs.writeFileSync(path.join(agentsDir, 'planner.md'), '# Planner');
    fs.writeFileSync(path.join(agentsDir, 'reviewer.md'), '# Reviewer');
    // 세 가지 workflow 라인: 공백 없음, 이중 공백, 탭으로 구분
    fs.writeFileSync(path.join(testDir, 'flow.md'),
      '# Workflow\n\nplanner->reviewer\nplanner  ->  reviewer');

    const result = runValidatorWithDirs('validate-commands', {
      COMMANDS_DIR: testDir, AGENTS_DIR: agentsDir, SKILLS_DIR: skillsDir
    });
    assert.strictEqual(result.code, 0, 'Workflow arrows with irregular whitespace should be valid');
    cleanupTestDir(testDir); cleanupTestDir(agentsDir); cleanupTestDir(skillsDir);
  })) passed++; else failed++;

  console.log('\nRound 52: validate-rules (code-only content):');

  if (test('passes rule file containing only a fenced code block', () => {
    const testDir = createTestDir();
    fs.writeFileSync(path.join(testDir, 'code-only.md'),
      '```javascript\nfunction example() {\n  return true;\n}\n```');

    const result = runValidatorWithDir('validate-rules', 'RULES_DIR', testDir);
    assert.strictEqual(result.code, 0, 'Rule with only code block should pass (non-empty)');
    assert.ok(result.stdout.includes('Validated 1'), 'Should count the code-only file');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  // ── Round 57: readFileSync 오류 경로, statSync catch 블록, 인접 코드 블록 ──
  console.log('\nRound 57: validate-skills.js (SKILL.md is a directory — readFileSync error):');

  if (test('fails gracefully when SKILL.md is a directory instead of a file', () => {
    const testDir = createTestDir();
    const skillDir = path.join(testDir, 'dir-skill');
    fs.mkdirSync(skillDir);
    // SKILL.md를 파일이 아닌 DIRECTORY로 생성 — existsSync는 true를 반환하지만
    // readFileSync는 EISDIR을 던져 catch 블록을 실행함 (33-37 라인)
    fs.mkdirSync(path.join(skillDir, 'SKILL.md'));

    const result = runValidatorWithDir('validate-skills', 'SKILLS_DIR', testDir);
    assert.strictEqual(result.code, 1, 'Should fail when SKILL.md is a directory');
    assert.ok(result.stderr.includes('dir-skill'), 'Should report the problematic skill');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  console.log('\nRound 57: validate-rules.js (broken symlink — statSync catch block):');

  if (test('reports error for broken symlink .md file in rules directory', () => {
    const testDir = createTestDir();
    // 먼저 유효한 규칙 생성
    fs.writeFileSync(path.join(testDir, 'valid.md'), '# Valid Rule');
    // 깨진 심볼릭 링크 생성 (타겟이 존재하지 않음)
    // statSync는 심볼릭 링크를 따라가다가 ENOENT를 던져 catch를 실행함 (35-38 라인)
    try {
      fs.symlinkSync('/nonexistent/target.md', path.join(testDir, 'broken.md'));
    } catch {
      // 심볼릭 링크를 지원하지 않는 시스템에서는 건너뜀
      console.log('    (skipped — symlinks not supported)');
      cleanupTestDir(testDir);
      return;
    }

    const result = runValidatorWithDir('validate-rules', 'RULES_DIR', testDir);
    assert.strictEqual(result.code, 1, 'Should fail on broken symlink');
    assert.ok(result.stderr.includes('broken.md'), 'Should report the broken symlink file');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  console.log('\nRound 57: validate-commands.js (adjacent code blocks both stripped):');

  if (test('strips multiple adjacent code blocks before checking references', () => {
    const testDir = createTestDir();
    const agentsDir = createTestDir();
    const skillsDir = createTestDir();
    // 깨진 참조가 있는 두 개의 인접한 코드 블록 — 둘 다 제거되어야 함
    fs.writeFileSync(path.join(testDir, 'multi-blocks.md'),
      '# Multi Block\n\n' +
      '```\n`/phantom-a` in first block\n```\n\n' +
      'Content between blocks\n\n' +
      '```\n`/phantom-b` in second block\nagents/ghost-agent.md\n```\n\n' +
      'Final content');

    const result = runValidatorWithDirs('validate-commands', {
      COMMANDS_DIR: testDir, AGENTS_DIR: agentsDir, SKILLS_DIR: skillsDir
    });
    assert.strictEqual(result.code, 0,
      'Both code blocks should be stripped — no broken refs reported');
    assert.ok(!result.stderr.includes('phantom-a'), 'First block ref should be stripped');
    assert.ok(!result.stderr.includes('phantom-b'), 'Second block ref should be stripped');
    assert.ok(!result.stderr.includes('ghost-agent'), 'Agent ref in second block should be stripped');
    cleanupTestDir(testDir); cleanupTestDir(agentsDir); cleanupTestDir(skillsDir);
  })) passed++; else failed++;

  // ── Round 58: readFileSync catch 블록, colonIdx 엣지 케이스, 객체 형태의 명령 ──
  console.log('\nRound 58: validate-agents.js (unreadable agent file — readFileSync catch):');

  if (test('reports error when agent .md file is unreadable (chmod 000)', () => {
    // Windows이거나 root로 실행 중인 경우 건너뜀 (권한 설정이 작동하지 않음)
    if (process.platform === 'win32' || (process.getuid && process.getuid() === 0)) {
      console.log('    (skipped — not supported on this platform)');
      return;
    }
    const testDir = createTestDir();
    const agentFile = path.join(testDir, 'locked.md');
    fs.writeFileSync(agentFile, '---\nmodel: sonnet\ntools: Read\n---\n# Agent');
    fs.chmodSync(agentFile, 0o000);

    try {
      const result = runValidatorWithDir('validate-agents', 'AGENTS_DIR', testDir);
      assert.strictEqual(result.code, 1, 'Should exit 1 on read error');
      assert.ok(result.stderr.includes('locked.md'), 'Should mention the unreadable file');
    } finally {
      fs.chmodSync(agentFile, 0o644);
      cleanupTestDir(testDir);
    }
  })) passed++; else failed++;

  console.log('\nRound 58: validate-agents.js (frontmatter line with colon at position 0):');

  if (test('rejects agent when required field key has colon at position 0 (no key name)', () => {
    const testDir = createTestDir();
    fs.writeFileSync(path.join(testDir, 'bad-colon.md'),
      '---\n:sonnet\ntools: Read\n---\n# Agent with leading colon');

    const result = runValidatorWithDir('validate-agents', 'AGENTS_DIR', testDir);
    assert.strictEqual(result.code, 1, 'Should fail — model field is missing (colon at idx 0 skipped)');
    assert.ok(result.stderr.includes('model'), 'Should report missing model field');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  console.log('\nRound 58: validate-hooks.js (command is a plain object — not string or array):');

  if (test('rejects hook entry where command is a plain object', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    fs.writeFileSync(hooksFile, JSON.stringify({
      hooks: {
        PreToolUse: [{ matcher: 'test', hooks: [{ type: 'command', command: { run: 'echo hi' } }] }]
      }
    }));

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 1, 'Should reject object command (not string or array)');
    assert.ok(result.stderr.includes('command'), 'Should report invalid command field');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  // ── Round 63: 객체 포맷의 누락된 matcher, 읽을 수 없는 명령 파일, 빈 명령 디렉토리 ──
  console.log('\nRound 63: validate-hooks.js (object-format matcher missing matcher field):');

  if (test('rejects object-format matcher entry missing matcher field', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    // 객체 포맷: matcher 엔트리에 hooks 배열은 있지만 matcher 필드가 없음
    fs.writeFileSync(hooksFile, JSON.stringify({
      hooks: {
        PreToolUse: [{ hooks: [{ type: 'command', command: 'echo ok' }] }]
      }
    }));

    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 1, 'Should fail on missing matcher field in object format');
    assert.ok(result.stderr.includes("missing 'matcher' field"), 'Should report missing matcher field');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  console.log('\nRound 63: validate-commands.js (unreadable command file):');

  if (test('reports error when command .md file is unreadable (chmod 000)', () => {
    if (process.platform === 'win32' || (process.getuid && process.getuid() === 0)) {
      console.log('    (skipped — not supported on this platform)');
      return;
    }
    const testDir = createTestDir();
    const cmdFile = path.join(testDir, 'locked.md');
    fs.writeFileSync(cmdFile, '# Locked Command');
    fs.chmodSync(cmdFile, 0o000);

    try {
      const result = runValidatorWithDirs('validate-commands', {
        COMMANDS_DIR: testDir, AGENTS_DIR: '/nonexistent', SKILLS_DIR: '/nonexistent'
      });
      assert.strictEqual(result.code, 1, 'Should exit 1 on read error');
      assert.ok(result.stderr.includes('locked.md'), 'Should mention the unreadable file');
    } finally {
      fs.chmodSync(cmdFile, 0o644);
      cleanupTestDir(testDir);
    }
  })) passed++; else failed++;

  console.log('\nRound 63: validate-commands.js (empty commands directory):');

  if (test('passes on empty commands directory (no .md files)', () => {
    const testDir = createTestDir();
    // .md가 아닌 파일만 있음 — 검증할 .md 파일이 없음
    fs.writeFileSync(path.join(testDir, 'readme.txt'), 'not a command');

    const result = runValidatorWithDirs('validate-commands', {
      COMMANDS_DIR: testDir, AGENTS_DIR: '/nonexistent', SKILLS_DIR: '/nonexistent'
    });
    assert.strictEqual(result.code, 0, 'Should pass on empty commands directory');
    assert.ok(result.stdout.includes('Validated 0'), 'Should report 0 validated');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  // ── Round 65: 규칙 및 스킬을 위한 빈 디렉토리 ──
  console.log('\nRound 65: validate-rules.js (empty directory — no .md files):');

  if (test('passes on rules directory with no .md files (Validated 0)', () => {
    const testDir = createTestDir();
    // .md가 아닌 파일만 있음 — readdirSync 필터 결과가 빈 배열
    fs.writeFileSync(path.join(testDir, 'notes.txt'), 'not a rule');
    fs.writeFileSync(path.join(testDir, 'config.json'), '{}');

    const result = runValidatorWithDir('validate-rules', 'RULES_DIR', testDir);
    assert.strictEqual(result.code, 0, 'Should pass on empty rules directory');
    assert.ok(result.stdout.includes('Validated 0'), 'Should report 0 validated rule files');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  console.log('\nRound 65: validate-skills.js (empty directory — no subdirectories):');

  if (test('passes on skills directory with only files, no subdirectories (Validated 0)', () => {
    const testDir = createTestDir();
    // 파일만 있고 하위 디렉토리가 없음 — isDirectory 필터 결과가 빈 배열
    fs.writeFileSync(path.join(testDir, 'README.md'), '# Skills');
    fs.writeFileSync(path.join(testDir, '.gitkeep'), '');

    const result = runValidatorWithDir('validate-skills', 'SKILLS_DIR', testDir);
    assert.strictEqual(result.code, 0, 'Should pass on skills directory with no subdirectories');
    assert.ok(result.stdout.includes('Validated 0'), 'Should report 0 validated skill directories');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  // ── Round 70: validate-commands.js "would create:" 라인 건너뛰기 ──
  console.log('\nRound 70: validate-commands.js (would create: skip):');

  if (test('skips command references on "would create:" lines', () => {
    const testDir = createTestDir();
    const agentsDir = createTestDir();
    const skillsDir = createTestDir();
    // "Would create:"는 80번 라인의 regex에 의해 확인되는 대체 형태입니다:
    //   if (/creates:|would create:/i.test(line)) continue;
    // 이전에 "creates:"만 테스트되었습니다 (Round 20). "Would create:"는 regex의 두 번째 
    // 양자택일을 실행합니다.
    fs.writeFileSync(path.join(testDir, 'gen-cmd.md'),
      '# Generator Command\n\nWould create: `/phantom-cmd` in your project.\n\nThis is safe.');

    const result = runValidatorWithDirs('validate-commands', {
      COMMANDS_DIR: testDir, AGENTS_DIR: agentsDir, SKILLS_DIR: skillsDir
    });
    assert.strictEqual(result.code, 0, 'Should skip "would create:" lines');
    assert.ok(!result.stderr.includes('phantom-cmd'), 'Should not flag ref on "would create:" line');
    cleanupTestDir(testDir); cleanupTestDir(agentsDir); cleanupTestDir(skillsDir);
  })) passed++; else failed++;

  // ── Round 72: validate-hooks.js async/timeout 타입 검증 ──
  console.log('\nRound 72: validate-hooks.js (async and timeout type validation):');

  if (test('rejects hook with non-boolean async field', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    fs.writeFileSync(hooksFile, JSON.stringify({
      PreToolUse: [{
        matcher: 'Write',
        hooks: [{
          type: 'command',
          command: 'echo test',
          async: 'yes'  // 문자열이 아닌 boolean이어야 함
        }]
      }]
    }));
    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 1, 'Should fail on non-boolean async');
    assert.ok(result.stderr.includes('async'), 'Should mention async in error');
    assert.ok(result.stderr.includes('boolean'), 'Should mention boolean type');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  if (test('rejects hook with negative timeout value', () => {
    const testDir = createTestDir();
    const hooksFile = path.join(testDir, 'hooks.json');
    fs.writeFileSync(hooksFile, JSON.stringify({
      PostToolUse: [{
        matcher: 'Edit',
        hooks: [{
          type: 'command',
          command: 'echo test',
          timeout: -5  // 0 이상이어야 함
        }]
      }]
    }));
    const result = runValidatorWithDir('validate-hooks', 'HOOKS_FILE', hooksFile);
    assert.strictEqual(result.code, 1, 'Should fail on negative timeout');
    assert.ok(result.stderr.includes('timeout'), 'Should mention timeout in error');
    assert.ok(result.stderr.includes('non-negative'), 'Should mention non-negative');
    cleanupTestDir(testDir);
  })) passed++; else failed++;

  // ── Round 73: validate-commands.js skill 디렉토리 statSync catch ──
  console.log('\nRound 73: validate-commands.js (unreadable skill entry — statSync catch):');

  if (test('skips unreadable skill directory entries without error (broken symlink)', () => {
    const testDir = createTestDir();
    const agentsDir = createTestDir();
    const skillsDir = createTestDir();

    // 유효한 skill 디렉토리 하나와 깨진 심볼릭 링크 하나를 생성
    const validSkill = path.join(skillsDir, 'valid-skill');
    fs.mkdirSync(validSkill, { recursive: true });
    // 깨진 심볼릭 링크: 타겟이 존재하지 않음 — statSync가 ENOENT를 던짐
    const brokenLink = path.join(skillsDir, 'broken-skill');
    fs.symlinkSync('/nonexistent/target/path', brokenLink);

    // 유효한 skill을 참조하는 명령 (해결되어야 함)
    fs.writeFileSync(path.join(testDir, 'cmd.md'),
      '# Command\nSee skills/valid-skill/ for details.');

    const result = runValidatorWithDirs('validate-commands', {
      COMMANDS_DIR: testDir, AGENTS_DIR: agentsDir, SKILLS_DIR: skillsDir
    });
--- End of content ---

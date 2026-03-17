# Go 마이크로서비스 — 프로젝트 CLAUDE.md

> PostgreSQL, gRPC, Docker를 사용하는 Go 마이크로서비스의 실제 예제입니다.
> 프로젝트 루트에 복사하여 서비스에 맞게 커스터마이징하세요.

## 프로젝트 개요

**스택:** Go 1.22+, PostgreSQL, gRPC + REST (grpc-gateway), Docker, sqlc (타입 안전 SQL), Wire (의존성 주입)

**아키텍처:** 도메인, 리포지토리, 서비스, 핸들러 레이어를 갖춘 클린 아키텍처. gRPC를 주 전송 수단으로 사용하고 외부 클라이언트를 위한 REST 게이트웨이 제공.

## 핵심 규칙

### Go 컨벤션

- Effective Go 및 Go Code Review Comments 가이드 준수
- 에러 래핑에는 `errors.New` / `fmt.Errorf`와 `%w` 사용 — 에러 문자열 매칭 금지
- `init()` 함수 사용 금지 — `main()` 또는 생성자에서 명시적으로 초기화
- 전역 가변 상태 금지 — 생성자를 통해 의존성 전달
- Context는 첫 번째 매개변수여야 하며 모든 레이어에 전파

### 데이터베이스

- 모든 쿼리는 `queries/`에 일반 SQL로 작성 — sqlc가 타입 안전한 Go 코드 생성
- 마이그레이션은 golang-migrate를 사용하여 `migrations/`에 관리 — 데이터베이스 직접 변경 금지
- 다단계 작업에는 `pgx.Tx`를 통한 트랜잭션 사용
- 모든 쿼리는 매개변수화된 플레이스홀더(`$1`, `$2`) 사용 — 문자열 포맷팅 금지

### 에러 처리

- 에러를 반환하고, 패닉 금지 — 패닉은 진정으로 복구 불가능한 상황에만 사용
- 컨텍스트와 함께 에러 래핑: `fmt.Errorf("creating user: %w", err)`
- 비즈니스 로직을 위한 센티넬 에러는 `domain/errors.go`에 정의
- 핸들러 레이어에서 도메인 에러를 gRPC 상태 코드로 매핑

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

- 코드나 주석에 이모지 사용 금지
- 내보내는 타입과 함수에는 문서 주석 필수
- 함수는 50줄 이하로 유지 — 헬퍼 함수로 분리
- 여러 케이스가 있는 모든 로직에 테이블 기반 테스트 사용
- 시그널 채널에는 `bool`이 아닌 `struct{}` 사용

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

### 리포지토리 인터페이스

```go
type UserRepository interface {
    Create(ctx context.Context, user *User) error
    FindByID(ctx context.Context, id uuid.UUID) (*User, error)
    FindByEmail(ctx context.Context, email string) (*User, error)
    Update(ctx context.Context, user *User) error
    Delete(ctx context.Context, id uuid.UUID) error
}
```

### 의존성 주입을 사용하는 서비스

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

### 테이블 기반 테스트

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
/go-test             # TDD workflow for Go
/go-review           # Go-specific code review
/go-build            # Fix build errors
```

### 테스트 명령어

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

## ECC 워크플로우

```bash
# Planning
/plan "Add rate limiting to user endpoints"

# Development
/go-test                  # TDD with Go-specific patterns

# Review
/go-review                # Go idioms, error handling, concurrency
/security-scan            # Secrets and vulnerabilities

# Before merge
go vet ./...
staticcheck ./...
```

## Git 워크플로우

- `feat:` 새 기능, `fix:` 버그 수정, `refactor:` 코드 변경
- `main`에서 기능 브랜치 생성, PR 필수
- CI: `go vet`, `staticcheck`, `go test -race`, `golangci-lint`
- 배포: CI에서 Docker 이미지 빌드 후 Kubernetes에 배포

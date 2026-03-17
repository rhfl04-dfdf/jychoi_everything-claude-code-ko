# Rust API 서비스 — 프로젝트 CLAUDE.md

> Axum, PostgreSQL, Docker를 사용하는 Rust API 서비스의 실제 예제입니다.
> 프로젝트 루트에 복사하여 서비스에 맞게 커스터마이징하세요.

## 프로젝트 개요

**스택:** Rust 1.78+, Axum (웹 프레임워크), SQLx (비동기 데이터베이스), PostgreSQL, Tokio (비동기 런타임), Docker

**아키텍처:** 핸들러 → 서비스 → 리포지토리 분리를 갖춘 레이어드 아키텍처. HTTP에는 Axum, 컴파일 타임에 타입 검증되는 SQL에는 SQLx, 크로스 커팅 관심사에는 Tower 미들웨어 사용.

## 핵심 규칙

### Rust 컨벤션

- 라이브러리 에러에는 `thiserror`, 바이너리 크레이트나 테스트에만 `anyhow` 사용
- 프로덕션 코드에서 `.unwrap()` 또는 `.expect()` 사용 금지 — `?`로 에러 전파
- 함수 매개변수에는 `String` 대신 `&str` 선호; 소유권이 이전될 때 `String` 반환
- `#![deny(clippy::all, clippy::pedantic)]`와 함께 `clippy` 사용 — 모든 경고 수정
- 모든 공개 타입에 `Debug` 파생; `Clone`, `PartialEq`는 필요할 때만 파생
- `// SAFETY:` 주석 없이 `unsafe` 블록 사용 금지

### 데이터베이스

- 모든 쿼리는 SQLx `query!` 또는 `query_as!` 매크로 사용 — 스키마에 대해 컴파일 타임 검증
- 마이그레이션은 `sqlx migrate`를 사용하여 `migrations/`에 관리 — 데이터베이스 직접 변경 금지
- `sqlx::Pool<Postgres>`를 공유 상태로 사용 — 요청마다 연결 생성 금지
- 모든 쿼리는 매개변수화된 플레이스홀더(`$1`, `$2`) 사용 — 문자열 포맷팅 금지

```rust
// BAD: String interpolation (SQL injection risk)
let q = format!("SELECT * FROM users WHERE id = '{}'", id);

// GOOD: Parameterized query, compile-time checked
let user = sqlx::query_as!(User, "SELECT * FROM users WHERE id = $1", id)
    .fetch_optional(&pool)
    .await?;
```

### 에러 처리

- 모듈별로 `thiserror`를 사용하여 도메인 에러 열거형 정의
- `IntoResponse`를 통해 에러를 HTTP 응답으로 매핑 — 내부 세부 정보 노출 금지
- 구조화된 로깅에는 `tracing` 사용 — `println!`이나 `eprintln!` 사용 금지

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum AppError {
    #[error("Resource not found")]
    NotFound,
    #[error("Validation failed: {0}")]
    Validation(String),
    #[error("Unauthorized")]
    Unauthorized,
    #[error(transparent)]
    Internal(#[from] anyhow::Error),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, message) = match &self {
            Self::NotFound => (StatusCode::NOT_FOUND, self.to_string()),
            Self::Validation(msg) => (StatusCode::BAD_REQUEST, msg.clone()),
            Self::Unauthorized => (StatusCode::UNAUTHORIZED, self.to_string()),
            Self::Internal(err) => {
                tracing::error!(?err, "internal error");
                (StatusCode::INTERNAL_SERVER_ERROR, "Internal error".into())
            }
        };
        (status, Json(json!({ "error": message }))).into_response()
    }
}
```

### 테스트

- 단위 테스트는 각 소스 파일 내 `#[cfg(test)]` 모듈에 작성
- 통합 테스트는 실제 PostgreSQL(Testcontainers 또는 Docker)을 사용하여 `tests/` 디렉토리에 작성
- 자동 마이그레이션과 롤백을 포함한 데이터베이스 테스트에는 `#[sqlx::test]` 사용
- 외부 서비스 목킹에는 `mockall` 또는 `wiremock` 사용

### 코드 스타일

- 최대 줄 길이: 100자 (rustfmt로 강제)
- 임포트 그룹화: `std`, 외부 크레이트, `crate`/`super` — 빈 줄로 구분
- 모듈: 모듈당 하나의 파일, `mod.rs`는 재내보내기(re-export)에만 사용
- 타입: PascalCase, 함수/변수: snake_case, 상수: UPPER_SNAKE_CASE

## 파일 구조

```
src/
  main.rs              # Entrypoint, server setup, graceful shutdown
  lib.rs               # Re-exports for integration tests
  config.rs            # Environment config with envy or figment
  router.rs            # Axum router with all routes
  middleware/
    auth.rs            # JWT extraction and validation
    logging.rs         # Request/response tracing
  handlers/
    mod.rs             # Route handlers (thin — delegate to services)
    users.rs
    orders.rs
  services/
    mod.rs             # Business logic
    users.rs
    orders.rs
  repositories/
    mod.rs             # Database access (SQLx queries)
    users.rs
    orders.rs
  domain/
    mod.rs             # Domain types, error enums
    user.rs
    order.rs
migrations/
  001_create_users.sql
  002_create_orders.sql
tests/
  common/mod.rs        # Shared test helpers, test server setup
  api_users.rs         # Integration tests for user endpoints
  api_orders.rs        # Integration tests for order endpoints
```

## 주요 패턴

### 핸들러 (얇게)

```rust
async fn create_user(
    State(ctx): State<AppState>,
    Json(payload): Json<CreateUserRequest>,
) -> Result<(StatusCode, Json<UserResponse>), AppError> {
    let user = ctx.user_service.create(payload).await?;
    Ok((StatusCode::CREATED, Json(UserResponse::from(user))))
}
```

### 서비스 (비즈니스 로직)

```rust
impl UserService {
    pub async fn create(&self, req: CreateUserRequest) -> Result<User, AppError> {
        if self.repo.find_by_email(&req.email).await?.is_some() {
            return Err(AppError::Validation("Email already registered".into()));
        }

        let password_hash = hash_password(&req.password)?;
        let user = self.repo.insert(&req.email, &req.name, &password_hash).await?;

        Ok(user)
    }
}
```

### 리포지토리 (데이터 접근)

```rust
impl UserRepository {
    pub async fn find_by_email(&self, email: &str) -> Result<Option<User>, sqlx::Error> {
        sqlx::query_as!(User, "SELECT * FROM users WHERE email = $1", email)
            .fetch_optional(&self.pool)
            .await
    }

    pub async fn insert(
        &self,
        email: &str,
        name: &str,
        password_hash: &str,
    ) -> Result<User, sqlx::Error> {
        sqlx::query_as!(
            User,
            r#"INSERT INTO users (email, name, password_hash)
               VALUES ($1, $2, $3) RETURNING *"#,
            email, name, password_hash,
        )
        .fetch_one(&self.pool)
        .await
    }
}
```

### 통합 테스트

```rust
#[tokio::test]
async fn test_create_user() {
    let app = spawn_test_app().await;

    let response = app
        .client
        .post(&format!("{}/api/v1/users", app.address))
        .json(&json!({
            "email": "alice@example.com",
            "name": "Alice",
            "password": "securepassword123"
        }))
        .send()
        .await
        .expect("Failed to send request");

    assert_eq!(response.status(), StatusCode::CREATED);
    let body: serde_json::Value = response.json().await.unwrap();
    assert_eq!(body["email"], "alice@example.com");
}

#[tokio::test]
async fn test_create_user_duplicate_email() {
    let app = spawn_test_app().await;
    // Create first user
    create_test_user(&app, "alice@example.com").await;
    // Attempt duplicate
    let response = create_user_request(&app, "alice@example.com").await;
    assert_eq!(response.status(), StatusCode::BAD_REQUEST);
}
```

## 환경 변수

```bash
# Server
HOST=0.0.0.0
PORT=8080
RUST_LOG=info,tower_http=debug

# Database
DATABASE_URL=postgres://user:pass@localhost:5432/myapp

# Auth
JWT_SECRET=your-secret-key-min-32-chars
JWT_EXPIRY_HOURS=24

# Optional
CORS_ALLOWED_ORIGINS=http://localhost:3000
```

## 테스트 전략

```bash
# Run all tests
cargo test

# Run with output
cargo test -- --nocapture

# Run specific test module
cargo test api_users

# Check coverage (requires cargo-llvm-cov)
cargo llvm-cov --html
open target/llvm-cov/html/index.html

# Lint
cargo clippy -- -D warnings

# Format check
cargo fmt -- --check
```

## ECC 워크플로우

```bash
# Planning
/plan "Add order fulfillment with Stripe payment"

# Development with TDD
/tdd                    # cargo test-based TDD workflow

# Review
/code-review            # Rust-specific code review
/security-scan          # Dependency audit + unsafe scan

# Verification
/verify                 # Build, clippy, test, security scan
```

## Git 워크플로우

- `feat:` 새 기능, `fix:` 버그 수정, `refactor:` 코드 변경
- `main`에서 기능 브랜치 생성, PR 필수
- CI: `cargo fmt --check`, `cargo clippy`, `cargo test`, `cargo audit`
- 배포: `scratch` 또는 `distroless` 베이스를 사용한 Docker 멀티 스테이지 빌드

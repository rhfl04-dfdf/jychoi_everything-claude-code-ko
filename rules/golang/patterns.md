---
paths:
  - "**/*.go"
  - "**/go.mod"
  - "**/go.sum"
---
# Go 패턴

> 이 파일은 [common/patterns.md](../common/patterns.md)를 확장하여 Go 특화 내용을 다룹니다.

## 함수형 옵션 패턴

```go
type Option func(*Server)

func WithPort(port int) Option {
    return func(s *Server) { s.port = port }
}

func NewServer(opts ...Option) *Server {
    s := &Server{port: 8080}
    for _, opt := range opts {
        opt(s)
    }
    return s
}
```

## 작은 인터페이스

인터페이스는 구현하는 곳이 아닌, 사용하는 곳에서 정의합니다.

## 의존성 주입

생성자 함수를 사용하여 의존성을 주입:

```go
func NewUserService(repo UserRepository, logger Logger) *UserService {
    return &UserService{repo: repo, logger: logger}
}
```

## 참고

동시성, 에러 처리, 패키지 구성 등 포괄적인 Go 패턴은 skill: `golang-patterns`를 참조하세요.

---
paths:
  - "**/*.go"
  - "**/go.mod"
  - "**/go.sum"
---
# Go 보안

> 이 파일은 [common/security.md](../common/security.md)를 확장하여 Go 특화 내용을 다룹니다.

## 시크릿 관리

```go
apiKey := os.Getenv("OPENAI_API_KEY")
if apiKey == "" {
    log.Fatal("OPENAI_API_KEY not configured")
}
```

## 보안 스캐닝

- 정적 보안 분석을 위해 **gosec** 사용:
  ```bash
  gosec ./...
  ```

## Context & 타임아웃

타임아웃 제어를 위해 항상 `context.Context` 사용:

```go
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()
```

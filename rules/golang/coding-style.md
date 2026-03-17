---
paths:
  - "**/*.go"
  - "**/go.mod"
  - "**/go.sum"
---
# Go 코딩 스타일

> 이 파일은 [common/coding-style.md](../common/coding-style.md)를 확장하여 Go 특화 내용을 다룹니다.

## 포맷팅

- **gofmt**와 **goimports**는 필수 — 스타일 논쟁 없음

## 설계 원칙

- 인터페이스를 받고, 구조체를 반환
- 인터페이스는 작게 유지 (메서드 1~3개)

## 에러 처리

항상 컨텍스트와 함께 에러를 래핑:

```go
if err != nil {
    return fmt.Errorf("failed to create user: %w", err)
}
```

## 참고

포괄적인 Go 관용구 및 패턴은 skill: `golang-patterns`를 참조하세요.

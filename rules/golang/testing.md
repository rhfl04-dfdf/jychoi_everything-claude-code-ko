---
paths:
  - "**/*.go"
  - "**/go.mod"
  - "**/go.sum"
---
# Go 테스팅

> 이 파일은 [common/testing.md](../common/testing.md)를 확장하여 Go 특화 내용을 다룹니다.

## 프레임워크

**테이블 기반 테스트**와 함께 표준 `go test`를 사용합니다.

## 레이스 감지

항상 `-race` 플래그와 함께 실행:

```bash
go test -race ./...
```

## 커버리지

```bash
go test -cover ./...
```

## 참고

상세한 Go 테스팅 패턴 및 헬퍼는 skill: `golang-testing`을 참조하세요.

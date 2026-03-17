---
description: Go build 에러, go vet 경고, 린터 이슈를 점진적으로 수정합니다. 최소한의 정밀한 수정을 위해 go-build-resolver 에이전트를 호출합니다.
---

# Go Build and Fix

이 커맨드는 **go-build-resolver** 에이전트를 호출하여 최소한의 변경으로 Go build 에러를 점진적으로 수정합니다.

## 이 커맨드가 하는 것

1. **진단 실행**: `go build`, `go vet`, `staticcheck` 실행
2. **에러 분석**: 파일별로 그룹화하고 심각도순 정렬
3. **점진적 수정**: 한 번에 하나의 에러씩
4. **각 수정 검증**: 각 변경 후 build 재실행
5. **요약 보고**: 수정된 것과 남은 것 표시

## 사용 시점

`/go-build`를 사용해야 할 때:
- `go build ./...`가 에러로 실패할 때
- `go vet ./...`가 이슈를 보고할 때
- `golangci-lint run`이 경고를 보여줄 때
- 모듈 의존성이 깨졌을 때
- 변경사항을 pull한 후 build가 깨졌을 때

## 실행되는 진단 커맨드

```bash
# Primary build check
go build ./...

# Static analysis
go vet ./...

# Extended linting (if available)
staticcheck ./...
golangci-lint run

# Module issues
go mod verify
go mod tidy -v
```

## 예시 세션

```text
User: /go-build

Agent:
# Go Build Resolution

## Initial Diagnostics

```bash
$ go build ./...
internal/service/user.go:25:15: undefined: UserRepository
internal/handler/api.go:42:9: cannot use x (type string) as type int
internal/handler/api.go:58:2: missing return at end of function
```

Errors found: 3

## Fix 1: Undefined Identifier

File: internal/service/user.go:25
Error: undefined: UserRepository
Cause: Missing import

```go
// import 추가
import "project/internal/repository"

// 변경 전
var repo UserRepository
// 변경 후
var repo repository.UserRepository
```

```bash
$ go build ./...
# 2개 에러 남음
```

## Fix 2: Type Mismatch

File: internal/handler/api.go:42
Error: cannot use x (type string) as type int

```go
// 변경 전
count := params.Get("count")
// 변경 후
countStr := params.Get("count")
count, _ := strconv.Atoi(countStr)
```

```bash
$ go build ./...
# 1개 에러 남음
```

## Fix 3: Missing Return

File: internal/handler/api.go:58
Error: missing return at end of function

```go
func GetUser(id string) (*User, error) {
    if id == "" {
        return nil, ErrInvalidID
    }
    user := findUser(id)
    // 누락된 return 추가
    return user, nil
}
```

```bash
$ go build ./...
# Build 성공!
```

## Final Verification

```bash
$ go vet ./...
# 이슈 없음

$ go test ./...
ok      project/internal/service   0.015s
ok      project/internal/handler   0.023s
```

## Summary

| Metric | Count |
|--------|-------|
| Build errors fixed | 3 |
| Vet warnings fixed | 0 |
| Files modified | 2 |
| Remaining issues | 0 |

Build Status: ✅ SUCCESS
````

## 자주 발생하는 에러

| 에러 | 일반적인 수정 방법 |
|------|-------------------|
| `undefined: X` | import 추가 또는 오타 수정 |
| `cannot use X as Y` | 타입 변환 또는 할당 수정 |
| `missing return` | return 문 추가 |
| `X does not implement Y` | 누락된 메서드 추가 |
| `import cycle` | 패키지 구조 재구성 |
| `declared but not used` | 변수 제거 또는 사용 |
| `cannot find package` | `go get` 또는 `go mod tidy` |

## 수정 전략

1. **Build 에러 먼저** - 코드가 컴파일되어야 함
2. **Vet 경고 두 번째** - 의심스러운 구조 수정
3. **Lint 경고 세 번째** - 스타일과 모범 사례
4. **한 번에 하나씩** - 각 변경 검증
5. **최소한의 변경** - 리팩토링이 아닌 수정만

## 중단 조건

에이전트가 중단하고 보고하는 경우:
- 3번 시도 후에도 같은 에러가 지속
- 수정이 더 많은 에러를 발생시킴
- 아키텍처 변경이 필요한 경우
- 외부 의존성이 누락된 경우

## 관련 커맨드

- `/go-test` - build 성공 후 테스트 실행
- `/go-review` - 코드 품질 리뷰
- `/verify` - 전체 검증 루프

## 관련 항목

- 에이전트: `agents/go-build-resolver.md`
- 스킬: `skills/golang-patterns/`

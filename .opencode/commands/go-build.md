---
description: Go 빌드 및 vet 에러 수정
agent: go-build-resolver
subtask: true
---

# Go Build 커맨드

Go 빌드, vet, 컴파일 에러 수정: $ARGUMENTS

## 작업 내용

1. **go build 실행**: `go build ./...`
2. **go vet 실행**: `go vet ./...`
3. **에러 하나씩 수정**
4. **수정 사항이 새로운 에러를 유발하지 않는지 검증**

## 일반적인 Go 에러

### import 에러
```
imported and not used: "package"
```
**수정**: 미사용 import 제거 또는 `_` 접두사 사용

### 타입 에러
```
cannot use x (type T) as type U
```
**수정**: 타입 변환 추가 또는 타입 정의 수정

### undefined 에러
```
undefined: identifier
```
**수정**: 패키지 import, 변수 정의 또는 오타 수정

### vet 에러
```
printf: call has arguments but no formatting directives
```
**수정**: 포맷 지시자 추가 또는 인수 제거

## 수정 순서

1. **import 에러** - import 수정 또는 제거
2. **타입 정의** - 타입 존재 확인
3. **함수 시그니처** - 파라미터 일치
4. **vet 경고** - 정적 분석 해결

## 빌드 커맨드

```bash
# Build all packages
go build ./...

# Build with race detector
go build -race ./...

# Build for specific OS/arch
GOOS=linux GOARCH=amd64 go build ./...

# Run go vet
go vet ./...

# Run staticcheck
staticcheck ./...

# Format code
gofmt -w .

# Tidy dependencies
go mod tidy
```

## 검증

수정 후:
```bash
go build ./...    # Should succeed
go vet ./...      # Should have no warnings
go test ./...     # Tests should pass
```

---

**중요**: 에러만 수정하세요. 리팩토링이나 개선은 하지 마세요. 최소한의 변경으로 빌드를 성공시키세요.

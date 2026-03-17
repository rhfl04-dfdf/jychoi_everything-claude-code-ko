---
description: 테이블 기반 테스트를 활용한 Go TDD 워크플로우
agent: tdd-guide
subtask: true
---

# Go Test 커맨드

Go TDD 방법론을 적용하여 구현: $ARGUMENTS

## 작업 내용

Go 관용구를 활용한 테스트 주도 개발 적용:

1. **타입 정의** - 인터페이스와 struct
2. **테이블 기반 테스트 작성** - 포괄적 커버리지
3. **최소한의 코드 구현** - 테스트 통과
4. **벤치마크** - 성능 검증

## Go TDD 사이클

### 스텝 1: 인터페이스 정의
```go
type Calculator interface {
    Calculate(input Input) (Output, error)
}

type Input struct {
    // fields
}

type Output struct {
    // fields
}
```

### 스텝 2: 테이블 기반 테스트
```go
func TestCalculate(t *testing.T) {
    tests := []struct {
        name    string
        input   Input
        want    Output
        wantErr bool
    }{
        {
            name:  "valid input",
            input: Input{...},
            want:  Output{...},
        },
        {
            name:    "invalid input",
            input:   Input{...},
            wantErr: true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := Calculate(tt.input)
            if (err != nil) != tt.wantErr {
                t.Errorf("Calculate() error = %v, wantErr %v", err, tt.wantErr)
                return
            }
            if !reflect.DeepEqual(got, tt.want) {
                t.Errorf("Calculate() = %v, want %v", got, tt.want)
            }
        })
    }
}
```

### 스텝 3: 테스트 실행 (RED)
```bash
go test -v ./...
```

### 스텝 4: 구현 (GREEN)
```go
func Calculate(input Input) (Output, error) {
    // Minimal implementation
}
```

### 스텝 5: 벤치마크
```go
func BenchmarkCalculate(b *testing.B) {
    input := Input{...}
    for i := 0; i < b.N; i++ {
        Calculate(input)
    }
}
```

## Go 테스트 커맨드

```bash
# Run all tests
go test ./...

# Run with verbose output
go test -v ./...

# Run with coverage
go test -cover ./...

# Run with race detector
go test -race ./...

# Run benchmarks
go test -bench=. ./...

# Generate coverage report
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out
```

## 테스트 파일 구조

```
package/
├── calculator.go       # Implementation
├── calculator_test.go  # Tests
├── testdata/           # Test fixtures
│   └── input.json
└── mock_test.go        # Mock implementations
```

---

**팁**: 더 깔끔한 assertion을 위해 `testify/assert`를 사용하거나, 단순함을 위해 표준 라이브러리를 사용하세요.

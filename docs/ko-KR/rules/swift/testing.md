---
paths:
  - "**/*.swift"
  - "**/Package.swift"
---
# Swift 테스팅

> 이 파일은 [common/testing.md](../common/testing.md)를 확장하여 Swift 특화 내용을 다룹니다.

## 프레임워크

새로운 테스트에는 **Swift Testing** (`import Testing`)을 사용하세요. `@Test`와 `#expect`를 사용합니다:

```swift
 @everything-claude-code/docs/ko-KR/rules/python/testing.md("User creation validates email")
func userCreationValidatesEmail() throws {
    #expect(throws: ValidationError.invalidEmail) {
        try User(email: "not-an-email")
    }
}
```

## 테스트 격리

각 테스트는 새로운 인스턴스를 할당받습니다 — `init`에서 설정(set up)하고 `deinit`에서 해제(tear down)하세요. 테스트 간에 공유되는 가변 상태(shared mutable state)가 없어야 합니다.

## 파라미터화된 테스트

```swift
 @everything-claude-code/docs/ko-KR/rules/python/testing.md("Validates formats", arguments: ["json", "xml", "csv"])
func validatesFormat(format: String) throws {
    let parser = try Parser(format: format)
    #expect(parser.isValid)
}
```

## 커버리지

```bash
swift test --enable-code-coverage
```

## 참고

프로토콜 기반의 dependency injection 및 Swift Testing을 사용한 mock 패턴은 skill: `swift-protocol-di-testing`을 참조하세요.
--- Content from referenced files ---
Content from @everything-claude-code/docs/ko-KR/rules/python/testing.md:
---
paths:
  - "**/*.py"
  - "**/*.pyi"
---
# Python 테스팅

> 이 파일은 [common/testing.md](../common/testing.md)를 확장하여 Python 특화 내용을 다룹니다.

## 프레임워크

테스트 프레임워크로 **pytest** 사용.

## 커버리지

```bash
pytest --cov=src --cov-report=term-missing
```

## 테스트 구성

테스트 분류에 `pytest.mark` 사용:

```python
import pytest

@pytest.mark.unit
def test_calculate_total():
    ...

@pytest.mark.integration
def test_database_connection():
    ...
```

## 참고

상세한 pytest 패턴 및 픽스처는 skill: `python-testing`을 참조하세요.
--- End of content ---

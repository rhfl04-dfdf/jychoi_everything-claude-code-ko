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

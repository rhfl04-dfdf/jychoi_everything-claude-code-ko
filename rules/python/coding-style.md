---
paths:
  - "**/*.py"
  - "**/*.pyi"
---
# Python 코딩 스타일

> 이 파일은 [common/coding-style.md](../common/coding-style.md)를 확장하여 Python 특화 내용을 다룹니다.

## 표준

- **PEP 8** 컨벤션 준수
- 모든 함수 시그니처에 **타입 어노테이션** 사용

## 불변성

불변 데이터 구조 선호:

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class User:
    name: str
    email: str

from typing import NamedTuple

class Point(NamedTuple):
    x: float
    y: float
```

## 포맷팅

- 코드 포맷팅에는 **black**
- import 정렬에는 **isort**
- 린팅에는 **ruff**

## 참고

포괄적인 Python 관용구 및 패턴은 skill: `python-patterns`를 참조하세요.

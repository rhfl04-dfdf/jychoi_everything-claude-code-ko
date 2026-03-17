---
paths:
  - "**/*.py"
  - "**/*.pyi"
---
# Python 패턴

> 이 파일은 [common/patterns.md](../common/patterns.md)를 확장하여 Python 특화 내용을 다룹니다.

## Protocol (덕 타이핑)

```python
from typing import Protocol

class Repository(Protocol):
    def find_by_id(self, id: str) -> dict | None: ...
    def save(self, entity: dict) -> dict: ...
```

## DTO로서의 Dataclass

```python
from dataclasses import dataclass

@dataclass
class CreateUserRequest:
    name: str
    email: str
    age: int | None = None
```

## 컨텍스트 매니저 & 제너레이터

- 리소스 관리에는 컨텍스트 매니저(`with` 구문) 사용
- 지연 평가 및 메모리 효율적인 반복에는 제너레이터 사용

## 참고

데코레이터, 동시성, 패키지 구성 등 포괄적인 패턴은 skill: `python-patterns`를 참조하세요.

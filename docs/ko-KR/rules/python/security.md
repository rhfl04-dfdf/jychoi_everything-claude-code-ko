---
paths:
  - "**/*.py"
  - "**/*.pyi"
---
# Python 보안

> 이 파일은 [common/security.md](../common/security.md)를 확장하여 Python 특화 내용을 다룹니다.

## 시크릿 관리

```python
import os
from dotenv import load_dotenv

load_dotenv()

api_key = os.environ["OPENAI_API_KEY"]  # 없으면 KeyError 발생
```

## 보안 스캐닝

- 정적 보안 분석을 위해 **bandit** 사용:
  ```bash
  bandit -r src/
  ```

## 참고

Django 특화 보안 지침은 skill: `django-security`를 참조하세요 (해당하는 경우).

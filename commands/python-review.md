---
description: PEP 8 준수, type hints, 보안 및 Pythonic 관용구에 대한 포괄적인 Python code review입니다. python-reviewer agent를 호출합니다.
---

# Python Code Review

이 command는 포괄적인 Python 전용 code review를 위해 **python-reviewer** agent를 호출합니다.

## 이 Command의 역할

1. **Python 변경 사항 식별**: `git diff`를 통해 수정된 `.py` 파일 찾기
2. **정적 분석 실행**: `ruff`, `mypy`, `pylint`, `black --check` 실행
3. **보안 스캔**: SQL injection, command injection, 안전하지 않은 역직렬화 확인
4. **Type Safety 검토**: type hints 및 mypy 에러 분석
5. **Pythonic Code 체크**: 코드가 PEP 8 및 Python best practices를 따르는지 확인
6. **보고서 생성**: 심각도별로 이슈 분류

## 사용 시기

다음과 같은 경우 `/python-review`를 사용하세요:
- Python 코드를 작성하거나 수정한 후
- Python 변경 사항을 커밋하기 전
- Python 코드가 포함된 pull request를 검토할 때
- 새로운 Python codebase에 온보딩할 때
- Pythonic 패턴과 관용구를 배울 때

## 검토 카테고리

### CRITICAL (필수 수정)
- SQL/Command injection 취약점
- 안전하지 않은 eval/exec 사용
- Pickle의 안전하지 않은 역직렬화
- Hardcoded credentials (코드 내 자격 증명 포함)
- YAML의 안전하지 않은 로드
- 에러를 숨기는 빈 except 절 (Bare except clauses)

### HIGH (수정 권장)
- 공개 함수(public functions)의 누락된 type hints
- 가변 기본 인자 (Mutable default arguments)
- 예외를 소리 없이 무시하는 경우
- 리소스에 context managers를 사용하지 않는 경우
- comprehensions 대신 C-style 루프 사용
- isinstance() 대신 type() 사용
- lock이 없는 경쟁 상태 (Race conditions)

### MEDIUM (고려 사항)
- PEP 8 포맷팅 위반
- 공개 함수의 누락된 docstrings
- logging 대신 print 문 사용
- 비효율적인 문자열 연산
- 이름이 지정된 상수(named constants)를 사용하지 않는 Magic numbers
- 포맷팅에 f-strings를 사용하지 않는 경우
- 불필요한 리스트 생성

## 실행되는 자동 체크

```bash
# Type checking
mypy .

# Linting and formatting
ruff check .
black --check .
isort --check-only .

# Security scanning
bandit -r .

# Dependency audit
pip-audit
safety check

# Testing
pytest --cov=app --cov-report=term-missing
```

## 사용 예시

```text
User: /python-review

Agent:
# Python Code Review Report

## Files Reviewed
- app/routes/user.py (modified)
- app/services/auth.py (modified)

## Static Analysis Results
✓ ruff: No issues
✓ mypy: No errors
⚠️ black: 2 files need reformatting
✓ bandit: No security issues

## Issues Found

[CRITICAL] SQL Injection vulnerability
File: app/routes/user.py:42
Issue: User input directly interpolated into SQL query
```python
query = f"SELECT * FROM users WHERE id = {user_id}"  # Bad
```
Fix: Use parameterized query
```python
query = "SELECT * FROM users WHERE id = %s"  # Good
cursor.execute(query, (user_id,))
```

[HIGH] Mutable default argument
File: app/services/auth.py:18
Issue: Mutable default argument causes shared state
```python
def process_items(items=[]):  # Bad
    items.append("new")
    return items
```
Fix: Use None as default
```python
def process_items(items=None):  # Good
    if items is None:
        items = []
    items.append("new")
    return items
```

[MEDIUM] Missing type hints
File: app/services/auth.py:25
Issue: Public function without type annotations
```python
def get_user(user_id):  # Bad
    return db.find(user_id)
```
Fix: Add type hints
```python
def get_user(user_id: str) -> Optional[User]:  # Good
    return db.find(user_id)
```

[MEDIUM] Not using context manager
File: app/routes/user.py:55
Issue: File not closed on exception
```python
f = open("config.json")  # Bad
data = f.read()
f.close()
```
Fix: Use context manager
```python
with open("config.json") as f:  # Good
    data = f.read()
```

## Summary
- CRITICAL: 1
- HIGH: 1
- MEDIUM: 2

Recommendation: ❌ Block merge until CRITICAL issue is fixed

## Formatting Required
Run: `black app/routes/user.py app/services/auth.py`
```

## 승인 기준

| 상태 | 조건 |
|--------|-----------|
| ✅ 승인 | CRITICAL 또는 HIGH 이슈 없음 |
| ⚠️ 경고 | MEDIUM 이슈만 있음 (주의하며 병합) |
| ❌ 차단 | CRITICAL 또는 HIGH 이슈 발견됨 |

## 다른 Command와의 연동

- 테스트 통과 확인을 위해 `/tdd`를 먼저 사용하세요
- Python 이외의 일반적인 사항은 `/code-review`를 사용하세요
- 커밋하기 전에 `/python-review`를 사용하세요
- 정적 분석 도구가 실패하면 `/build-fix`를 사용하세요

## 프레임워크별 검토

### Django 프로젝트
검토자는 다음 사항을 확인합니다:
- N+1 쿼리 이슈 (`select_related` 및 `prefetch_related` 사용 여부)
- 모델 변경에 따른 누락된 migrations
- ORM으로 해결 가능한 경우에도 Raw SQL 사용 여부
- 다단계 작업을 위한 `transaction.atomic()` 누락 여부

### FastAPI 프로젝트
검토자는 다음 사항을 확인합니다:
- CORS 설정 오류
- 요청 검증을 위한 Pydantic 모델 사용 여부
- 응답 모델의 정확성
- 적절한 async/await 사용
- Dependency injection 패턴

### Flask 프로젝트
검토자는 다음 사항을 확인합니다:
- Context 관리 (app context, request context)
- 적절한 에러 핸들링
- Blueprint 구성
- 설정 관리 (Configuration management)

## 관련 항목

- Agent: `agents/python-reviewer.md`
- Skills: `skills/python-patterns/`, `skills/python-testing/`

## 일반적인 수정 사항

### Type Hints 추가
```python
# Before
def calculate(x, y):
    return x + y

# After
from typing import Union

def calculate(x: Union[int, float], y: Union[int, float]) -> Union[int, float]:
    return x + y
```

### Context Managers 사용
```python
# Before
f = open("file.txt")
data = f.read()
f.close()

# After
with open("file.txt") as f:
    data = f.read()
```

### List Comprehensions 사용
```python
# Before
result = []
for item in items:
    if item.active:
        result.append(item.name)

# After
result = [item.name for item in items if item.active]
```

### 가변 기본값 수정
```python
# Before
def append(value, items=[]):
    items.append(value)
    return items

# After
def append(value, items=None):
    if items is None:
        items = []
    items.append(value)
    return items
```

### f-strings 사용 (Python 3.6+)
```python
# Before
name = "Alice"
greeting = "Hello, " + name + "!"
greeting2 = "Hello, {}".format(name)

# After
greeting = f"Hello, {name}!"
```

### 루프 내 문자열 연결 수정
```python
# Before
result = ""
for item in items:
    result += str(item)

# After
result = "".join(str(item) for item in items)
```

## Python 버전 호환성

검토자는 코드가 최신 Python 버전의 기능을 사용하는지 확인합니다:

| 기능 | 최소 Python 버전 |
|---------|----------------|
| Type hints | 3.5+ |
| f-strings | 3.6+ |
| Walrus operator (`:=`) | 3.8+ |
| 위치 전용 파라미터 (Position-only parameters) | 3.8+ |
| Match 문 | 3.10+ |
| 타입 유니온 (&#96;x &#124; None&#96;) | 3.10+ |

프로젝트의 `pyproject.toml` 또는 `setup.py`에 올바른 최소 Python 버전이 명시되어 있는지 확인하세요.

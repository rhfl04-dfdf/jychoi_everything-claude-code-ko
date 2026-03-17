---
name: django-verification
description: "Django 프로젝트를 위한 검증 루프: 배포 또는 PR 전 migration, linting, 커버리지가 포함된 테스트, 보안 스캔 및 배포 준비 상태 체크."
origin: ECC
---

# Django Verification Loop

PR 전, 주요 변경 사항 이후, 그리고 배포 전에 실행하여 Django 애플리케이션의 품질과 보안을 보장합니다.

## 활성화 시점

- Django 프로젝트에 대한 pull request를 열기 전
- 주요 모델 변경, migration 업데이트 또는 종속성 업그레이드 후
- 스테이징 또는 프로덕션에 대한 배포 전 검증
- 전체 환경 → lint → test → 보안 → 배포 준비 파이프라인 실행
- migration 안정성 및 테스트 커버리지 검증

## 1단계: 환경 체크

```bash
# Verify Python version
python --version  # Should match project requirements

# Check virtual environment
which python
pip list --outdated

# Verify environment variables
python -c "import os; import environ; print('DJANGO_SECRET_KEY set' if os.environ.get('DJANGO_SECRET_KEY') else 'MISSING: DJANGO_SECRET_KEY')"
```

환경이 잘못 구성된 경우 중단하고 수정하세요.

## 2단계: 코드 품질 및 포매팅

```bash
# Type checking
mypy . --config-file pyproject.toml

# Linting with ruff
ruff check . --fix

# Formatting with black
black . --check
black .  # Auto-fix

# Import sorting
isort . --check-only
isort .  # Auto-fix

# Django-specific checks
python manage.py check --deploy
```

일반적인 문제:
- 공용 함수의 타입 힌트 누락
- PEP 8 포매팅 위반
- 정렬되지 않은 임포트
- 프로덕션 설정에 남겨진 디버그 설정

## 3단계: Migration

```bash
# Check for unapplied migrations
python manage.py showmigrations

# Create missing migrations
python manage.py makemigrations --check

# Dry-run migration application
python manage.py migrate --plan

# Apply migrations (test environment)
python manage.py migrate

# Check for migration conflicts
python manage.py makemigrations --merge  # Only if conflicts exist
```

리포트:
- 보류 중인 migration 수
- migration 충돌 여부
- migration이 없는 모델 변경 사항

## 4단계: 테스트 + 커버리지

```bash
# Run all tests with pytest
pytest --cov=apps --cov-report=html --cov-report=term-missing --reuse-db

# Run specific app tests
pytest apps/users/tests/

# Run with markers
pytest -m "not slow"  # Skip slow tests
pytest -m integration  # Only integration tests

# Coverage report
open htmlcov/index.html
```

리포트:
- 총 테스트: X 통과, Y 실패, Z 건너뜀
- 전체 커버리지: XX%
- 앱별 커버리지 세부 사항

커버리지 목표:

| 컴포넌트 | 목표 |
|-----------|--------|
| Models | 90%+ |
| Serializers | 85%+ |
| Views | 80%+ |
| Services | 90%+ |
| Overall | 80%+ |

## 5단계: 보안 스캔

```bash
# Dependency vulnerabilities
pip-audit
safety check --full-report

# Django security checks
python manage.py check --deploy

# Bandit security linter
bandit -r . -f json -o bandit-report.json

# Secret scanning (if gitleaks is installed)
gitleaks detect --source . --verbose

# Environment variable check
python -c "from django.core.exceptions import ImproperlyConfigured; from django.conf import settings; settings.DEBUG"
```

리포트:
- 취약한 종속성 발견됨
- 보안 설정 문제
- 하드코딩된 secret 감지됨
- DEBUG 모드 상태 (프로덕션에서는 False여야 함)

## 6단계: Django 관리 명령

```bash
# Check for model issues
python manage.py check

# Collect static files
python manage.py collectstatic --noinput --clear

# Create superuser (if needed for tests)
echo "from apps.users.models import User; User.objects.create_superuser('admin@example.com', 'admin')" | python manage.py shell

# Database integrity
python manage.py check --database default

# Cache verification (if using Redis)
python -c "from django.core.cache import cache; cache.set('test', 'value', 10); print(cache.get('test'))"
```

## 7단계: 성능 체크

```bash
# Django Debug Toolbar output (check for N+1 queries)
# Run in dev mode with DEBUG=True and access a page
# Look for duplicate queries in SQL panel

# Query count analysis
django-admin debugsqlshell  # If django-debug-sqlshell installed

# Check for missing indexes
python manage.py shell << EOF
from django.db import connection
with connection.cursor() as cursor:
    cursor.execute("SELECT table_name, index_name FROM information_schema.statistics WHERE table_schema = 'public'")
    print(cursor.fetchall())
EOF
```

리포트:
- 페이지당 쿼리 수 (일반적인 페이지의 경우 50개 미만이어야 함)
- 누락된 데이터베이스 인덱스
- 중복 쿼리 감지됨

## 8단계: 정적 에셋

```bash
# Check for npm dependencies (if using npm)
npm audit
npm audit fix

# Build static files (if using webpack/vite)
npm run build

# Verify static files
ls -la staticfiles/
python manage.py findstatic css/style.css
```

## 9단계: 설정 리뷰

```python
# Run in Python shell to verify settings
python manage.py shell << EOF
from django.conf import settings
import os

# Critical checks
checks = {
    'DEBUG is False': not settings.DEBUG,
    'SECRET_KEY set': bool(settings.SECRET_KEY and len(settings.SECRET_KEY) > 30),
    'ALLOWED_HOSTS set': len(settings.ALLOWED_HOSTS) > 0,
    'HTTPS enabled': getattr(settings, 'SECURE_SSL_REDIRECT', False),
    'HSTS enabled': getattr(settings, 'SECURE_HSTS_SECONDS', 0) > 0,
    'Database configured': settings.DATABASES['default']['ENGINE'] != 'django.db.backends.sqlite3',
}

for check, result in checks.items():
    status = '✓' if result else '✗'
    print(f"{status} {check}")
EOF
```

## 10단계: 로깅 설정

```bash
# Test logging output
python manage.py shell << EOF
import logging
logger = logging.getLogger('django')
logger.warning('Test warning message')
logger.error('Test error message')
EOF

# Check log files (if configured)
tail -f /var/log/django/django.log
```

## 11단계: API 문서 (DRF인 경우)

```bash
# Generate schema
python manage.py generateschema --format openapi-json > schema.json

# Validate schema
# Check if schema.json is valid JSON
python -c "import json; json.load(open('schema.json'))"

# Access Swagger UI (if using drf-yasg)
# Visit http://localhost:8000/swagger/ in browser
```

## 12단계: Diff 리뷰

```bash
# Show diff statistics
git diff --stat

# Show actual changes
git diff

# Show changed files
git diff --name-only

# Check for common issues
git diff | grep -i "todo\|fixme\|hack\|xxx"
git diff | grep "print("  # Debug statements
git diff | grep "DEBUG = True"  # Debug mode
git diff | grep "import pdb"  # Debugger
```

체크리스트:
- 디버깅 구문 없음 (print, pdb, breakpoint())
- 중요한 코드에 TODO/FIXME 주석 없음
- 하드코딩된 secret 또는 자격 증명 없음
- 모델 변경에 대한 데이터베이스 migration 포함됨
- 설정 변경 사항 문서화됨
- 외부 호출에 대한 에러 핸들링 존재
- 필요한 경우 트랜잭션 관리 적용

## 출력 템플릿

```
DJANGO VERIFICATION REPORT
==========================

Phase 1: Environment Check
  ✓ Python 3.11.5
  ✓ Virtual environment active
  ✓ All environment variables set

Phase 2: Code Quality
  ✓ mypy: No type errors
  ✗ ruff: 3 issues found (auto-fixed)
  ✓ black: No formatting issues
  ✓ isort: Imports properly sorted
  ✓ manage.py check: No issues

Phase 3: Migrations
  ✓ No unapplied migrations
  ✓ No migration conflicts
  ✓ All models have migrations

Phase 4: Tests + Coverage
  Tests: 247 passed, 0 failed, 5 skipped
  Coverage:
    Overall: 87%
    users: 92%
    products: 89%
    orders: 85%
    payments: 91%

Phase 5: Security Scan
  ✗ pip-audit: 2 vulnerabilities found (fix required)
  ✓ safety check: No issues
  ✓ bandit: No security issues
  ✓ No secrets detected
  ✓ DEBUG = False

Phase 6: Django Commands
  ✓ collectstatic completed
  ✓ Database integrity OK
  ✓ Cache backend reachable

Phase 7: Performance
  ✓ No N+1 queries detected
  ✓ Database indexes configured
  ✓ Query count acceptable

Phase 8: Static Assets
  ✓ npm audit: No vulnerabilities
  ✓ Assets built successfully
  ✓ Static files collected

Phase 9: Configuration
  ✓ DEBUG = False
  ✓ SECRET_KEY configured
  ✓ ALLOWED_HOSTS set
  ✓ HTTPS enabled
  ✓ HSTS enabled
  ✓ Database configured

Phase 10: Logging
  ✓ Logging configured
  ✓ Log files writable

Phase 11: API Documentation
  ✓ Schema generated
  ✓ Swagger UI accessible

Phase 12: Diff Review
  Files changed: 12
  +450, -120 lines
  ✓ No debug statements
  ✓ No hardcoded secrets
  ✓ Migrations included

RECOMMENDATION: ⚠️ Fix pip-audit vulnerabilities before deploying

NEXT STEPS:
1. Update vulnerable dependencies
2. Re-run security scan
3. Deploy to staging for final testing
```

## 배포 전 체크리스트

- [ ] 모든 테스트 통과
- [ ] 커버리지 ≥ 80%
- [ ] 보안 취약점 없음
- [ ] 적용되지 않은 migration 없음
- [ ] 프로덕션 설정에서 DEBUG = False
- [ ] SECRET_KEY가 적절히 구성됨
- [ ] ALLOWED_HOSTS가 올바르게 설정됨
- [ ] 데이터베이스 백업 활성화됨
- [ ] 정적 파일이 수집되고 제공됨
- [ ] 로깅이 구성되고 작동함
- [ ] 에러 모니터링(Sentry 등)이 구성됨
- [ ] CDN 구성됨 (해당하는 경우)
- [ ] Redis/캐시 백엔드 구성됨
- [ ] Celery 워커 실행 중 (해당하는 경우)
- [ ] HTTPS/SSL 구성됨
- [ ] 환경 변수 문서화됨

## 지속적 통합 (CI)

### GitHub Actions 예시

```yaml
# .github/workflows/django-verification.yml
name: Django Verification

on: [push, pull_request]

jobs:
  verify:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Cache pip
        uses: actions/cache@v3
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}

      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install ruff black mypy pytest pytest-django pytest-cov bandit safety pip-audit

      - name: Code quality checks
        run: |
          ruff check .
          black . --check
          isort . --check-only
          mypy .

      - name: Security scan
        run: |
          bandit -r . -f json -o bandit-report.json
          safety check --full-report
          pip-audit

      - name: Run tests
        env:
          DATABASE_URL: postgres://postgres:postgres@localhost:5432/test
          DJANGO_SECRET_KEY: test-secret-key
        run: |
          pytest --cov=apps --cov-report=xml --cov-report=term-missing

      - name: Upload coverage
        uses: codecov/codecov-action@v3
```

## 빠른 참조

| 체크 항목 | 명령 |
|-------|---------|
| 환경 | `python --version` |
| 타입 체크 | `mypy .` |
| Linting | `ruff check .` |
| 포매팅 | `black . --check` |
| Migration | `python manage.py makemigrations --check` |
| 테스트 | `pytest --cov=apps` |
| 보안 | `pip-audit && bandit -r .` |
| Django 체크 | `python manage.py check --deploy` |
| Collectstatic | `python manage.py collectstatic --noinput` |
| Diff 통계 | `git diff --stat` |

주의: 자동화된 검증은 일반적인 문제를 잡아내지만, 수동 코드 리뷰 및 스테이징 환경에서의 테스트를 대체하지는 않습니다.

# Django REST API — 프로젝트 CLAUDE.md

> PostgreSQL과 Celery를 사용하는 Django REST Framework API의 실제 예제입니다.
> 프로젝트 루트에 복사하여 서비스에 맞게 커스터마이징하세요.

## 프로젝트 개요

**스택:** Python 3.12+, Django 5.x, Django REST Framework, PostgreSQL, Celery + Redis, pytest, Docker Compose

**아키텍처:** 비즈니스 도메인별 앱을 사용하는 도메인 주도 설계. API 레이어에는 DRF, 비동기 작업에는 Celery, 테스트에는 pytest 사용. 모든 엔드포인트는 JSON 반환 — 템플릿 렌더링 없음.

## 핵심 규칙

### Python 컨벤션

- 모든 함수 시그니처에 타입 힌트 — `from __future__ import annotations` 사용
- `print()` 문 사용 금지 — `logging.getLogger(__name__)` 사용
- 문자열 포맷팅에 f-string 사용, `%` 또는 `.format()` 사용 금지
- 파일 작업에는 `os.path` 대신 `pathlib.Path` 사용
- isort로 임포트 정렬: 표준 라이브러리, 서드파티, 로컬 순서 (ruff로 강제)

### 데이터베이스

- 모든 쿼리는 Django ORM 사용 — 원시 SQL은 `.raw()`와 매개변수화된 쿼리로만 사용
- 마이그레이션은 git에 커밋 — 프로덕션에서 `--fake` 사용 금지
- N+1 쿼리 방지를 위해 `select_related()`와 `prefetch_related()` 사용
- 모든 모델에는 `created_at`과 `updated_at` 자동 필드가 있어야 함
- `filter()`, `order_by()`, `WHERE` 절에 사용되는 필드에 인덱스 설정

```python
# BAD: N+1 query
orders = Order.objects.all()
for order in orders:
    print(order.customer.name)  # hits DB for each order

# GOOD: Single query with join
orders = Order.objects.select_related("customer").all()
```

### 인증

- `djangorestframework-simplejwt`를 통한 JWT — 액세스 토큰(15분) + 리프레시 토큰(7일)
- 모든 뷰에 권한 클래스 설정 — 기본값에 의존 금지
- 기본으로 `IsAuthenticated` 사용, 객체 수준 접근에는 커스텀 권한 추가
- 로그아웃을 위한 토큰 블랙리스트 활성화

### Serializer

- 단순 CRUD에는 `ModelSerializer`, 복잡한 검증에는 `Serializer` 사용
- 입출력 형태가 다를 경우 읽기/쓰기 serializer 분리
- 뷰가 아닌 serializer 레벨에서 검증 — 뷰는 얇게 유지

```python
class CreateOrderSerializer(serializers.Serializer):
    product_id = serializers.UUIDField()
    quantity = serializers.IntegerField(min_value=1, max_value=100)

    def validate_product_id(self, value):
        if not Product.objects.filter(id=value, active=True).exists():
            raise serializers.ValidationError("Product not found or inactive")
        return value

class OrderDetailSerializer(serializers.ModelSerializer):
    customer = CustomerSerializer(read_only=True)
    product = ProductSerializer(read_only=True)

    class Meta:
        model = Order
        fields = ["id", "customer", "product", "quantity", "total", "status", "created_at"]
```

### 에러 처리

- 일관된 에러 응답을 위해 DRF 예외 핸들러 사용
- 비즈니스 로직의 커스텀 예외는 `core/exceptions.py`에 정의
- 내부 에러 세부 정보를 클라이언트에 노출하지 않음

```python
# core/exceptions.py
from rest_framework.exceptions import APIException

class InsufficientStockError(APIException):
    status_code = 409
    default_detail = "Insufficient stock for this order"
    default_code = "insufficient_stock"
```

### 코드 스타일

- 코드나 주석에 이모지 사용 금지
- 최대 줄 길이: 120자 (ruff로 강제)
- 클래스: PascalCase, 함수/변수: snake_case, 상수: UPPER_SNAKE_CASE
- 뷰는 얇게 — 비즈니스 로직은 서비스 함수나 모델 메서드에 위치

## 파일 구조

```
config/
  settings/
    base.py              # Shared settings
    local.py             # Dev overrides (DEBUG=True)
    production.py        # Production settings
  urls.py                # Root URL config
  celery.py              # Celery app configuration
apps/
  accounts/              # User auth, registration, profile
    models.py
    serializers.py
    views.py
    services.py          # Business logic
    tests/
      test_views.py
      test_services.py
      factories.py       # Factory Boy factories
  orders/                # Order management
    models.py
    serializers.py
    views.py
    services.py
    tasks.py             # Celery tasks
    tests/
  products/              # Product catalog
    models.py
    serializers.py
    views.py
    tests/
core/
  exceptions.py          # Custom API exceptions
  permissions.py         # Shared permission classes
  pagination.py          # Custom pagination
  middleware.py          # Request logging, timing
  tests/
```

## 주요 패턴

### 서비스 레이어

```python
# apps/orders/services.py
from django.db import transaction

def create_order(*, customer, product_id: uuid.UUID, quantity: int) -> Order:
    """Create an order with stock validation and payment hold."""
    product = Product.objects.select_for_update().get(id=product_id)

    if product.stock < quantity:
        raise InsufficientStockError()

    with transaction.atomic():
        order = Order.objects.create(
            customer=customer,
            product=product,
            quantity=quantity,
            total=product.price * quantity,
        )
        product.stock -= quantity
        product.save(update_fields=["stock", "updated_at"])

    # Async: send confirmation email
    send_order_confirmation.delay(order.id)
    return order
```

### 뷰 패턴

```python
# apps/orders/views.py
class OrderViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated]
    pagination_class = StandardPagination

    def get_serializer_class(self):
        if self.action == "create":
            return CreateOrderSerializer
        return OrderDetailSerializer

    def get_queryset(self):
        return (
            Order.objects
            .filter(customer=self.request.user)
            .select_related("product", "customer")
            .order_by("-created_at")
        )

    def perform_create(self, serializer):
        order = create_order(
            customer=self.request.user,
            product_id=serializer.validated_data["product_id"],
            quantity=serializer.validated_data["quantity"],
        )
        serializer.instance = order
```

### 테스트 패턴 (pytest + Factory Boy)

```python
# apps/orders/tests/factories.py
import factory
from apps.accounts.tests.factories import UserFactory
from apps.products.tests.factories import ProductFactory

class OrderFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = "orders.Order"

    customer = factory.SubFactory(UserFactory)
    product = factory.SubFactory(ProductFactory, stock=100)
    quantity = 1
    total = factory.LazyAttribute(lambda o: o.product.price * o.quantity)

# apps/orders/tests/test_views.py
import pytest
from rest_framework.test import APIClient

@pytest.mark.django_db
class TestCreateOrder:
    def setup_method(self):
        self.client = APIClient()
        self.user = UserFactory()
        self.client.force_authenticate(self.user)

    def test_create_order_success(self):
        product = ProductFactory(price=29_99, stock=10)
        response = self.client.post("/api/orders/", {
            "product_id": str(product.id),
            "quantity": 2,
        })
        assert response.status_code == 201
        assert response.data["total"] == 59_98

    def test_create_order_insufficient_stock(self):
        product = ProductFactory(stock=0)
        response = self.client.post("/api/orders/", {
            "product_id": str(product.id),
            "quantity": 1,
        })
        assert response.status_code == 409

    def test_create_order_unauthenticated(self):
        self.client.force_authenticate(None)
        response = self.client.post("/api/orders/", {})
        assert response.status_code == 401
```

## 환경 변수

```bash
# Django
SECRET_KEY=
DEBUG=False
ALLOWED_HOSTS=api.example.com

# Database
DATABASE_URL=postgres://user:pass@localhost:5432/myapp

# Redis (Celery broker + cache)
REDIS_URL=redis://localhost:6379/0

# JWT
JWT_ACCESS_TOKEN_LIFETIME=15       # minutes
JWT_REFRESH_TOKEN_LIFETIME=10080   # minutes (7 days)

# Email
EMAIL_BACKEND=django.core.mail.backends.smtp.EmailBackend
EMAIL_HOST=smtp.example.com
```

## 테스트 전략

```bash
# Run all tests
pytest --cov=apps --cov-report=term-missing

# Run specific app tests
pytest apps/orders/tests/ -v

# Run with parallel execution
pytest -n auto

# Only failing tests from last run
pytest --lf
```

## ECC 워크플로우

```bash
# Planning
/plan "Add order refund system with Stripe integration"

# Development with TDD
/tdd                    # pytest-based TDD workflow

# Review
/python-review          # Python-specific code review
/security-scan          # Django security audit
/code-review            # General quality check

# Verification
/verify                 # Build, lint, test, security scan
```

## Git 워크플로우

- `feat:` 새 기능, `fix:` 버그 수정, `refactor:` 코드 변경
- `main`에서 기능 브랜치 생성, PR 필수
- CI: ruff (린트 + 포맷), mypy (타입), pytest (테스트), safety (의존성 검사)
- 배포: Docker 이미지, Kubernetes 또는 Railway로 관리

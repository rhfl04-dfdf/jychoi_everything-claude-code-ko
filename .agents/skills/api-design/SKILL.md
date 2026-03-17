---
name: api-design
description: REST API 설계 패턴 — 리소스 네이밍, 상태 코드, 페이지네이션, 필터링, 에러 응답, 버전 관리, 요청 제한을 포함한 프로덕션 API를 위한 패턴.
origin: ECC
---

# API 설계 패턴

일관되고 개발자 친화적인 REST API를 설계하기 위한 규칙 및 모범 사례.

## 활성화 시점

- 새 API 엔드포인트 설계
- 기존 API 계약 검토
- 페이지네이션, 필터링, 또는 정렬 추가
- API 에러 처리 구현
- API 버전 관리 전략 계획
- 공개 또는 파트너 대상 API 구축

## 리소스 설계

### URL 구조

```
# Resources are nouns, plural, lowercase, kebab-case
GET    /api/v1/users
GET    /api/v1/users/:id
POST   /api/v1/users
PUT    /api/v1/users/:id
PATCH  /api/v1/users/:id
DELETE /api/v1/users/:id

# Sub-resources for relationships
GET    /api/v1/users/:id/orders
POST   /api/v1/users/:id/orders

# Actions that don't map to CRUD (use verbs sparingly)
POST   /api/v1/orders/:id/cancel
POST   /api/v1/auth/login
POST   /api/v1/auth/refresh
```

### 네이밍 규칙

```
# GOOD
/api/v1/team-members          # kebab-case for multi-word resources
/api/v1/orders?status=active  # query params for filtering
/api/v1/users/123/orders      # nested resources for ownership

# BAD
/api/v1/getUsers              # verb in URL
/api/v1/user                  # singular (use plural)
/api/v1/team_members          # snake_case in URLs
/api/v1/users/123/getOrders   # verb in nested resource
```

## HTTP 메서드 및 상태 코드

### 메서드 의미론

| 메서드 | 멱등성 | 안전성 | 사용 용도 |
|--------|-------|------|---------|
| GET | 예 | 예 | 리소스 조회 |
| POST | 아니오 | 아니오 | 리소스 생성, 액션 트리거 |
| PUT | 예 | 아니오 | 리소스 전체 교체 |
| PATCH | 아니오* | 아니오 | 리소스 부분 업데이트 |
| DELETE | 예 | 아니오 | 리소스 삭제 |

*PATCH는 적절한 구현으로 멱등성을 가질 수 있음

### 상태 코드 참고

```
# Success
200 OK                    — GET, PUT, PATCH (with response body)
201 Created               — POST (include Location header)
204 No Content            — DELETE, PUT (no response body)

# Client Errors
400 Bad Request           — Validation failure, malformed JSON
401 Unauthorized          — Missing or invalid authentication
403 Forbidden             — Authenticated but not authorized
404 Not Found             — Resource doesn't exist
409 Conflict              — Duplicate entry, state conflict
422 Unprocessable Entity  — Semantically invalid (valid JSON, bad data)
429 Too Many Requests     — Rate limit exceeded

# Server Errors
500 Internal Server Error — Unexpected failure (never expose details)
502 Bad Gateway           — Upstream service failed
503 Service Unavailable   — Temporary overload, include Retry-After
```

### 흔한 실수

```
# BAD: 200 for everything
{ "status": 200, "success": false, "error": "Not found" }

# GOOD: Use HTTP status codes semantically
HTTP/1.1 404 Not Found
{ "error": { "code": "not_found", "message": "User not found" } }

# BAD: 500 for validation errors
# GOOD: 400 or 422 with field-level details

# BAD: 200 for created resources
# GOOD: 201 with Location header
HTTP/1.1 201 Created
Location: /api/v1/users/abc-123
```

## 응답 형식

### 성공 응답

```json
{
  "data": {
    "id": "abc-123",
    "email": "alice@example.com",
    "name": "Alice",
    "created_at": "2025-01-15T10:30:00Z"
  }
}
```

### 컬렉션 응답 (페이지네이션 포함)

```json
{
  "data": [
    { "id": "abc-123", "name": "Alice" },
    { "id": "def-456", "name": "Bob" }
  ],
  "meta": {
    "total": 142,
    "page": 1,
    "per_page": 20,
    "total_pages": 8
  },
  "links": {
    "self": "/api/v1/users?page=1&per_page=20",
    "next": "/api/v1/users?page=2&per_page=20",
    "last": "/api/v1/users?page=8&per_page=20"
  }
}
```

### 에러 응답

```json
{
  "error": {
    "code": "validation_error",
    "message": "Request validation failed",
    "details": [
      {
        "field": "email",
        "message": "Must be a valid email address",
        "code": "invalid_format"
      },
      {
        "field": "age",
        "message": "Must be between 0 and 150",
        "code": "out_of_range"
      }
    ]
  }
}
```

### 응답 Envelope 변형

```typescript
// Option A: Envelope with data wrapper (recommended for public APIs)
interface ApiResponse<T> {
  data: T;
  meta?: PaginationMeta;
  links?: PaginationLinks;
}

interface ApiError {
  error: {
    code: string;
    message: string;
    details?: FieldError[];
  };
}

// Option B: Flat response (simpler, common for internal APIs)
// Success: just return the resource directly
// Error: return error object
// Distinguish by HTTP status code
```

## 페이지네이션

### 오프셋 기반 (간단)

```
GET /api/v1/users?page=2&per_page=20

# Implementation
SELECT * FROM users
ORDER BY created_at DESC
LIMIT 20 OFFSET 20;
```

**장점:** 구현이 쉽고, "N 페이지로 이동" 지원
**단점:** 큰 오프셋에서 느림 (OFFSET 100000), 동시 삽입 시 일관성 없음

### 커서 기반 (확장 가능)

```
GET /api/v1/users?cursor=eyJpZCI6MTIzfQ&limit=20

# Implementation
SELECT * FROM users
WHERE id > :cursor_id
ORDER BY id ASC
LIMIT 21;  -- fetch one extra to determine has_next
```

```json
{
  "data": [...],
  "meta": {
    "has_next": true,
    "next_cursor": "eyJpZCI6MTQzfQ"
  }
}
```

**장점:** 위치에 관계없이 일관된 성능, 동시 삽입 시 안정적
**단점:** 임의 페이지로 이동 불가, 커서가 불투명함

### 어떤 것을 사용할지

| 사용 사례 | 페이지네이션 타입 |
|----------|----------------|
| 관리자 대시보드, 소규모 데이터셋 (<10K) | 오프셋 |
| 무한 스크롤, 피드, 대규모 데이터셋 | 커서 |
| 공개 API | 커서 (기본) + 오프셋 (선택) |
| 검색 결과 | 오프셋 (사용자가 페이지 번호 기대) |

## 필터링, 정렬, 검색

### 필터링

```
# Simple equality
GET /api/v1/orders?status=active&customer_id=abc-123

# Comparison operators (use bracket notation)
GET /api/v1/products?price[gte]=10&price[lte]=100
GET /api/v1/orders?created_at[after]=2025-01-01

# Multiple values (comma-separated)
GET /api/v1/products?category=electronics,clothing

# Nested fields (dot notation)
GET /api/v1/orders?customer.country=US
```

### 정렬

```
# Single field (prefix - for descending)
GET /api/v1/products?sort=-created_at

# Multiple fields (comma-separated)
GET /api/v1/products?sort=-featured,price,-created_at
```

### 전체 텍스트 검색

```
# Search query parameter
GET /api/v1/products?q=wireless+headphones

# Field-specific search
GET /api/v1/users?email=alice
```

### Sparse Fieldset

```
# Return only specified fields (reduces payload)
GET /api/v1/users?fields=id,name,email
GET /api/v1/orders?fields=id,total,status&include=customer.name
```

## 인증 및 인가

### 토큰 기반 인증

```
# Bearer token in Authorization header
GET /api/v1/users
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...

# API key (for server-to-server)
GET /api/v1/data
X-API-Key: sk_live_abc123
```

### 인가 패턴

```typescript
// Resource-level: check ownership
app.get("/api/v1/orders/:id", async (req, res) => {
  const order = await Order.findById(req.params.id);
  if (!order) return res.status(404).json({ error: { code: "not_found" } });
  if (order.userId !== req.user.id) return res.status(403).json({ error: { code: "forbidden" } });
  return res.json({ data: order });
});

// Role-based: check permissions
app.delete("/api/v1/users/:id", requireRole("admin"), async (req, res) => {
  await User.delete(req.params.id);
  return res.status(204).send();
});
```

## 요청 제한

### 헤더

```
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1640000000

# When exceeded
HTTP/1.1 429 Too Many Requests
Retry-After: 60
{
  "error": {
    "code": "rate_limit_exceeded",
    "message": "Rate limit exceeded. Try again in 60 seconds."
  }
}
```

### 요청 제한 티어

| 티어 | 제한 | 윈도우 | 사용 사례 |
|------|-------|--------|----------|
| 익명 | 30/분 | IP별 | 공개 엔드포인트 |
| 인증됨 | 100/분 | 사용자별 | 표준 API 접근 |
| 프리미엄 | 1000/분 | API 키별 | 유료 API 플랜 |
| 내부 | 10000/분 | 서비스별 | 서비스 간 통신 |

## 버전 관리

### URL 경로 버전 관리 (권장)

```
/api/v1/users
/api/v2/users
```

**장점:** 명시적, 라우팅 용이, 캐시 가능
**단점:** 버전 간 URL 변경

### 헤더 버전 관리

```
GET /api/users
Accept: application/vnd.myapp.v2+json
```

**장점:** 깔끔한 URL
**단점:** 테스트 어렵고, 잊기 쉬움

### 버전 관리 전략

```
1. Start with /api/v1/ — don't version until you need to
2. Maintain at most 2 active versions (current + previous)
3. Deprecation timeline:
   - Announce deprecation (6 months notice for public APIs)
   - Add Sunset header: Sunset: Sat, 01 Jan 2026 00:00:00 GMT
   - Return 410 Gone after sunset date
4. Non-breaking changes don't need a new version:
   - Adding new fields to responses
   - Adding new optional query parameters
   - Adding new endpoints
5. Breaking changes require a new version:
   - Removing or renaming fields
   - Changing field types
   - Changing URL structure
   - Changing authentication method
```

## 구현 패턴

### TypeScript (Next.js API 라우트)

```typescript
import { z } from "zod";
import { NextRequest, NextResponse } from "next/server";

const createUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1).max(100),
});

export async function POST(req: NextRequest) {
  const body = await req.json();
  const parsed = createUserSchema.safeParse(body);

  if (!parsed.success) {
    return NextResponse.json({
      error: {
        code: "validation_error",
        message: "Request validation failed",
        details: parsed.error.issues.map(i => ({
          field: i.path.join("."),
          message: i.message,
          code: i.code,
        })),
      },
    }, { status: 422 });
  }

  const user = await createUser(parsed.data);

  return NextResponse.json(
    { data: user },
    {
      status: 201,
      headers: { Location: `/api/v1/users/${user.id}` },
    },
  );
}
```

### Python (Django REST Framework)

```python
from rest_framework import serializers, viewsets, status
from rest_framework.response import Response

class CreateUserSerializer(serializers.Serializer):
    email = serializers.EmailField()
    name = serializers.CharField(max_length=100)

class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ["id", "email", "name", "created_at"]

class UserViewSet(viewsets.ModelViewSet):
    serializer_class = UserSerializer
    permission_classes = [IsAuthenticated]

    def get_serializer_class(self):
        if self.action == "create":
            return CreateUserSerializer
        return UserSerializer

    def create(self, request):
        serializer = CreateUserSerializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        user = UserService.create(**serializer.validated_data)
        return Response(
            {"data": UserSerializer(user).data},
            status=status.HTTP_201_CREATED,
            headers={"Location": f"/api/v1/users/{user.id}"},
        )
```

### Go (net/http)

```go
func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request) {
    var req CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        writeError(w, http.StatusBadRequest, "invalid_json", "Invalid request body")
        return
    }

    if err := req.Validate(); err != nil {
        writeError(w, http.StatusUnprocessableEntity, "validation_error", err.Error())
        return
    }

    user, err := h.service.Create(r.Context(), req)
    if err != nil {
        switch {
        case errors.Is(err, domain.ErrEmailTaken):
            writeError(w, http.StatusConflict, "email_taken", "Email already registered")
        default:
            writeError(w, http.StatusInternalServerError, "internal_error", "Internal error")
        }
        return
    }

    w.Header().Set("Location", fmt.Sprintf("/api/v1/users/%s", user.ID))
    writeJSON(w, http.StatusCreated, map[string]any{"data": user})
}
```

## API 설계 체크리스트

새 엔드포인트 출시 전:

- [ ] 리소스 URL이 네이밍 규칙을 따름 (복수형, kebab-case, 동사 없음)
- [ ] 올바른 HTTP 메서드 사용 (읽기에 GET, 생성에 POST 등)
- [ ] 적절한 상태 코드 반환 (모든 것에 200 아님)
- [ ] 스키마로 입력 유효성 검사 (Zod, Pydantic, Bean Validation)
- [ ] 에러 응답이 코드와 메시지를 포함한 표준 형식 따름
- [ ] 목록 엔드포인트에 페이지네이션 구현 (커서 또는 오프셋)
- [ ] 인증 필요 (또는 명시적으로 공개로 표시)
- [ ] 인가 확인됨 (사용자는 자신의 리소스만 접근 가능)
- [ ] 요청 제한 구성됨
- [ ] 응답이 내부 세부 정보 노출 안 함 (스택 트레이스, SQL 에러)
- [ ] 기존 엔드포인트와 일관된 네이밍 (camelCase vs snake_case)
- [ ] 문서화됨 (OpenAPI/Swagger 스펙 업데이트)

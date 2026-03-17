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
# 리소스는 명사, 복수형, 소문자, kebab-case
GET    /api/v1/users
GET    /api/v1/users/:id
POST   /api/v1/users
PUT    /api/v1/users/:id
PATCH  /api/v1/users/:id
DELETE /api/v1/users/:id

# 관계를 위한 서브 리소스
GET    /api/v1/users/:id/orders
POST   /api/v1/users/:id/orders

# CRUD에 매핑되지 않는 액션 (동사 최소 사용)
POST   /api/v1/orders/:id/cancel
POST   /api/v1/auth/login
POST   /api/v1/auth/refresh
```

### 네이밍 규칙

```
# 좋음
/api/v1/team-members          # 복합 단어 리소스는 kebab-case
/api/v1/orders?status=active  # 필터링에는 쿼리 파라미터
/api/v1/users/123/orders      # 소유 관계에 중첩 리소스

# 나쁨
/api/v1/getUsers              # URL에 동사
/api/v1/user                  # 단수형 (복수형 사용)
/api/v1/team_members          # URL에 snake_case
/api/v1/users/123/getOrders   # 중첩 리소스에 동사
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
# 성공
200 OK                    — GET, PUT, PATCH (응답 본문 있음)
201 Created               — POST (Location 헤더 포함)
204 No Content            — DELETE, PUT (응답 본문 없음)

# 클라이언트 에러
400 Bad Request           — 유효성 검사 실패, 잘못된 형식의 JSON
401 Unauthorized          — 누락 또는 유효하지 않은 인증
403 Forbidden             — 인증됐지만 권한 없음
404 Not Found             — 리소스가 존재하지 않음
409 Conflict              — 중복 항목, 상태 충돌
422 Unprocessable Entity  — 의미론적으로 유효하지 않음 (유효한 JSON이나 잘못된 데이터)
429 Too Many Requests     — 요청 제한 초과

# 서버 에러
500 Internal Server Error — 예상치 못한 실패 (세부 정보 절대 노출 금지)
502 Bad Gateway           — 업스트림 서비스 실패
503 Service Unavailable   — 임시 과부하, Retry-After 포함
```

### 흔한 실수

```
# 나쁨: 모든 것에 200
{ "status": 200, "success": false, "error": "Not found" }

# 좋음: HTTP 상태 코드를 의미론적으로 사용
HTTP/1.1 404 Not Found
{ "error": { "code": "not_found", "message": "User not found" } }

# 나쁨: 유효성 검사 에러에 500
# 좋음: 필드 수준 세부 정보와 함께 400 또는 422

# 나쁨: 생성된 리소스에 200
# 좋음: Location 헤더와 함께 201
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
// Option A: data 래퍼를 포함한 Envelope (공개 API에 권장)
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

// Option B: Flat 응답 (더 간단함, 내부 API에 일반적)
// 성공: 리소스를 직접 반환
// 에러: 에러 객체 반환
// HTTP 상태 코드로 구분
```

## 페이지네이션

### 오프셋 기반 (간단)

```
GET /api/v1/users?page=2&per_page=20

# 구현
SELECT * FROM users
ORDER BY created_at DESC
LIMIT 20 OFFSET 20;
```

**장점:** 구현이 쉽고, "N 페이지로 이동" 지원
**단점:** 큰 오프셋에서 느림 (OFFSET 100000), 동시 삽입 시 일관성 없음

### 커서 기반 (확장 가능)

```
GET /api/v1/users?cursor=eyJpZCI6MTIzfQ&limit=20

# 구현
SELECT * FROM users
WHERE id > :cursor_id
ORDER BY id ASC
LIMIT 21;  -- has_next 결정을 위해 하나 더 가져옴
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
# 단순 동등
GET /api/v1/orders?status=active&customer_id=abc-123

# 비교 연산자 (괄호 표기법 사용)
GET /api/v1/products?price[gte]=10&price[lte]=100
GET /api/v1/orders?created_at[after]=2025-01-01

# 다중 값 (쉼표로 구분)
GET /api/v1/products?category=electronics,clothing

# 중첩 필드 (점 표기법)
GET /api/v1/orders?customer.country=US
```

### 정렬

```
# 단일 필드 (내림차순에 - 접두사)
GET /api/v1/products?sort=-created_at

# 다중 필드 (쉼표로 구분)
GET /api/v1/products?sort=-featured,price,-created_at
```

### 전체 텍스트 검색

```
# 검색 쿼리 파라미터
GET /api/v1/products?q=wireless+headphones

# 필드별 검색
GET /api/v1/users?email=alice
```

### Sparse Fieldset

```
# 지정된 필드만 반환 (페이로드 감소)
GET /api/v1/users?fields=id,name,email
GET /api/v1/orders?fields=id,total,status&include=customer.name
```

## 인증 및 인가

### 토큰 기반 인증

```
# Authorization 헤더에 Bearer 토큰
GET /api/v1/users
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...

# API 키 (서버 간 통신용)
GET /api/v1/data
X-API-Key: sk_live_abc123
```

### 인가 패턴

```typescript
// 리소스 수준: 소유권 확인
app.get("/api/v1/orders/:id", async (req, res) => {
  const order = await Order.findById(req.params.id);
  if (!order) return res.status(404).json({ error: { code: "not_found" } });
  if (order.userId !== req.user.id) return res.status(403).json({ error: { code: "forbidden" } });
  return res.json({ data: order });
});

// 역할 기반: 권한 확인
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

# 초과 시
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
1. /api/v1/로 시작 — 필요할 때까지 버전 관리 하지 않음
2. 최대 2개의 활성 버전 유지 (현재 + 이전)
3. 폐기 타임라인:
   - 폐기 공지 (공개 API는 6개월 전 공지)
   - Sunset 헤더 추가: Sunset: Sat, 01 Jan 2026 00:00:00 GMT
   - 폐기일 이후 410 Gone 반환
4. 하위 호환 변경은 새 버전 불필요:
   - 응답에 새 필드 추가
   - 새 선택적 쿼리 파라미터 추가
   - 새 엔드포인트 추가
5. 하위 비호환 변경은 새 버전 필요:
   - 필드 제거 또는 이름 변경
   - 필드 타입 변경
   - URL 구조 변경
   - 인증 방식 변경
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

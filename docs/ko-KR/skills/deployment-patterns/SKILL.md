---
name: deployment-patterns
description: 웹 애플리케이션을 위한 deployment 워크플로우, CI/CD pipeline 패턴, Docker 컨테이너화, health check, rollback 전략, 그리고 production readiness 체크리스트.
origin: ECC
---

# Deployment Patterns

Production deployment 워크플로우 및 CI/CD best practices.

## When to Activate

- CI/CD 파이프라인 설정 시
- 애플리케이션을 Docker 화할 때
- deployment 전략(blue-green, canary, rolling)을 계획할 때
- health check 및 readiness probe를 구현할 때
- production 릴리스를 준비할 때
- 환경별 설정을 구성할 때

## Deployment Strategies

### Rolling Deployment (Default)

인스턴스를 점진적으로 교체합니다 — rollout 도중 이전 버전과 새 버전이 동시에 실행됩니다.

```
Instance 1: v1 → v2  (update first)
Instance 2: v1        (still running v1)
Instance 3: v1        (still running v1)

Instance 1: v2
Instance 2: v1 → v2  (update second)
Instance 3: v1

Instance 1: v2
Instance 2: v2
Instance 3: v1 → v2  (update last)
```

**장점:** 다운타임 없음, 점진적 rollout
**단점:** 두 버전이 동시에 실행됨 — 하위 호환성(backward-compatible) 변경이 필요함
**사용 시점:** 표준 deployment, 하위 호환성 변경 시

### Blue-Green Deployment

두 개의 동일한 환경을 실행합니다. 트래픽을 원자적으로(atomically) 전환합니다.

```
Blue  (v1) ← traffic
Green (v2)   idle, running new version

# After verification:
Blue  (v1)   idle (becomes standby)
Green (v2) ← traffic
```

**장점:** 즉각적인 rollback(blue로 다시 전환), 깔끔한 전환(cutover)
**단점:** deployment 중 2배의 인프라가 필요함
**사용 시점:** 중요한 서비스, 이슈 무관용 원칙 적용 시

### Canary Deployment

먼저 적은 비율의 트래픽을 새 버전으로 라우팅합니다.

```
v1: 95% of traffic
v2:  5% of traffic  (canary)

# If metrics look good:
v1: 50% of traffic
v2: 50% of traffic

# Final:
v2: 100% of traffic
```

**장점:** 전체 rollout 전에 실제 트래픽으로 이슈를 포착함
**단점:** 트래픽 분할 인프라와 모니터링이 필요함
**사용 시점:** 트래픽이 많은 서비스, 위험한 변경 사항, feature flag 사용 시

## Docker

### Multi-Stage Dockerfile (Node.js)

```dockerfile
# Stage 1: Install dependencies
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --production=false

# Stage 2: Build
FROM node:22-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build
RUN npm prune --production

# Stage 3: Production image
FROM node:22-alpine AS runner
WORKDIR /app

RUN addgroup -g 1001 -S appgroup && adduser -S appuser -u 1001
USER appuser

COPY --from=builder --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --from=builder --chown=appuser:appgroup /app/dist ./dist
COPY --from=builder --chown=appuser:appgroup /app/package.json ./

ENV NODE_ENV=production
EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

CMD ["node", "dist/server.js"]
```

### Multi-Stage Dockerfile (Go)

```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /server ./cmd/server

FROM alpine:3.19 AS runner
RUN apk --no-cache add ca-certificates
RUN adduser -D -u 1001 appuser
USER appuser

COPY --from=builder /server /server

EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s CMD wget -qO- http://localhost:8080/health || exit 1
CMD ["/server"]
```

### Multi-Stage Dockerfile (Python/Django)

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
RUN pip install --no-cache-dir uv
COPY requirements.txt .
RUN uv pip install --system --no-cache -r requirements.txt

FROM python:3.12-slim AS runner
WORKDIR /app

RUN useradd -r -u 1001 appuser
USER appuser

COPY --from=builder /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY --from=builder /usr/local/bin /usr/local/bin
COPY . .

ENV PYTHONUNBUFFERED=1
EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=3s CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health/')" || exit 1
CMD ["gunicorn", "config.wsgi:application", "--bind", "0.0.0.0:8000", "--workers", "4"]
```

### Docker Best Practices

```
# GOOD practices
- Use specific version tags (node:22-alpine, not node:latest)
- Multi-stage builds to minimize image size
- Run as non-root user
- Copy dependency files first (layer caching)
- Use .dockerignore to exclude node_modules, .git, tests
- Add HEALTHCHECK instruction
- Set resource limits in docker-compose or k8s

# BAD practices
- Running as root
- Using :latest tags
- Copying entire repo in one COPY layer
- Installing dev dependencies in production image
- Storing secrets in image (use env vars or secrets manager)
```

## CI/CD Pipeline

### GitHub Actions (Standard Pipeline)

```yaml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck
      - run: npm test -- --coverage
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: coverage
          path: coverage/

  build:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production
    steps:
      - name: Deploy to production
        run: |
          # Platform-specific deployment command
          # Railway: railway up
          # Vercel: vercel --prod
          # K8s: kubectl set image deployment/app app=ghcr.io/${{ github.repository }}:${{ github.sha }}
          echo "Deploying ${{ github.sha }}"
```

### Pipeline Stages

```
PR opened:
  lint → typecheck → unit tests → integration tests → preview deploy

Merged to main:
  lint → typecheck → unit tests → integration tests → build image → deploy staging → smoke tests → deploy production
```

## Health Checks

### Health Check Endpoint

```typescript
// Simple health check
app.get("/health", (req, res) => {
  res.status(200).json({ status: "ok" });
});

// Detailed health check (for internal monitoring)
app.get("/health/detailed", async (req, res) => {
  const checks = {
    database: await checkDatabase(),
    redis: await checkRedis(),
    externalApi: await checkExternalApi(),
  };

  const allHealthy = Object.values(checks).every(c => c.status === "ok");

  res.status(allHealthy ? 200 : 503).json({
    status: allHealthy ? "ok" : "degraded",
    timestamp: new Date().toISOString(),
    version: process.env.APP_VERSION || "unknown",
    uptime: process.uptime(),
    checks,
  });
});

async function checkDatabase(): Promise<HealthCheck> {
  try {
    await db.query("SELECT 1");
    return { status: "ok", latency_ms: 2 };
  } catch (err) {
    return { status: "error", message: "Database unreachable" };
  }
}
```

### Kubernetes Probes

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 10
  periodSeconds: 30
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 10
  failureThreshold: 2

startupProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 0
  periodSeconds: 5
  failureThreshold: 30    # 30 * 5s = 150s max startup time
```

## Environment Configuration

### Twelve-Factor App Pattern

```bash
# All config via environment variables — never in code
DATABASE_URL=postgres://user:pass@host:5432/db
REDIS_URL=redis://host:6379/0
API_KEY=${API_KEY}           # injected by secrets manager
LOG_LEVEL=info
PORT=3000

# Environment-specific behavior
NODE_ENV=production          # or staging, development
APP_ENV=production           # explicit app environment
```

### Configuration Validation

```typescript
import { z } from "zod";

const envSchema = z.object({
  NODE_ENV: z.enum(["development", "staging", "production"]),
  PORT: z.coerce.number().default(3000),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
  LOG_LEVEL: z.enum(["debug", "info", "warn", "error"]).default("info"),
});

// Validate at startup — fail fast if config is wrong
export const env = envSchema.parse(process.env);
```

## Rollback Strategy

### Instant Rollback

```bash
# Docker/Kubernetes: point to previous image
kubectl rollout undo deployment/app

# Vercel: promote previous deployment
vercel rollback

# Railway: redeploy previous commit
railway up --commit <previous-sha>

# Database: rollback migration (if reversible)
npx prisma migrate resolve --rolled-back <migration-name>
```

### Rollback Checklist

- [ ] 이전 이미지/artifact가 사용 가능하며 태그가 지정됨
- [ ] 데이터베이스 마이그레이션이 하위 호환 가능함 (파괴적인 변경 없음)
- [ ] feature flag를 통해 배포 없이 새 기능을 비활성화할 수 있음
- [ ] 에러율 급증에 대한 모니터링 알림이 구성됨
- [ ] production 릴리스 전에 staging에서 rollback을 테스트함

## Production Readiness Checklist

모든 production deployment 전 확인 사항:

### Application
- [ ] 모든 테스트 통과 (단위, 통합, E2E)
- [ ] 코드나 설정 파일에 하드코딩된 비밀 정보 없음
- [ ] 에러 핸들링이 모든 edge case를 커버함
- [ ] 로깅이 구조화(JSON)되어 있으며 PII(개인 식별 정보)를 포함하지 않음
- [ ] health check endpoint가 유의미한 상태를 반환함

### Infrastructure
- [ ] Docker 이미지가 재현 가능하게 빌드됨 (버전 고정)
- [ ] 환경 변수가 문서화되었으며 시작 시 검증됨
- [ ] 리소스 제한 설정 (CPU, 메모리)
- [ ] 수평 확장(Horizontal scaling) 구성 (최소/최대 인스턴스)
- [ ] 모든 endpoint에 SSL/TLS 활성화

### Monitoring
- [ ] 애플리케이션 메트릭 내보내기 (요청률, 지연 시간, 에러)
- [ ] 에러율 > 임계값에 대한 알림 구성
- [ ] 로그 집계(aggregation) 설정 (구조화된 로그, 검색 가능)
- [ ] health endpoint에 대한 업타임 모니터링

### Security
- [ ] 종속성의 CVE 스캔 완료
- [ ] 허용된 origin에 대해서만 CORS 구성
- [ ] 공개 endpoint에 rate limiting 활성화
- [ ] 인증 및 권한 부여 확인 완료
- [ ] 보안 헤더 설정 (CSP, HSTS, X-Frame-Options)

### Operations
- [ ] rollback 계획이 문서화되고 테스트됨
- [ ] production 규모의 데이터에 대해 데이터베이스 마이그레이션 테스트 완료
- [ ] 일반적인 실패 시나리오에 대한 Runbook 마련
- [ ] On-call 로테이션 및 에스컬레이션 경로 정의

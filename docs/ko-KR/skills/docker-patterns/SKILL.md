---
name: docker-patterns
description: 로컬 개발, container 보안, networking, volume 전략 및 멀티 서비스 orchestration을 위한 Docker 및 Docker Compose 패턴.
origin: ECC
---

# Docker Patterns

컨테이너화된 개발을 위한 Docker 및 Docker Compose best practices.

## 활성화 시점

- 로컬 개발을 위한 Docker Compose 설정 시
- 멀티 container 아키텍처 설계 시
- container networking 또는 volume 문제 해결 시
- 보안 및 크기 최적화를 위한 Dockerfile 검토 시
- 로컬 개발 환경에서 컨테이너화된 workflow로 마이그레이션 시

## 로컬 개발을 위한 Docker Compose

### 표준 웹 앱 스택

```yaml
# docker-compose.yml
services:
  app:
    build:
      context: .
      target: dev                     # Use dev stage of multi-stage Dockerfile
    ports:
      - "3000:3000"
    volumes:
      - .:/app                        # Bind mount for hot reload
      - /app/node_modules             # Anonymous volume -- preserves container deps
    environment:
      - DATABASE_URL=postgres://postgres:postgres@db:5432/app_dev
      - REDIS_URL=redis://redis:6379/0
      - NODE_ENV=development
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    command: npm run dev

  db:
    image: postgres:16-alpine
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: app_dev
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./scripts/init-db.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redisdata:/data

  mailpit:                            # Local email testing
    image: axllent/mailpit
    ports:
      - "8025:8025"                   # Web UI
      - "1025:1025"                   # SMTP

volumes:
  pgdata:
  redisdata:
```

### 개발용 vs 프로덕션용 Dockerfile

```dockerfile
# Stage: dependencies
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

# Stage: dev (hot reload, debug tools)
FROM node:22-alpine AS dev
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
EXPOSE 3000
CMD ["npm", "run", "dev"]

# Stage: build
FROM node:22-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build && npm prune --production

# Stage: production (minimal image)
FROM node:22-alpine AS production
WORKDIR /app
RUN addgroup -g 1001 -S appgroup && adduser -S appuser -u 1001
USER appuser
COPY --from=build --chown=appuser:appgroup /app/dist ./dist
COPY --from=build --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --from=build --chown=appuser:appgroup /app/package.json ./
ENV NODE_ENV=production
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s CMD wget -qO- http://localhost:3000/health || exit 1
CMD ["node", "dist/server.js"]
```

### Override 파일

```yaml
# docker-compose.override.yml (auto-loaded, dev-only settings)
services:
  app:
    environment:
      - DEBUG=app:*
      - LOG_LEVEL=debug
    ports:
      - "9229:9229"                   # Node.js debugger

# docker-compose.prod.yml (explicit for production)
services:
  app:
    build:
      target: production
    restart: always
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
```

```bash
# Development (auto-loads override)
docker compose up

# Production
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

## Networking

### Service Discovery

동일한 Compose network 내의 서비스들은 서비스 이름으로 식별됩니다:
```
# From "app" container:
postgres://postgres:postgres@db:5432/app_dev    # "db" resolves to the db container
redis://redis:6379/0                             # "redis" resolves to the redis container
```

### Custom Networks

```yaml
services:
  frontend:
    networks:
      - frontend-net

  api:
    networks:
      - frontend-net
      - backend-net

  db:
    networks:
      - backend-net              # Only reachable from api, not frontend

networks:
  frontend-net:
  backend-net:
```

### 필요한 것만 노출하기

```yaml
services:
  db:
    ports:
      - "127.0.0.1:5432:5432"   # Only accessible from host, not network
    # Omit ports entirely in production -- accessible only within Docker network
```

## Volume 전략

```yaml
volumes:
  # Named volume: container 재시작 후에도 유지되며, Docker에 의해 관리됨
  pgdata:

  # Bind mount: 호스트 디렉토리를 container 내부로 매핑 (개발용)
  # - ./src:/app/src

  # Anonymous volume: bind mount override로부터 container에서 생성된 콘텐츠를 보존
  # - /app/node_modules
```

### 일반적인 패턴

```yaml
services:
  app:
    volumes:
      - .:/app                   # Source code (bind mount for hot reload)
      - /app/node_modules        # Protect container's node_modules from host
      - /app/.next               # Protect build cache

  db:
    volumes:
      - pgdata:/var/lib/postgresql/data          # Persistent data
      - ./scripts/init.sql:/docker-entrypoint-initdb.d/init.sql  # Init scripts
```

## Container 보안

### Dockerfile 보안 강화

```dockerfile
# 1. 특정 태그 사용 (절대 :latest 사용 금지)
FROM node:22.12-alpine3.20

# 2. non-root 사용자로 실행
RUN addgroup -g 1001 -S app && adduser -S app -u 1001
USER app

# 3. capability 제거 (compose에서 설정)
# 4. 가능한 경우 read-only 루트 파일시스템 사용
# 5. 이미지 레이어에 secret 포함 금지
```

### Compose 보안

```yaml
services:
  app:
    security_opt:
      - no-new-privileges:true
    read_only: true
    tmpfs:
      - /tmp
      - /app/.cache
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE          # Only if binding to ports < 1024
```

### Secret 관리

```yaml
# 권장: 환경 변수 사용 (런타임에 주입)
services:
  app:
    env_file:
      - .env                     # Never commit .env to git
    environment:
      - API_KEY                  # Inherits from host environment

# 권장: Docker secrets (Swarm 모드)
secrets:
  db_password:
    file: ./secrets/db_password.txt

services:
  db:
    secrets:
      - db_password

# 금지: 이미지에 하드코딩
# ENV API_KEY=sk-proj-xxxxx      # NEVER DO THIS
```

## .dockerignore

```
node_modules
.git
.env
.env.*
dist
coverage
*.log
.next
.cache
docker-compose*.yml
Dockerfile*
README.md
tests/
```

## Debugging

### 주요 명령어

```bash
# View logs
docker compose logs -f app           # Follow app logs
docker compose logs --tail=50 db     # Last 50 lines from db

# Execute commands in running container
docker compose exec app sh           # Shell into app
docker compose exec db psql -U postgres  # Connect to postgres

# Inspect
docker compose ps                     # Running services
docker compose top                    # Processes in each container
docker stats                          # Resource usage

# Rebuild
docker compose up --build             # Rebuild images
docker compose build --no-cache app   # Force full rebuild

# Clean up
docker compose down                   # Stop and remove containers
docker compose down -v                # Also remove volumes (DESTRUCTIVE)
docker system prune                   # Remove unused images/containers
```

### Network 문제 디버깅

```bash
# Check DNS resolution inside container
docker compose exec app nslookup db

# Check connectivity
docker compose exec app wget -qO- http://api:3000/health

# Inspect network
docker network ls
docker network inspect <project>_default
```

## Anti-Patterns

```
# 금지: orchestration 없이 프로덕션에서 docker compose 사용
# 프로덕션의 멀티 container 워크로드에는 Kubernetes, ECS 또는 Docker Swarm 사용

# 금지: volume 없이 container에 데이터 저장
# 컨테이너는 휘발성입니다. volume이 없으면 재시작 시 모든 데이터가 손실됩니다.

# 금지: root 권한으로 실행
# 항상 non-root 사용자를 생성하고 사용하세요.

# 금지: :latest 태그 사용
# 재현 가능한 빌드를 위해 특정 버전을 고정하세요.

# 금지: 모든 서비스를 하나의 거대한 container에 포함
# 관심사 분리: container당 하나의 프로세스

# 금지: docker-compose.yml에 secret 포함
# .env 파일(gitignored) 또는 Docker secrets를 사용하세요.
```

---
name: database-migrations
description: 스키마 변경, 데이터 migration, rollback 및 PostgreSQL, MySQL, 주요 ORM(Prisma, Drizzle, Django, TypeORM, golang-migrate) 전반의 zero-downtime 배포를 위한 database migration best practice입니다.
origin: ECC
---

# Database Migration 패턴

운영 시스템을 위한 안전하고 가역적인 데이터베이스 스키마 변경.

## 활성화 시점

- 데이터베이스 테이블 생성 또는 수정
- 컬럼 또는 인덱스 추가/삭제
- 데이터 migration 실행 (backfill, transform)
- zero-downtime 스키마 변경 계획
- 새 프로젝트를 위한 migration 도구 설정

## 핵심 원칙

1. **모든 변경은 migration임** — 운영 데이터베이스를 수동으로 수정하지 않음
2. **운영 환경에서 migration은 정방향(forward-only)으로만 진행됨** — rollback은 새로운 정방향 migration을 사용함
3. **스키마와 데이터 migration은 분리함** — 하나의 migration에 DDL과 DML을 섞지 않음
4. **운영 규모의 데이터로 migration을 테스트함** — 100개 행에서 작동하는 migration이 1,000만 개 행에서는 lock을 유발할 수 있음
5. **배포된 migration은 불변(immutable)임** — 운영 환경에서 실행된 migration은 절대 수정하지 않음

## Migration 안전 체크리스트

migration 적용 전 확인 사항:

- [ ] Migration에 UP과 DOWN이 모두 있음 (또는 명시적으로 되돌릴 수 없음으로 표시됨)
- [ ] 대규모 테이블에 전체 테이블 lock이 없음 (concurrent 작업을 사용함)
- [ ] 새 컬럼에 기본값이 있거나 nullable임 (기본값 없이 NOT NULL을 추가하지 않음)
- [ ] 인덱스가 concurrently하게 생성됨 (기존 테이블의 경우 CREATE TABLE과 함께 생성하지 않음)
- [ ] 데이터 backfill이 스키마 변경과 별개의 migration으로 분리됨
- [ ] 운영 데이터의 복사본으로 테스트됨
- [ ] Rollback 계획이 문서화됨

## PostgreSQL 패턴

### 안전하게 컬럼 추가하기

```sql
-- GOOD: Nullable column, no lock
ALTER TABLE users ADD COLUMN avatar_url TEXT;

-- GOOD: Column with default (Postgres 11+ is instant, no rewrite)
ALTER TABLE users ADD COLUMN is_active BOOLEAN NOT NULL DEFAULT true;

-- BAD: NOT NULL without default on existing table (requires full rewrite)
ALTER TABLE users ADD COLUMN role TEXT NOT NULL;
-- This locks the table and rewrites every row
```

### 다운타임 없이 인덱스 추가하기

```sql
-- BAD: Blocks writes on large tables
CREATE INDEX idx_users_email ON users (email);

-- GOOD: Non-blocking, allows concurrent writes
CREATE INDEX CONCURRENTLY idx_users_email ON users (email);

-- Note: CONCURRENTLY cannot run inside a transaction block
-- Most migration tools need special handling for this
```

### 컬럼 이름 변경 (Zero-Downtime)

운영 환경에서 직접 이름을 변경하지 마세요. expand-contract 패턴을 사용하세요:

```sql
-- Step 1: Add new column (migration 001)
ALTER TABLE users ADD COLUMN display_name TEXT;

-- Step 2: Backfill data (migration 002, data migration)
UPDATE users SET display_name = username WHERE display_name IS NULL;

-- Step 3: Update application code to read/write both columns
-- Deploy application changes

-- Step 4: Stop writing to old column, drop it (migration 003)
ALTER TABLE users DROP COLUMN username;
```

### 안전하게 컬럼 삭제하기

```sql
-- Step 1: Remove all application references to the column
-- Step 2: Deploy application without the column reference
-- Step 3: Drop column in next migration
ALTER TABLE orders DROP COLUMN legacy_status;

-- For Django: use SeparateDatabaseAndState to remove from model
-- without generating DROP COLUMN (then drop in next migration)
```

### 대규모 데이터 Migration

```sql
-- BAD: Updates all rows in one transaction (locks table)
UPDATE users SET normalized_email = LOWER(email);

-- GOOD: Batch update with progress
DO $$
DECLARE
  batch_size INT := 10000;
  rows_updated INT;
BEGIN
  LOOP
    UPDATE users
    SET normalized_email = LOWER(email)
    WHERE id IN (
      SELECT id FROM users
      WHERE normalized_email IS NULL
      LIMIT batch_size
      FOR UPDATE SKIP LOCKED
    );
    GET DIAGNOSTICS rows_updated = ROW_COUNT;
    RAISE NOTICE 'Updated % rows', rows_updated;
    EXIT WHEN rows_updated = 0;
    COMMIT;
  END LOOP;
END $$;
```

## Prisma (TypeScript/Node.js)

### Workflow

```bash
# Create migration from schema changes
npx prisma migrate dev --name add_user_avatar

# Apply pending migrations in production
npx prisma migrate deploy

# Reset database (dev only)
npx prisma migrate reset

# Generate client after schema changes
npx prisma generate
```

### 스키마 예시

```prisma
model User {
  id        String   @id @default(cuid())
  email     String   @unique
  name      String?
  avatarUrl String?  @map("avatar_url")
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")
  orders    Order[]

  @@map("users")
  @@index([email])
}
```

### 커스텀 SQL Migration

Prisma로 표현할 수 없는 작업(concurrent 인덱스, 데이터 backfill 등)의 경우:

```bash
# Create empty migration, then edit the SQL manually
npx prisma migrate dev --create-only --name add_email_index
```

```sql
-- migrations/20240115_add_email_index/migration.sql
-- Prisma cannot generate CONCURRENTLY, so we write it manually
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_users_email ON users (email);
```

## Drizzle (TypeScript/Node.js)

### Workflow

```bash
# Generate migration from schema changes
npx drizzle-kit generate

# Apply migrations
npx drizzle-kit migrate

# Push schema directly (dev only, no migration file)
npx drizzle-kit push
```

### 스키마 예시

```typescript
import { pgTable, text, timestamp, uuid, boolean } from "drizzle-orm/pg-core";

export const users = pgTable("users", {
  id: uuid("id").primaryKey().defaultRandom(),
  email: text("email").notNull().unique(),
  name: text("name"),
  isActive: boolean("is_active").notNull().default(true),
  createdAt: timestamp("created_at").notNull().defaultNow(),
  updatedAt: timestamp("updated_at").notNull().defaultNow(),
});
```

## Django (Python)

### Workflow

```bash
# Generate migration from model changes
python manage.py makemigrations

# Apply migrations
python manage.py migrate

# Show migration status
python manage.py showmigrations

# Generate empty migration for custom SQL
python manage.py makemigrations --empty app_name -n description
```

### 데이터 Migration

```python
from django.db import migrations

def backfill_display_names(apps, schema_editor):
    User = apps.get_model("accounts", "User")
    batch_size = 5000
    users = User.objects.filter(display_name="")
    while users.exists():
        batch = list(users[:batch_size])
        for user in batch:
            user.display_name = user.username
        User.objects.bulk_update(batch, ["display_name"], batch_size=batch_size)

def reverse_backfill(apps, schema_editor):
    pass  # Data migration, no reverse needed

class Migration(migrations.Migration):
    dependencies = [("accounts", "0015_add_display_name")]

    operations = [
        migrations.RunPython(backfill_display_names, reverse_backfill),
    ]
```

### SeparateDatabaseAndState

데이터베이스에서 즉시 삭제하지 않고 Django 모델에서 컬럼 제거:

```python
class Migration(migrations.Migration):
    operations = [
        migrations.SeparateDatabaseAndState(
            state_operations=[
                migrations.RemoveField(model_name="user", name="legacy_field"),
            ],
            database_operations=[],  # Don't touch the DB yet
        ),
    ]
```

## golang-migrate (Go)

### Workflow

```bash
# Create migration pair
migrate create -ext sql -dir migrations -seq add_user_avatar

# Apply all pending migrations
migrate -path migrations -database "$DATABASE_URL" up

# Rollback last migration
migrate -path migrations -database "$DATABASE_URL" down 1

# Force version (fix dirty state)
migrate -path migrations -database "$DATABASE_URL" force VERSION
```

### Migration 파일

```sql
-- migrations/000003_add_user_avatar.up.sql
ALTER TABLE users ADD COLUMN avatar_url TEXT;
CREATE INDEX CONCURRENTLY idx_users_avatar ON users (avatar_url) WHERE avatar_url IS NOT NULL;

-- migrations/000003_add_user_avatar.down.sql
DROP INDEX IF EXISTS idx_users_avatar;
ALTER TABLE users DROP COLUMN IF EXISTS avatar_url;
```

## Zero-Downtime Migration 전략

중요한 운영 변경사항의 경우 expand-contract 패턴을 따르세요:

```
1단계: 확장 (EXPAND)
  - 새 컬럼/테이블 추가 (nullable 또는 기본값 포함)
  - 배포: 앱이 기존 컬럼과 새 컬럼 모두에 쓰기(write) 수행
  - 기존 데이터 backfill

2단계: 이관 (MIGRATE)
  - 배포: 앱이 새 컬럼에서 읽기(read) 수행, 쓰기는 양쪽 모두 수행
  - 데이터 일관성 검증

3단계: 축소 (CONTRACT)
  - 배포: 앱이 새 컬럼만 사용
  - 별도의 migration으로 이전 컬럼/테이블 삭제
```

### 타임라인 예시

```
1일차: Migration으로 new_status 컬럼 추가 (nullable)
1일차: 앱 v2 배포 — status와 new_status 모두에 쓰기 수행
2일차: 기존 행들에 대해 backfill migration 실행
3일차: 앱 v3 배포 — new_status에서만 읽기 수행
7일차: Migration으로 이전 status 컬럼 삭제
```

## 안티 패턴

| 안티 패턴 | 실패 원인 | 더 나은 접근 방식 |
|-------------|-------------|-----------------|
| 운영 환경에서 수동 SQL 실행 | 감사 추적(audit trail) 불가, 재현 불가능 | 항상 migration 파일 사용 |
| 배포된 migration 수정 | 환경 간의 차이(drift) 유발 | 대신 새로운 migration 생성 |
| 기본값 없이 NOT NULL 추가 | 테이블 lock 유발, 모든 행 재작성 | nullable로 추가, backfill 후 제약 조건 추가 |
| 대규모 테이블에 inline 인덱스 생성 | 빌드 중 쓰기 작업 차단 | CREATE INDEX CONCURRENTLY 사용 |
| 스키마와 데이터를 한 migration에 포함 | rollback이 어렵고 트랜잭션이 길어짐 | migration 분리 |
| 코드 제거 전 컬럼 삭제 | 누락된 컬럼으로 인해 애플리케이션 에러 발생 | 코드 먼저 제거 후 다음 배포 시 컬럼 삭제 |

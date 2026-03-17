---
name: update-docs
description: 코드베이스를 기준으로 문서를 동기화하고 생성된 섹션을 갱신합니다.
---

# 문서 업데이트

문서를 코드베이스와 동기화하고, 원본 소스 파일에서 생성합니다.

## 1단계: 원본 소스 식별

| 소스 | 생성 대상 |
|------|----------|
| `package.json` scripts | 사용 가능한 커맨드 참조 |
| `.env.example` | 환경 변수 문서 |
| `openapi.yaml` / 라우트 파일 | API 엔드포인트 참조 |
| 소스 코드 exports | 공개 API 문서 |
| `Dockerfile` / `docker-compose.yml` | 인프라 설정 문서 |

## 2단계: 스크립트 참조 생성

1. `package.json` (또는 `Makefile`, `Cargo.toml`, `pyproject.toml`) 읽기
2. 모든 스크립트/커맨드와 설명 추출
3. 참조 테이블 생성:

```markdown
| Command | Description |
|---------|-------------|
| `npm run dev` | Start development server with hot reload |
| `npm run build` | Production build with type checking |
| `npm test` | Run test suite with coverage |
```

## 3단계: 환경 변수 문서 생성

1. `.env.example` (또는 `.env.template`, `.env.sample`) 읽기
2. 모든 변수와 용도 추출
3. 필수 vs 선택으로 분류
4. 예상 형식과 유효 값 문서화

```markdown
| Variable | Required | Description | Example |
|----------|----------|-------------|---------|
| `DATABASE_URL` | Yes | PostgreSQL connection string | `postgres://user:pass@host:5432/db` |
| `LOG_LEVEL` | No | Logging verbosity (default: info) | `debug`, `info`, `warn`, `error` |
```

## 4단계: 기여 가이드 업데이트

`docs/CONTRIBUTING.md`를 생성 또는 업데이트합니다:
- 개발 환경 설정 (사전 요구 사항, 설치 단계)
- 사용 가능한 스크립트와 용도
- 테스트 절차 (실행 방법, 새 테스트 작성 방법)
- 코드 스타일 적용 (linter, formatter, pre-commit hook)
- PR 제출 체크리스트

## 5단계: 운영 매뉴얼 업데이트

`docs/RUNBOOK.md`를 생성 또는 업데이트합니다:
- 배포 절차 (단계별)
- 헬스 체크 엔드포인트 및 모니터링
- 일반적인 이슈와 해결 방법
- 롤백 절차
- 알림 및 에스컬레이션 경로

## 6단계: 오래된 항목 점검

1. 90일 이상 수정되지 않은 문서 파일 찾기
2. 최근 소스 코드 변경 사항과 교차 참조
3. 잠재적으로 오래된 문서를 수동 검토 대상으로 표시

## 7단계: 요약 표시

```
Documentation Update
──────────────────────────────
Updated:  docs/CONTRIBUTING.md (scripts table)
Updated:  docs/ENV.md (3 new variables)
Flagged:  docs/DEPLOY.md (142 days stale)
Skipped:  docs/API.md (no changes detected)
──────────────────────────────
```

## 규칙

- **단일 원본**: 항상 코드에서 생성하고, 생성된 섹션을 수동으로 편집하지 않기
- **수동 섹션 보존**: 생성된 섹션만 업데이트; 수기 작성 내용은 그대로 유지
- **생성된 콘텐츠 표시**: 생성된 섹션 주변에 `<!-- AUTO-GENERATED -->` 마커 사용
- **요청 없이 문서 생성하지 않기**: 커맨드가 명시적으로 요청한 경우에만 새 문서 파일 생성

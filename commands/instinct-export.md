---
name: instinct-export
description: 프로젝트/글로벌 범위의 instinct를 파일로 내보냅니다
command: /instinct-export
---

# Instinct Export 커맨드

instinct를 공유 가능한 형식으로 내보냅니다. 다음 용도에 적합합니다:
- 팀원과 공유
- 새 머신으로 이전
- 프로젝트 컨벤션에 기여

## 사용법

```
/instinct-export                           # Export all personal instincts
/instinct-export --domain testing          # Export only testing instincts
/instinct-export --min-confidence 0.7      # Only export high-confidence instincts
/instinct-export --output team-instincts.yaml
/instinct-export --scope project --output project-instincts.yaml
```

## 수행할 작업

1. 현재 프로젝트 컨텍스트 감지
2. 선택한 범위별로 instinct 로드:
   - `project`: 현재 프로젝트만
   - `global`: 글로벌만
   - `all`: 프로젝트 + 글로벌 병합 (기본값)
3. 필터 적용 (`--domain`, `--min-confidence`)
4. YAML 형식으로 파일에 내보내기 (출력 경로가 없으면 stdout으로 출력)

## 출력 형식

YAML 파일 생성:

```yaml
# Instincts Export
# Generated: 2025-01-22
# Source: personal
# Count: 12 instincts

---
id: prefer-functional-style
trigger: "when writing new functions"
confidence: 0.8
domain: code-style
source: session-observation
scope: project
project_id: a1b2c3d4e5f6
project_name: my-app
---

# Prefer Functional Style

## Action
Use functional patterns over classes.
```

## 플래그

- `--domain <name>`: 지정된 도메인만 내보내기
- `--min-confidence <n>`: 최소 신뢰도 임계값
- `--output <file>`: 출력 파일 경로 (생략 시 stdout으로 출력)
- `--scope <project|global|all>`: 내보내기 범위 (기본값: `all`)

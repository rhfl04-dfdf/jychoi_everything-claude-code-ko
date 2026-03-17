---
description: 코드베이스 탐색을 위한 codemap 업데이트
agent: doc-updater
subtask: true
---

# Update Codemaps 커맨드

현재 코드베이스 구조를 반영하도록 codemap 업데이트: $ARGUMENTS

## 작업 내용

`docs/CODEMAPS/` 디렉토리에 codemap 생성 또는 업데이트:

1. **코드베이스 구조 분석**
2. **컴포넌트 맵 생성**
3. **관계 문서화**
4. **탐색 가이드 업데이트**

## Codemap 유형

### 아키텍처 맵
```
docs/CODEMAPS/ARCHITECTURE.md
```
- 고수준 시스템 개요
- 컴포넌트 관계
- 데이터 흐름 다이어그램

### 모듈 맵
```
docs/CODEMAPS/MODULES.md
```
- 모듈 설명
- 공개 API
- 의존성

### 파일 맵
```
docs/CODEMAPS/FILES.md
```
- 디렉토리 구조
- 파일 용도
- 주요 파일

## Codemap 형식

### [모듈명]

**용도**: [간략한 설명]

**위치**: `src/[path]/`

**주요 파일**:
- `file1.ts` - [용도]
- `file2.ts` - [용도]

**의존성**:
- [모듈 A]
- [모듈 B]

**Export**:
- `functionName()` - [설명]
- `ClassName` - [설명]

**사용 예시**:
```typescript
import { functionName } from '@/module'
```

## 생성 프로세스

1. 디렉토리 구조 스캔
2. import/export 파싱
3. 의존성 그래프 구축
4. 마크다운 맵 생성
5. 링크 검증

---

**팁**: 새 모듈을 추가하거나 대규모 리팩토링 시 codemap을 최신 상태로 유지하세요.

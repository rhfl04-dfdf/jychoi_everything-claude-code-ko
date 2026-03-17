---
description: 외부 소스에서 instinct 가져오기
agent: build
---

# Instinct Import 명령어

파일 또는 URL에서 instinct를 가져옵니다: $ARGUMENTS

## 작업 내용

continuous-learning-v2 시스템에 instinct를 가져옵니다.

## 가져오기 소스

### 파일에서 가져오기
```
/instinct-import path/to/instincts.json
```

### URL에서 가져오기
```
/instinct-import https://example.com/instincts.json
```

### 팀 공유에서 가져오기
```
/instinct-import @teammate/instincts
```

## 가져오기 형식

예상 JSON 구조:

```json
{
  "instincts": [
    {
      "trigger": "[situation description]",
      "action": "[recommended action]",
      "confidence": 0.7,
      "category": "coding",
      "source": "imported"
    }
  ],
  "metadata": {
    "version": "1.0",
    "exported": "2025-01-15T10:00:00Z",
    "author": "username"
  }
}
```

## 가져오기 프로세스

1. **형식 검증** - JSON 구조 확인
2. **중복 제거** - 기존 instinct 건너뛰기
3. **신뢰도 조정** - 가져온 항목의 신뢰도 감소 (x0.8)
4. **병합** - 로컬 instinct 저장소에 추가
5. **보고** - 가져오기 요약 표시

## 가져오기 보고서

```
Import Summary
==============
Source: [path or URL]
Total in file: X
Imported: Y
Skipped (duplicates): Z
Errors: W

Imported Instincts:
- [trigger] (confidence: 0.XX)
- [trigger] (confidence: 0.XX)
...
```

## 충돌 해결

중복 가져오기 시:
- 더 높은 신뢰도 버전 유지
- 적용 횟수 병합
- 타임스탬프 업데이트

---

**팁**: 가져오기 후 `/instinct-status`로 가져온 instinct를 검토하세요.

---
description: 공유를 위한 instinct 내보내기
agent: build
---

# Instinct Export 명령어

다른 사용자와 공유하기 위해 instinct를 내보냅니다: $ARGUMENTS

## 작업 내용

continuous-learning-v2 시스템에서 instinct를 내보냅니다.

## 내보내기 옵션

### 전체 내보내기
```
/instinct-export
```

### 높은 신뢰도만 내보내기
```
/instinct-export --min-confidence 0.8
```

### 카테고리별 내보내기
```
/instinct-export --category coding
```

### 특정 경로로 내보내기
```
/instinct-export --output ./my-instincts.json
```

## 내보내기 형식

```json
{
  "instincts": [
    {
      "id": "instinct-123",
      "trigger": "[situation description]",
      "action": "[recommended action]",
      "confidence": 0.85,
      "category": "coding",
      "applications": 10,
      "successes": 9,
      "source": "session-observation"
    }
  ],
  "metadata": {
    "version": "1.0",
    "exported": "2025-01-15T10:00:00Z",
    "author": "username",
    "total": 25,
    "filter": "confidence >= 0.8"
  }
}
```

## 내보내기 보고서

```
Export Summary
==============
Output: ./instincts-export.json
Total instincts: X
Filtered: Y
Exported: Z

Categories:
- coding: N
- testing: N
- security: N
- git: N

Top Instincts (by confidence):
1. [trigger] (0.XX)
2. [trigger] (0.XX)
3. [trigger] (0.XX)
```

## 공유

내보내기 후:
- JSON 파일을 직접 공유
- 팀 저장소에 업로드
- instinct 레지스트리에 게시

---

**팁**: 더 나은 품질의 공유를 위해 높은 신뢰도(>0.8)의 instinct를 내보내세요.

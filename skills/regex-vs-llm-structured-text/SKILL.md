---
name: regex-vs-llm-structured-text
description: 구조화된 텍스트 파싱 시 regex와 LLM 중 선택하는 의사결정 프레임워크 — regex로 시작하고, 신뢰도가 낮은 엣지 케이스에만 LLM을 추가한다.
origin: ECC
---

# 구조화된 텍스트 파싱: Regex vs LLM

구조화된 텍스트(퀴즈, 양식, 청구서, 문서) 파싱을 위한 실용적인 의사결정 프레임워크. 핵심 인사이트: regex는 95~98%의 케이스를 저렴하고 결정론적으로 처리한다. 나머지 엣지 케이스에만 비싼 LLM 호출을 예약한다.

## 활성화 시점

- 반복 패턴이 있는 구조화된 텍스트 파싱 시 (질문, 양식, 표)
- 텍스트 추출에 regex와 LLM 중 선택해야 할 때
- 두 접근 방식을 결합한 하이브리드 파이프라인 구축 시
- 텍스트 처리에서 비용/정확도 트레이드오프 최적화 시

## 의사결정 프레임워크

```
Is the text format consistent and repeating?
├── Yes (>90% follows a pattern) → Start with Regex
│   ├── Regex handles 95%+ → Done, no LLM needed
│   └── Regex handles <95% → Add LLM for edge cases only
└── No (free-form, highly variable) → Use LLM directly
```

## 아키텍처 패턴

```
Source Text
    │
    ▼
[Regex Parser] ─── Extracts structure (95-98% accuracy)
    │
    ▼
[Text Cleaner] ─── Removes noise (markers, page numbers, artifacts)
    │
    ▼
[Confidence Scorer] ─── Flags low-confidence extractions
    │
    ├── High confidence (≥0.95) → Direct output
    │
    └── Low confidence (<0.95) → [LLM Validator] → Output
```

## 구현

### 1. Regex 파서 (대다수 처리)

```python
import re
from dataclasses import dataclass

@dataclass(frozen=True)
class ParsedItem:
    id: str
    text: str
    choices: tuple[str, ...]
    answer: str
    confidence: float = 1.0

def parse_structured_text(content: str) -> list[ParsedItem]:
    """Parse structured text using regex patterns."""
    pattern = re.compile(
        r"(?P<id>\d+)\.\s*(?P<text>.+?)\n"
        r"(?P<choices>(?:[A-D]\..+?\n)+)"
        r"Answer:\s*(?P<answer>[A-D])",
        re.MULTILINE | re.DOTALL,
    )
    items = []
    for match in pattern.finditer(content):
        choices = tuple(
            c.strip() for c in re.findall(r"[A-D]\.\s*(.+)", match.group("choices"))
        )
        items.append(ParsedItem(
            id=match.group("id"),
            text=match.group("text").strip(),
            choices=choices,
            answer=match.group("answer"),
        ))
    return items
```

### 2. 신뢰도 채점

LLM 검토가 필요한 항목에 플래그 지정:

```python
@dataclass(frozen=True)
class ConfidenceFlag:
    item_id: str
    score: float
    reasons: tuple[str, ...]

def score_confidence(item: ParsedItem) -> ConfidenceFlag:
    """Score extraction confidence and flag issues."""
    reasons = []
    score = 1.0

    if len(item.choices) < 3:
        reasons.append("few_choices")
        score -= 0.3

    if not item.answer:
        reasons.append("missing_answer")
        score -= 0.5

    if len(item.text) < 10:
        reasons.append("short_text")
        score -= 0.2

    return ConfidenceFlag(
        item_id=item.id,
        score=max(0.0, score),
        reasons=tuple(reasons),
    )

def identify_low_confidence(
    items: list[ParsedItem],
    threshold: float = 0.95,
) -> list[ConfidenceFlag]:
    """Return items below confidence threshold."""
    flags = [score_confidence(item) for item in items]
    return [f for f in flags if f.score < threshold]
```

### 3. LLM 검증기 (엣지 케이스 전용)

```python
def validate_with_llm(
    item: ParsedItem,
    original_text: str,
    client,
) -> ParsedItem:
    """Use LLM to fix low-confidence extractions."""
    response = client.messages.create(
        model="claude-haiku-4-5-20251001",  # Cheapest model for validation
        max_tokens=500,
        messages=[{
            "role": "user",
            "content": (
                f"Extract the question, choices, and answer from this text.\n\n"
                f"Text: {original_text}\n\n"
                f"Current extraction: {item}\n\n"
                f"Return corrected JSON if needed, or 'CORRECT' if accurate."
            ),
        }],
    )
    # Parse LLM response and return corrected item...
    return corrected_item
```

### 4. 하이브리드 파이프라인

```python
def process_document(
    content: str,
    *,
    llm_client=None,
    confidence_threshold: float = 0.95,
) -> list[ParsedItem]:
    """Full pipeline: regex -> confidence check -> LLM for edge cases."""
    # Step 1: Regex extraction (handles 95-98%)
    items = parse_structured_text(content)

    # Step 2: Confidence scoring
    low_confidence = identify_low_confidence(items, confidence_threshold)

    if not low_confidence or llm_client is None:
        return items

    # Step 3: LLM validation (only for flagged items)
    low_conf_ids = {f.item_id for f in low_confidence}
    result = []
    for item in items:
        if item.id in low_conf_ids:
            result.append(validate_with_llm(item, content, llm_client))
        else:
            result.append(item)

    return result
```

## 실제 지표

프로덕션 퀴즈 파싱 파이프라인(410개 항목) 기준:

| 지표 | 값 |
|--------|-------|
| Regex 성공률 | 98.0% |
| 낮은 신뢰도 항목 | 8개 (2.0%) |
| 필요한 LLM 호출 | ~5회 |
| 전체 LLM 대비 비용 절감 | ~95% |
| 테스트 커버리지 | 93% |

## 모범 사례

- **regex로 시작하라** — 불완전한 regex라도 개선할 기준선을 제공한다
- **신뢰도 채점을 사용하라** — LLM 도움이 필요한 항목을 프로그래밍 방식으로 식별한다
- **가장 저렴한 LLM을 사용하라** — 검증에는 Haiku급 모델로 충분하다
- **파싱된 항목을 절대 변경하지 마라** — 정제/검증 단계에서 새 인스턴스를 반환한다
- **파서에는 TDD가 잘 맞는다** — 먼저 알려진 패턴의 테스트를 작성하고, 그 다음 엣지 케이스를 작성한다
- **지표를 로깅하라** (regex 성공률, LLM 호출 수) — 파이프라인 상태를 추적한다

## 피해야 할 안티패턴

- regex가 95%+ 케이스를 처리할 때 모든 텍스트를 LLM으로 전송하는 것 (비싸고 느리다)
- 자유 형식이고 고도로 가변적인 텍스트에 regex 사용 (LLM이 더 낫다)
- 신뢰도 채점을 건너뛰고 regex가 "그냥 잘 동작하길" 바라는 것
- 정제/검증 단계에서 파싱된 객체를 변경하는 것
- 엣지 케이스 테스트를 건너뛰는 것 (잘못된 입력, 누락된 필드, 인코딩 문제)

## 사용 적합 케이스

- 퀴즈/시험 문제 파싱
- 양식 데이터 추출
- 청구서/영수증 처리
- 문서 구조 파싱 (헤더, 섹션, 표)
- 비용이 중요한 반복 패턴의 구조화된 텍스트

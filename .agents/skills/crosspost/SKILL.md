---
name: crosspost
description: X, LinkedIn, Threads, Bluesky에 걸친 멀티 플랫폼 콘텐츠 배포. content-engine 패턴을 사용하여 플랫폼별 콘텐츠를 적응. 동일한 콘텐츠를 여러 플랫폼에 올리지 않습니다. 사용자가 소셜 플랫폼 전반에 콘텐츠를 배포하고자 할 때 사용합니다.
origin: ECC
---

# 크로스포스트

플랫폼 네이티브 적응을 통해 여러 소셜 플랫폼에 콘텐츠를 배포합니다.

## 활성화 시점

- 사용자가 여러 플랫폼에 콘텐츠를 게시하고 싶을 때
- 소셜 미디어 전반에 공지, 출시, 업데이트를 게시할 때
- 한 플랫폼에서 다른 플랫폼으로 게시물을 재활용할 때
- 사용자가 "크로스포스트", "모든 곳에 게시", "모든 플랫폼에 공유", "배포"를 말할 때

## 핵심 규칙

1. **플랫폼 간 동일한 콘텐츠 절대 게시 금지.** 각 플랫폼은 네이티브 적응 버전을 받습니다.
2. **주요 플랫폼 먼저.** 메인 플랫폼에 먼저 게시한 후 다른 플랫폼용으로 적응합니다.
3. **플랫폼 관례 존중.** 길이 제한, 형식, 링크 처리 방식이 모두 다릅니다.
4. **게시물당 하나의 아이디어.** 원본 콘텐츠에 여러 아이디어가 있다면 여러 게시물로 분할합니다.
5. **출처 표기 중요.** 타인의 콘텐츠를 크로스포스팅하는 경우 출처를 밝힙니다.

## 플랫폼 사양

| 플랫폼 | 최대 길이 | 링크 처리 | 해시태그 | 미디어 |
|----------|-----------|---------------|----------|-------|
| X | 280자 (프리미엄은 4000) | 길이에 포함 | 최소 (1-2개) | 이미지, 영상, GIF |
| LinkedIn | 3000자 | 길이에 미포함 | 3-5개 관련 | 이미지, 영상, 문서, 캐러셀 |
| Threads | 500자 | 별도 링크 첨부 | 일반적으로 없음 | 이미지, 영상 |
| Bluesky | 300자 | facets를 통해 (리치 텍스트) | 없음 (피드 사용) | 이미지 |

## 워크플로우

### 1단계: 원본 콘텐츠 작성

핵심 아이디어로 시작합니다. 고품질 초안을 위해 `content-engine` 스킬 사용:
- 단일 핵심 메시지 파악
- 주요 플랫폼 결정 (청중이 가장 많은 곳)
- 주요 플랫폼 버전 먼저 초안 작성

### 2단계: 대상 플랫폼 파악

사용자에게 묻거나 문맥에서 결정:
- 어떤 플랫폼을 대상으로 할지
- 우선순위 순서 (주요 플랫폼이 최선의 버전을 받음)
- 플랫폼별 요구사항 (예: LinkedIn은 전문적인 톤 필요)

### 3단계: 플랫폼별 적응

각 대상 플랫폼에 맞게 콘텐츠를 변환합니다:

**X 적응:**
- 요약이 아닌 훅으로 시작
- 핵심 인사이트로 빠르게 전달
- 가능하면 본문에서 링크 제외
- 긴 콘텐츠는 스레드 형식 사용

**LinkedIn 적응:**
- 강력한 첫 줄 ("더 보기" 이전에 보이는 부분)
- 줄 바꿈이 있는 짧은 단락
- 교훈, 결과, 전문적인 시사점을 중심으로 구성
- X보다 더 명시적인 맥락 (LinkedIn 청중은 프레이밍이 필요)

**Threads 적응:**
- 대화적이고 캐주얼한 톤
- LinkedIn보다 짧고, X보다는 압축이 덜함
- 가능하면 비주얼 우선

**Bluesky 적응:**
- 직접적이고 간결하게 (300자 제한)
- 커뮤니티 지향적인 톤
- 해시태그 대신 피드/목록을 사용하여 주제 타겟팅

### 4단계: 주요 플랫폼에 게시

주요 플랫폼에 먼저 게시:
- X에는 `x-api` 스킬 사용
- 다른 플랫폼에는 플랫폼별 API 또는 도구 사용
- 크로스 참조를 위해 게시물 URL 저장

### 5단계: 보조 플랫폼에 게시

나머지 플랫폼에 적응된 버전을 게시:
- 타이밍을 분산 (동시가 아닌 30-60분 간격)
- 적절한 경우 플랫폼 간 참조 포함 ("X에 더 긴 스레드가 있습니다" 등)

## 콘텐츠 적응 예시

### 원본: 제품 출시

**X 버전:**
```
We just shipped [feature].

[One specific thing it does that's impressive]

[Link]
```

**LinkedIn 버전:**
```
Excited to share: we just launched [feature] at [Company].

Here's why it matters:

[2-3 short paragraphs with context]

[Takeaway for the audience]

[Link]
```

**Threads 버전:**
```
just shipped something cool — [feature]

[casual explanation of what it does]

link in bio
```

### 원본: 기술적 인사이트

**X 버전:**
```
TIL: [specific technical insight]

[Why it matters in one sentence]
```

**LinkedIn 버전:**
```
A pattern I've been using that's made a real difference:

[Technical insight with professional framing]

[How it applies to teams/orgs]

#relevantHashtag
```

## API 통합

### 일괄 크로스포스팅 서비스 (예시 패턴)
크로스포스팅 서비스(예: Postbridge, Buffer, 또는 커스텀 API)를 사용하는 경우, 패턴은 다음과 같습니다:

```python
import os
import requests

resp = requests.post(
    "https://api.postbridge.io/v1/posts",
    headers={"Authorization": f"Bearer {os.environ['POSTBRIDGE_API_KEY']}"},
    json={
        "platforms": ["twitter", "linkedin", "threads"],
        "content": {
            "twitter": {"text": x_version},
            "linkedin": {"text": linkedin_version},
            "threads": {"text": threads_version}
        }
    }
)
```

### 수동 게시
Postbridge 없이 각 플랫폼의 네이티브 API를 사용하여 게시:
- X: `x-api` 스킬 패턴 사용
- LinkedIn: OAuth 2.0이 포함된 LinkedIn API v2
- Threads: Threads API (Meta)
- Bluesky: AT Protocol API

## 품질 게이트

게시 전:
- [ ] 각 플랫폼 버전이 해당 플랫폼에서 자연스럽게 읽힘
- [ ] 플랫폼 간 동일한 콘텐츠 없음
- [ ] 길이 제한 준수
- [ ] 링크가 작동하고 적절하게 배치됨
- [ ] 톤이 플랫폼 관례에 맞음
- [ ] 미디어가 각 플랫폼에 맞게 크기 조정됨

## 관련 스킬

- `content-engine` — 플랫폼 네이티브 콘텐츠 생성
- `x-api` — X/Twitter API 통합

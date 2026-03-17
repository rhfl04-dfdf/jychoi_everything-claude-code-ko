---
name: deep-research
description: firecrawl 및 exa MCP를 사용한 다중 소스 deep research입니다. 웹을 검색하고, 결과를 종합하며, 소스 출처가 포함된 인용 보고서를 제공합니다. 사용자가 증거와 인용이 포함된 주제에 대한 철저한 research를 원할 때 사용하세요.
origin: ECC
---

# Deep Research

firecrawl 및 exa MCP tool을 사용하여 여러 웹 소스로부터 인용이 포함된 철저한 research 보고서를 작성합니다.

## 활성화 시점

- 사용자가 특정 주제에 대해 심도 있는 research를 요청할 때
- 경쟁 분석, 기술 평가 또는 시장 규모 산정
- 기업, 투자자 또는 기술에 대한 실사(Due diligence)
- 여러 소스의 종합이 필요한 모든 질문
- 사용자가 "research", "deep dive", "investigate" 또는 "현재 상태가 어떤지"라고 말할 때

## MCP 요구 사항

다음 중 최소 하나:
- **firecrawl** — `firecrawl_search`, `firecrawl_scrape`, `firecrawl_crawl`
- **exa** — `web_search_exa`, `web_search_advanced_exa`, `crawling_exa`

두 가지를 함께 사용하면 최상의 커버리지를 제공합니다. `~/.claude.json` 또는 `~/.codex/config.toml`에서 설정하세요.

## Workflow

### 1단계: 목표 이해

1-2개의 간단한 확인 질문을 던지세요:
- "목표가 무엇인가요 — 학습, 의사 결정, 아니면 무언가 작성을 위한 것인가요?"
- "원하는 특정 관점이나 깊이가 있나요?"

사용자가 "그냥 research해 줘"라고 말하면 적절한 기본값으로 진행하세요.

### 2단계: Research 계획

주제를 3-5개의 research 하위 질문으로 나눕니다. 예시:
- 주제: "AI가 의료에 미치는 영향"
  - 현재 의료 분야의 주요 AI 애플리케이션은 무엇인가요?
  - 어떤 임상 결과가 측정되었나요?
  - 규제 측면의 과제는 무엇인가요?
  - 이 분야를 선도하는 기업은 어디인가요?
  - 시장 규모와 성장 궤도는 어떻게 되나요?

### 3단계: 다중 소스 검색 실행

각 하위 질문에 대해 사용 가능한 MCP tool을 사용하여 검색합니다:

**firecrawl 사용 시:**
```
firecrawl_search(query: "<sub-question keywords>", limit: 8)
```

**exa 사용 시:**
```
web_search_exa(query: "<sub-question keywords>", numResults: 8)
web_search_advanced_exa(query: "<keywords>", numResults: 5, startPublishedDate: "2025-01-01")
```

**검색 전략:**
- 하위 질문당 2-3개의 서로 다른 키워드 변형을 사용하세요.
- 일반 쿼리와 뉴스 중심 쿼리를 혼합하세요.
- 총 15-30개의 고유 소스를 목표로 합니다.
- 우선순위: 학술, 공식, 평판 좋은 뉴스 > 블로그 > 포럼

### 4단계: 주요 소스 정독(Deep-Read)

가장 유망한 URL의 경우 전체 콘텐츠를 가져옵니다:

**firecrawl 사용 시:**
```
firecrawl_scrape(url: "<url>")
```

**exa 사용 시:**
```
crawling_exa(url: "<url>", tokensNum: 5000)
```

깊이 있는 분석을 위해 3-5개의 주요 소스 전체를 읽으세요. 검색 스니펫에만 의존하지 마세요.

### 5단계: 보고서 종합 및 작성

보고서 구조:

```markdown
# [Topic]: Research Report
*Generated: [date] | Sources: [N] | Confidence: [High/Medium/Low]*

## Executive Summary
[3-5 sentence overview of key findings]

## 1. [First Major Theme]
[Findings with inline citations]
- Key point ([Source Name](url))
- Supporting data ([Source Name](url))

## 2. [Second Major Theme]
...

## 3. [Third Major Theme]
...

## Key Takeaways
- [Actionable insight 1]
- [Actionable insight 2]
- [Actionable insight 3]

## Sources
1. [Title](url) — [one-line summary]
2. ...

## Methodology
Searched [N] queries across web and news. Analyzed [M] sources.
Sub-questions investigated: [list]
```

### 6단계: 전달

- **짧은 주제**: 채팅에 전체 보고서를 게시합니다.
- **긴 보고서**: 요약(executive summary) + 주요 요점(key takeaways)을 게시하고, 전체 보고서는 파일로 저장합니다.

## Subagent를 이용한 병렬 Research

광범위한 주제의 경우 Claude Code의 Task tool을 사용하여 병렬화하세요:

```
Launch 3 research agents in parallel:
1. Agent 1: Research sub-questions 1-2
2. Agent 2: Research sub-questions 3-4
3. Agent 3: Research sub-question 5 + cross-cutting themes
```

각 agent는 검색하고 소스를 읽고 결과를 반환합니다. 메인 세션은 이를 최종 보고서로 종합합니다.

## 품질 규칙

1. **모든 주장에는 소스가 필요합니다.** 근거 없는 주장은 배제하세요.
2. **교차 검증.** 단 하나의 소스만 언급한다면 검증되지 않은 것으로 표시하세요.
3. **최신성이 중요합니다.** 최근 12개월 이내의 소스를 선호하세요.
4. **공백을 인정하세요.** 하위 질문에 대해 좋은 정보를 찾을 수 없다면 그렇게 명시하세요.
5. **환각(hallucination) 금지.** 모르는 경우 "충분한 데이터를 찾을 수 없음"이라고 말하세요.
6. **사실과 추론을 분리하세요.** 추정치, 전망, 의견은 명확하게 라벨링하세요.

## 예시

```
"핵융합 에너지의 현재 상태 research"
"2026년 백엔드 서비스를 위한 Rust vs Go deep dive"
"SaaS 비즈니스 부트스트래핑을 위한 최상의 전략 research"
"현재 미국 주택 시장에 어떤 일이 일어나고 있나요?"
"AI 코드 에디터의 경쟁 환경 조사(Investigate)"
```

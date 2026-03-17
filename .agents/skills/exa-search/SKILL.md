---
name: exa-search
description: Exa MCP를 통한 뉴럴 검색으로 웹, 코드, 기업 리서치를 수행합니다. 사용자가 웹 검색, 코드 예제, 기업 정보, 인물 검색, 또는 Exa의 뉴럴 검색 엔진을 활용한 AI 기반 딥 리서치가 필요할 때 사용합니다.
origin: ECC
---

# Exa Search

Exa MCP 서버를 통해 웹 콘텐츠, 코드, 기업, 인물에 대한 뉴럴 검색을 수행합니다.

## 활성화 조건

- 사용자가 최신 웹 정보나 뉴스가 필요할 때
- 코드 예제, API 문서, 기술 레퍼런스를 검색할 때
- 기업, 경쟁사, 시장 참여자를 조사할 때
- 특정 분야의 전문가 프로필이나 인물을 찾을 때
- 개발 작업을 위한 배경 조사를 수행할 때
- 사용자가 "검색해줘", "찾아봐", "최신 정보 알려줘" 등을 말할 때

## MCP 요구사항

Exa MCP 서버가 설정되어 있어야 합니다. `~/.claude.json`에 다음을 추가하세요:

```json
"exa-web-search": {
  "command": "npx",
  "args": ["-y", "exa-mcp-server"],
  "env": { "EXA_API_KEY": "YOUR_EXA_API_KEY_HERE" }
}
```

API 키는 [exa.ai](https://exa.ai)에서 발급받으세요.

## 핵심 도구

### web_search_exa
최신 정보, 뉴스, 사실 확인을 위한 일반 웹 검색입니다.

```
web_search_exa(query: "latest AI developments 2026", numResults: 5)
```

**파라미터:**

| 파라미터 | 타입 | 기본값 | 설명 |
|-------|------|---------|-------|
| `query` | string | 필수 | 검색 쿼리 |
| `numResults` | number | 8 | 결과 수 |

### web_search_advanced_exa
도메인 및 날짜 제약 조건이 있는 필터링된 검색입니다.

```
web_search_advanced_exa(
  query: "React Server Components best practices",
  numResults: 5,
  includeDomains: ["github.com", "react.dev"],
  startPublishedDate: "2025-01-01"
)
```

**파라미터:**

| 파라미터 | 타입 | 기본값 | 설명 |
|-------|------|---------|-------|
| `query` | string | 필수 | 검색 쿼리 |
| `numResults` | number | 8 | 결과 수 |
| `includeDomains` | string[] | 없음 | 특정 도메인으로 제한 |
| `excludeDomains` | string[] | 없음 | 특정 도메인 제외 |
| `startPublishedDate` | string | 없음 | ISO 날짜 필터 (시작) |
| `endPublishedDate` | string | 없음 | ISO 날짜 필터 (종료) |

### get_code_context_exa
GitHub, Stack Overflow, 문서 사이트에서 코드 예제와 문서를 찾습니다.

```
get_code_context_exa(query: "Python asyncio patterns", tokensNum: 3000)
```

**파라미터:**

| 파라미터 | 타입 | 기본값 | 설명 |
|-------|------|---------|-------|
| `query` | string | 필수 | 코드 또는 API 검색 쿼리 |
| `tokensNum` | number | 5000 | 콘텐츠 토큰 수 (1000-50000) |

### company_research_exa
비즈니스 인텔리전스와 뉴스를 위한 기업 조사를 수행합니다.

```
company_research_exa(companyName: "Anthropic", numResults: 5)
```

**파라미터:**

| 파라미터 | 타입 | 기본값 | 설명 |
|-------|------|---------|-------|
| `companyName` | string | 필수 | 기업명 |
| `numResults` | number | 5 | 결과 수 |

### people_search_exa
전문가 프로필과 약력을 검색합니다.

```
people_search_exa(query: "AI safety researchers at Anthropic", numResults: 5)
```

### crawling_exa
URL에서 전체 페이지 콘텐츠를 추출합니다.

```
crawling_exa(url: "https://example.com/article", tokensNum: 5000)
```

**파라미터:**

| 파라미터 | 타입 | 기본값 | 설명 |
|-------|------|---------|-------|
| `url` | string | 필수 | 추출할 URL |
| `tokensNum` | number | 5000 | 콘텐츠 토큰 수 |

### deep_researcher_start / deep_researcher_check
비동기적으로 실행되는 AI 리서치 에이전트를 시작합니다.

```
# Start research
deep_researcher_start(query: "comprehensive analysis of AI code editors in 2026")

# Check status (returns results when complete)
deep_researcher_check(researchId: "<id from start>")
```

## 사용 패턴

### 빠른 조회
```
web_search_exa(query: "Node.js 22 new features", numResults: 3)
```

### 코드 리서치
```
get_code_context_exa(query: "Rust error handling patterns Result type", tokensNum: 3000)
```

### 기업 실사
```
company_research_exa(companyName: "Vercel", numResults: 5)
web_search_advanced_exa(query: "Vercel funding valuation 2026", numResults: 3)
```

### 기술 심층 분석
```
# Start async research
deep_researcher_start(query: "WebAssembly component model status and adoption")
# ... do other work ...
deep_researcher_check(researchId: "<id>")
```

## 팁

- 광범위한 쿼리에는 `web_search_exa`를, 필터링된 결과에는 `web_search_advanced_exa`를 사용하세요
- 집중적인 코드 스니펫에는 `tokensNum`을 낮게 (1000-2000), 포괄적인 컨텍스트에는 높게 (5000+) 설정하세요
- 철저한 기업 분석을 위해 `company_research_exa`와 `web_search_advanced_exa`를 결합하세요
- 검색 결과에서 찾은 특정 URL의 전체 콘텐츠를 가져오려면 `crawling_exa`를 사용하세요
- `deep_researcher_start`는 AI 합성이 유용한 포괄적인 주제에 가장 적합합니다

## 관련 Skill

- `deep-research` — firecrawl + exa를 함께 사용하는 전체 리서치 워크플로우
- `market-research` — 의사결정 프레임워크를 포함한 비즈니스 중심 리서치

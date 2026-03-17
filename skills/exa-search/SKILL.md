---
name: exa-search
description: Exa MCP를 통한 웹, 코드, 기업 리서치를 위한 신경망 검색. 사용자가 웹 검색, 코드 예제, 기업 정보, 사람 검색, 또는 Exa 신경망 검색 엔진을 활용한 AI 기반 심층 리서치가 필요할 때 사용합니다.
origin: ECC
---

# Exa 검색

Exa MCP 서버를 통한 웹 콘텐츠, 코드, 기업, 사람에 대한 신경망 검색.

## 활성화 시점

- 사용자가 최신 웹 정보나 뉴스가 필요할 때
- 코드 예제, API 문서, 기술 레퍼런스를 검색할 때
- 기업, 경쟁사, 시장 플레이어를 리서치할 때
- 특정 도메인의 전문가 프로필이나 사람을 찾을 때
- 개발 작업의 배경 리서치를 수행할 때
- 사용자가 "검색해줘", "찾아봐", "~에 대해 알려줘", "최신 정보는?"이라고 할 때

## MCP 요구사항

Exa MCP 서버가 설정되어 있어야 합니다. `~/.claude.json`에 추가:

```json
"exa-web-search": {
  "command": "npx",
  "args": ["-y", "exa-mcp-server"],
  "env": { "EXA_API_KEY": "YOUR_EXA_API_KEY_HERE" }
}
```

API 키는 [exa.ai](https://exa.ai)에서 발급받으세요.
이 저장소의 현재 Exa 설정은 여기서 노출되는 도구 표면을 문서화합니다: `web_search_exa`와 `get_code_context_exa`.
Exa 서버가 추가 도구를 노출하는 경우, 문서나 프롬프트에 의존하기 전에 정확한 이름을 확인하세요.

## 핵심 도구

### web_search_exa
최신 정보, 뉴스, 사실에 대한 일반 웹 검색.

```
web_search_exa(query: "latest AI developments 2026", numResults: 5)
```

**파라미터:**

| 파라미터 | 타입 | 기본값 | 비고 |
|-------|------|---------|-------|
| `query` | string | 필수 | 검색 쿼리 |
| `numResults` | number | 8 | 결과 수 |
| `type` | string | `auto` | 검색 모드 |
| `livecrawl` | string | `fallback` | 필요시 실시간 크롤링 선호 |
| `category` | string | 없음 | `company` 또는 `research paper` 등 선택적 집중 |

### get_code_context_exa
GitHub, Stack Overflow, 공식 문서 사이트에서 코드 예제와 문서를 검색.

```
get_code_context_exa(query: "Python asyncio patterns", tokensNum: 3000)
```

**파라미터:**

| 파라미터 | 타입 | 기본값 | 비고 |
|-------|------|---------|-------|
| `query` | string | 필수 | 코드 또는 API 검색 쿼리 |
| `tokensNum` | number | 5000 | 콘텐츠 토큰 수 (1000-50000) |

## 사용 패턴

### 빠른 조회
```
web_search_exa(query: "Node.js 22 new features", numResults: 3)
```

### 코드 리서치
```
get_code_context_exa(query: "Rust error handling patterns Result type", tokensNum: 3000)
```

### 기업 또는 사람 리서치
```
web_search_exa(query: "Vercel funding valuation 2026", numResults: 3, category: "company")
web_search_exa(query: "site:linkedin.com/in AI safety researchers Anthropic", numResults: 5)
```

### 기술 심층 분석
```
web_search_exa(query: "WebAssembly component model status and adoption", numResults: 5)
get_code_context_exa(query: "WebAssembly component model examples", tokensNum: 4000)
```

## 팁

- 최신 정보, 기업 조회, 광범위한 탐색에는 `web_search_exa` 사용
- `site:`, 따옴표 구문, `intitle:` 같은 검색 연산자를 활용해 결과를 좁힘
- 집중된 코드 스니펫에는 낮은 `tokensNum` (1000-2000), 포괄적인 컨텍스트에는 높은 값 (5000+) 사용
- 일반 웹 페이지가 아닌 API 사용법이나 코드 예제가 필요할 때는 `get_code_context_exa` 사용

## 관련 스킬

- `deep-research` — firecrawl + exa를 함께 사용하는 전체 리서치 워크플로우
- `market-research` — 의사결정 프레임워크가 포함된 비즈니스 중심 리서치

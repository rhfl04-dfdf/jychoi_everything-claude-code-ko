---
name: search-first
description: 코딩 전 조사 workflow. 커스텀 코드를 작성하기 전에 기존 tools, libraries, patterns를 검색합니다. researcher agent를 호출합니다.
origin: ECC
---

# /search-first — 코딩 전 조사하기

"구현하기 전에 기존 솔루션을 검색"하는 workflow를 체계화합니다.

## Trigger

다음과 같은 경우 이 skill을 사용하세요:
- 기존 솔루션이 있을 법한 새로운 feature를 시작할 때
- dependency 또는 integration을 추가할 때
- 사용자가 "X 기능 추가"를 요청하고 코드를 작성하려 할 때
- 새로운 utility, helper, abstraction을 만들기 전

## Workflow

```
┌─────────────────────────────────────────────┐
│  1. NEED ANALYSIS                           │
│     필요한 기능 정의                          │
│     언어/framework 제약 사항 식별            │
├─────────────────────────────────────────────┤
│  2. PARALLEL SEARCH (researcher agent)      │
│     ┌──────────┐ ┌──────────┐ ┌──────────┐  │
│     │  npm /   │ │  MCP /   │ │  GitHub / │  │
│     │  PyPI    │ │  Skills  │ │  Web      │  │
│     └──────────┘ └──────────┘ └──────────┘  │
├─────────────────────────────────────────────┤
│  3. EVALUATE                                │
│     후보군 점수화 (기능, 유지보수,             │
│     커뮤니티, docs, 라이선스, deps)          │
├─────────────────────────────────────────────┤
│  4. DECIDE                                  │
│     ┌─────────┐  ┌──────────┐  ┌─────────┐  │
│     │  Adopt  │  │  Extend  │  │  Build   │  │
│     │ as-is   │  │  /Wrap   │  │  Custom  │  │
│     └─────────┘  └──────────┘  └─────────┘  │
├─────────────────────────────────────────────┤
│  5. IMPLEMENT                               │
│     패키지 설치 / MCP 설정 /                 │
│     최소한의 커스텀 코드 작성                 │
└─────────────────────────────────────────────┘
```

## Decision Matrix

| Signal | Action |
|--------|--------|
| 정확히 일치, 잘 유지보수됨, MIT/Apache | **Adopt** — 설치 후 직접 사용 |
| 부분 일치, 좋은 토대 | **Extend** — 설치 + 얇은 wrapper 작성 |
| 여러 개의 약한 일치 항목들 | **Compose** — 2-3개의 작은 패키지 조합 |
| 적합한 항목 없음 | **Build** — 커스텀 작성하되, 조사 내용을 바탕으로 함 |

## How to Use

### Quick Mode (인라인)

utility를 작성하거나 기능을 추가하기 전에 다음 사항을 점검하세요:

0. repo에 이미 존재하나요? → 관련 modules/tests를 먼저 `rg`로 검색
1. 일반적인 문제인가요? → npm/PyPI 검색
2. 이를 위한 MCP가 있나요? → `~/.claude/settings.json` 확인 및 검색
3. 이를 위한 skill이 있나요? → `~/.claude/skills/` 확인
4. GitHub 구현체/템플릿이 있나요? → 완전히 새로운 코드를 작성하기 전에 유지보수되는 OSS를 GitHub 코드 검색으로 확인

### Full Mode (agent)

사소하지 않은 기능의 경우, researcher agent를 실행하세요:

```
Task(subagent_type="general-purpose", prompt="
  Research existing tools for: [DESCRIPTION]
  Language/framework: [LANG]
  Constraints: [ANY]

  Search: npm/PyPI, MCP servers, Claude Code skills, GitHub
  Return: Structured comparison with recommendation
")
```

## 카테고리별 검색 Shortcuts

### 개발 툴링
- Linting → `eslint`, `ruff`, `textlint`, `markdownlint`
- Formatting → `prettier`, `black`, `gofmt`
- Testing → `jest`, `pytest`, `go test`
- Pre-commit → `husky`, `lint-staged`, `pre-commit`

### AI/LLM Integration
- Claude SDK → 최신 docs를 위해 Context7 확인
- Prompt 관리 → MCP 서버 확인
- 문서 처리 → `unstructured`, `pdfplumber`, `mammoth`

### 데이터 & API
- HTTP clients → `httpx` (Python), `ky`/`got` (Node)
- Validation → `zod` (TS), `pydantic` (Python)
- Database → MCP 서버를 먼저 확인

### 콘텐츠 & 퍼블리싱
- 마크다운 처리 → `remark`, `unified`, `markdown-it`
- 이미지 최적화 → `sharp`, `imagemin`

## Integration Points

### planner agent와 함께 사용
planner는 Phase 1 (Architecture Review) 전에 researcher를 호출해야 합니다:
- Researcher가 사용 가능한 tools 식별
- Planner가 이를 구현 계획에 포함
- 계획에서 "바퀴를 재발명"하는 것을 방지

### architect agent와 함께 사용
architect는 다음 사항을 위해 researcher와 상담해야 합니다:
- 기술 스택 결정
- Integration pattern 발견
- 기존 참조 아키텍처

### iterative-retrieval skill과 함께 사용
점진적 발견을 위해 결합:
- Cycle 1: 광범위한 검색 (npm, PyPI, MCP)
- Cycle 2: 상위 후보군 상세 평가
- Cycle 3: 프로젝트 제약 사항과의 호환성 테스트

## Examples

### 예시 1: "데드 링크 체크 추가"
```
Need: 마크다운 파일의 깨진 링크 체크
Search: npm "markdown dead link checker"
Found: textlint-rule-no-dead-link (점수: 9/10)
Action: ADOPT — npm install textlint-rule-no-dead-link
Result: 커스텀 코드 0줄, 검증된 솔루션
```

### 예시 2: "HTTP client wrapper 추가"
```
Need: 재시도 및 타임아웃 처리가 포함된 회복 탄력적인 HTTP client
Search: npm "http client retry", PyPI "httpx retry"
Found: 재시도 플러그인이 있는 got (Node), 내장 재시도가 있는 httpx (Python)
Action: ADOPT — 재시도 설정과 함께 got/httpx 직접 사용
Result: 커스텀 코드 0줄, 프로덕션에서 검증된 라이브러리
```

### 예시 3: "설정 파일 linter 추가"
```
Need: schema에 대해 프로젝트 설정 파일 검증
Search: npm "config linter schema", "json schema validator cli"
Found: ajv-cli (점수: 8/10)
Action: ADOPT + EXTEND — ajv-cli 설치, 프로젝트 전용 schema 작성
Result: 패키지 1개 + schema 파일 1개, 커스텀 검증 로직 없음
```

## Anti-Patterns

- **코드부터 작성**: 존재하는지 확인하지 않고 utility 작성
- **MCP 무시**: MCP 서버가 이미 해당 기능을 제공하는지 확인하지 않음
- **과도한 커스텀**: 라이브러리를 너무 과하게 래핑하여 이점을 잃음
- **Dependency 비대화**: 작은 기능 하나를 위해 거대한 패키지 설치

---
name: configure-ecc
description: Everything Claude Code를 위한 대화형 인스톨러 — 사용자에게 사용자 레벨 또는 프로젝트 레벨 디렉토리에 skill과 rule을 선택하고 설치하도록 안내하고, 경로를 검증하며, 선택적으로 설치된 파일을 최적화합니다.
origin: ECC
---

# Everything Claude Code (ECC) 설정

Everything Claude Code 프로젝트를 위한 단계별 대화형 설치 마법사입니다. `AskUserQuestion`을 사용하여 사용자에게 skill 및 rule의 선택적 설치를 안내한 다음, 정확성을 검증하고 최적화를 제안합니다.

## 활성화 시점

- 사용자가 "configure ecc", "install ecc", "setup everything claude code" 또는 이와 유사한 말을 할 때
- 사용자가 이 프로젝트에서 skill 또는 rule을 선택적으로 설치하고 싶어할 때
- 사용자가 기존 ECC 설치를 검증하거나 수정하고 싶어할 때
- 사용자가 프로젝트에 맞게 설치된 skill 또는 rule을 최적화하고 싶어할 때

## 사전 요구 사항

이 skill은 활성화하기 전에 Claude Code가 액세스할 수 있어야 합니다. 부트스트랩하는 두 가지 방법:
1. **Plugin을 통한 방법**: `/plugin install everything-claude-code` — plugin이 이 skill을 자동으로 로드합니다.
2. **수동 방법**: 이 skill만 `~/.claude/skills/configure-ecc/SKILL.md`에 복사한 다음, "configure ecc"라고 말하여 활성화합니다.

---

## 0단계: ECC Repository 클론

설치를 시작하기 전에, 최신 ECC 소스를 `/tmp`에 클론합니다:

```bash
rm -rf /tmp/everything-claude-code
git clone https://github.com/affaan-m/everything-claude-code.git /tmp/everything-claude-code
```

이후 모든 복사 작업의 소스로 `ECC_ROOT=/tmp/everything-claude-code`를 설정합니다.

클론이 실패하면(네트워크 문제 등), `AskUserQuestion`을 사용하여 사용자에게 기존 ECC 클론의 로컬 경로를 제공하도록 요청합니다.

---

## 1단계: 설치 레벨 선택

`AskUserQuestion`을 사용하여 사용자에게 설치 위치를 묻습니다:

```
질문: "ECC 컴포넌트를 어디에 설치할까요?"
옵션:
  - "사용자 레벨 (~/.claude/)" — "모든 Claude Code 프로젝트에 적용됩니다."
  - "프로젝트 레벨 (.claude/)" — "현재 프로젝트에만 적용됩니다."
  - "둘 다" — "공통/공유 항목은 사용자 레벨에, 프로젝트별 항목은 프로젝트 레벨에 설치합니다."
```

선택 사항을 `INSTALL_LEVEL`로 저장합니다. 대상 디렉토리를 설정합니다:
- 사용자 레벨: `TARGET=~/.claude`
- 프로젝트 레벨: `TARGET=.claude` (현재 프로젝트 루트 기준 상대 경로)
- 둘 다: `TARGET_USER=~/.claude`, `TARGET_PROJECT=.claude`

대상 디렉토리가 없으면 생성합니다:
```bash
mkdir -p $TARGET/skills $TARGET/rules
```

---

## 2단계: Skill 선택 및 설치

### 2a: 범위 선택 (Core vs Niche)

**Core(새 사용자에게 권장)**를 기본값으로 합니다. — research-first workflow를 위해 `.agents/skills/*` 및 `skills/search-first/`를 복사합니다. 이 번들은 엔지니어링, eval, 검증, security, 전략적 compaction, frontend 디자인 및 Anthropic 크로스 기능 skill(article-writing, content-engine, market-research, frontend-slides)을 포함합니다.

`AskUserQuestion` (단일 선택)을 사용합니다:
```
질문: "Core skill만 설치하시겠습니까, 아니면 niche/framework 팩을 포함하시겠습니까?"
옵션:
  - "Core만 (권장)" — "tdd, e2e, evals, verification, research-first, security, frontend patterns, compacting, cross-functional Anthropic skills"
  - "Core + 선택한 niche" — "Core 이후에 프레임워크/도메인별 skill 추가"
  - "Niche만" — "Core를 건너뛰고 특정 프레임워크/도메인 skill 설치"
기본값: Core만
```

사용자가 niche 또는 core + niche를 선택하면 아래의 카테고리 선택으로 넘어가서 선택한 niche skill만 포함합니다.

### 2b: Skill 카테고리 선택

아래에 7개의 선택 가능한 카테고리 그룹이 있습니다. 뒤따르는 상세 확인 목록은 8개 카테고리에 걸친 41개의 skill과 1개의 독립 실행형 템플릿을 다룹니다. `multiSelect: true`와 함께 `AskUserQuestion`을 사용합니다:

```
질문: "어떤 skill 카테고리를 설치하시겠습니까?"
옵션:
  - "Framework & Language" — "Django, Spring Boot, Go, Python, Java, Frontend, Backend 패턴"
  - "Database" — "PostgreSQL, ClickHouse, JPA/Hibernate 패턴"
  - "Workflow & Quality" — "TDD, verification, learning, security review, compaction"
  - "Research & APIs" — "Deep research, Exa search, Claude API 패턴"
  - "Social & Content Distribution" — "X/Twitter API, content-engine과 함께 크로스 포스팅"
  - "Media Generation" — "VideoDB와 함께 fal.ai 이미지/비디오/오디오"
  - "Orchestration" — "dmux 멀티 agent workflow"
  - "모든 skill" — "사용 가능한 모든 skill 설치"
```

### 2c: 개별 Skill 확인

선택한 각 카테고리에 대해 아래의 전체 skill 목록을 출력하고 사용자에게 특정 skill을 확인하거나 선택 해제하도록 요청합니다. 목록이 4개를 초과하면 목록을 텍스트로 출력하고 "나열된 모든 항목 설치" 옵션과 사용자가 특정 이름을 붙여넣을 수 있는 "기타" 옵션이 포함된 `AskUserQuestion`을 사용합니다.

**카테고리: Framework & Language (17개 skill)**

| Skill | 설명 |
|-------|-------------|
| `backend-patterns` | Node.js/Express/Next.js를 위한 Backend architecture, API design, server-side best practices |
| `coding-standards` | TypeScript, JavaScript, React, Node.js를 위한 유니버설 coding standards |
| `django-patterns` | Django architecture, DRF를 사용한 REST API, ORM, caching, signal, middleware |
| `django-security` | Django security: auth, CSRF, SQL injection, XSS 예방 |
| `django-tdd` | pytest-django, factory_boy, mocking, coverage를 사용한 Django 테스팅 |
| `django-verification` | Django verification loop: migration, linting, test, security scan |
| `frontend-patterns` | React, Next.js, 상태 관리, 성능, UI 패턴 |
| `frontend-slides` | 의존성 없는 HTML 프레젠테이션, 스타일 미리보기 및 PPTX-to-web 변환 |
| `golang-patterns` | idiomatic Go 패턴, 견고한 Go 애플리케이션을 위한 컨벤션 |
| `golang-testing` | Go 테스팅: table-driven test, subtest, benchmark, fuzzing |
| `java-coding-standards` | Spring Boot를 위한 Java coding standards: 명명 규칙, 불변성, Optional, stream |
| `python-patterns` | Pythonic idiom, PEP 8, type hint, best practices |
| `python-testing` | pytest, TDD, fixture, mocking, parametrization을 사용한 Python 테스팅 |
| `springboot-patterns` | Spring Boot architecture, REST API, 계층형 서비스, caching, async |
| `springboot-security` | Spring Security: authn/authz, 검증, CSRF, secret, rate limiting |
| `springboot-tdd` | JUnit 5, Mockito, MockMvc, Testcontainers를 사용한 Spring Boot TDD |
| `springboot-verification` | Spring Boot verification: build, 정적 분석, test, security scan |

**카테고리: Database (3개 skill)**

| Skill | 설명 |
|-------|-------------|
| `clickhouse-io` | ClickHouse 패턴, query 최적화, analytics, data engineering |
| `jpa-patterns` | JPA/Hibernate entity 설계, 관계, query 최적화, transaction |
| `postgres-patterns` | PostgreSQL query 최적화, schema 설계, indexing, security |

**카테고리: Workflow & Quality (8개 skill)**

| Skill | 설명 |
|-------|-------------|
| `continuous-learning` | 세션에서 재사용 가능한 패턴을 학습된 skill로 자동 추출 |
| `continuous-learning-v2` | 신뢰도 점수가 포함된 instinct 기반 학습, skill/command/agent로 진화 |
| `eval-harness` | eval-driven development (EDD)를 위한 공식 evaluation framework |
| `iterative-retrieval` | subagent context 문제를 위한 점진적 context 정제 |
| `security-review` | Security 체크리스트: auth, input, secret, API, 결제 기능 |
| `strategic-compact` | 논리적 간격으로 수동 context compaction 제안 |
| `tdd-workflow` | 80% 이상의 커버리지를 가진 TDD 강제: unit, integration, E2E |
| `verification-loop` | verification 및 quality loop 패턴 |

**카테고리: Business & Content (5개 skill)**

| Skill | 설명 |
|-------|-------------|
| `article-writing` | 노트, 예시 또는 소스 문서를 사용하여 제공된 음성으로 long-form 작성 |
| `content-engine` | 멀티 플랫폼 소셜 콘텐츠, script 및 repurposing workflow |
| `market-research` | 소스가 포함된 시장, 경쟁업체, 펀드 및 기술 조사 |
| `investor-materials` | Pitch deck, one-pager, 투자자 메모 및 재무 모델 |
| `investor-outreach` | 개인화된 투자자 콜드 메일, warm intro 및 후속 조치 |

**카테고리: Research & APIs (3개 skill)**

| Skill | 설명 |
|-------|-------------|
| `deep-research` | 인용 보고서가 포함된 firecrawl 및 exa MCP를 사용하는 멀티 소스 deep research |
| `exa-search` | 웹, 코드, 회사 및 인물 조사를 위한 Exa MCP 기반 neural search |
| `claude-api` | Anthropic Claude API 패턴: Message, streaming, tool use, vision, batch, Agent SDK |

**카테고리: Social & Content Distribution (2개 skill)**

| Skill | 설명 |
|-------|-------------|
| `x-api` | 게시, thread, 검색 및 분석을 위한 X/Twitter API integration |
| `crosspost` | 플랫폼 네이티브 적응이 포함된 멀티 플랫폼 콘텐츠 배포 |

**카테고리: Media Generation (2개 skill)**

| Skill | 설명 |
|-------|-------------|
| `fal-ai-media` | fal.ai MCP를 통한 통합 AI 미디어 생성 (이미지, 비디오, 오디오) |
| `video-editing` | 실제 영상을 자르고, 구조화하고, 보강하기 위한 AI 지원 비디오 편집 |

**카테고리: Orchestration (1개 skill)**

| Skill | 설명 |
|-------|-------------|
| `dmux-workflows` | 병렬 agent 세션을 위해 dmux를 사용하는 멀티 agent orchestration |

**독립 실행형**

| Skill | 설명 |
|-------|-------------|
| `project-guidelines-example` | 프로젝트별 skill 생성을 위한 템플릿 |

### 2d: 설치 실행

선택한 각 skill에 대해 전체 skill 디렉토리를 복사합니다:
```bash
cp -r $ECC_ROOT/skills/<skill-name> $TARGET/skills/
```

참고: `continuous-learning` 및 `continuous-learning-v2`에는 추가 파일(config.json, hook, script)이 있습니다. `SKILL.md`뿐만 아니라 전체 디렉토리가 복사되었는지 확인하십시오.

---

## 3단계: Rule 선택 및 설치

`multiSelect: true`와 함께 `AskUserQuestion`을 사용합니다:

```
질문: "어떤 rule 세트를 설치하시겠습니까?"
옵션:
  - "Common rule (권장)" — "언어 중립적 원칙: coding style, git workflow, testing, security 등 (8개 파일)"
  - "TypeScript/JavaScript" — "TS/JS 패턴, hook, Playwright를 사용한 테스팅 (5개 파일)"
  - "Python" — "Python 패턴, pytest, black/ruff 포맷팅 (5개 파일)"
  - "Go" — "Go 패턴, table-driven test, gofmt/staticcheck (5개 파일)"
```

설치 실행:
```bash
# Common rule (rules/로 플랫 복사)
cp -r $ECC_ROOT/rules/common/* $TARGET/rules/

# 언어별 rule (rules/로 플랫 복사)
cp -r $ECC_ROOT/rules/typescript/* $TARGET/rules/   # 선택된 경우
cp -r $ECC_ROOT/rules/python/* $TARGET/rules/        # 선택된 경우
cp -r $ECC_ROOT/rules/golang/* $TARGET/rules/        # 선택된 경우
```

**중요**: 사용자가 언어별 rule을 선택했지만 common rule을 선택하지 않은 경우 경고를 표시합니다:
> "언어별 rule은 common rule을 확장합니다. common rule 없이 설치하면 커버리지가 불완전할 수 있습니다. common rule도 함께 설치하시겠습니까?"

---

## 4단계: 설치 후 검증

설치 후 다음 자동 검사를 수행합니다:

### 4a: 파일 존재 여부 확인

설치된 모든 파일을 나열하고 대상 위치에 존재하는지 확인합니다:
```bash
ls -la $TARGET/skills/
ls -la $TARGET/rules/
```

### 4b: 경로 참조 확인

설치된 모든 `.md` 파일에서 경로 참조를 검색합니다:
```bash
grep -rn "~/.claude/" $TARGET/skills/ $TARGET/rules/
grep -rn "../common/" $TARGET/rules/
grep -rn "skills/" $TARGET/skills/
```

**프로젝트 레벨 설치의 경우**, `~/.claude/` 경로에 대한 참조를 플래그로 표시합니다:
- skill이 `~/.claude/settings.json`을 참조하는 경우 — 이는 일반적으로 괜찮습니다 (설정은 항상 사용자 레벨임).
- skill이 `~/.claude/skills/` 또는 `~/.claude/rules/`를 참조하는 경우 — 프로젝트 레벨에만 설치된 경우 깨질 수 있습니다.
- skill이 다른 skill을 이름으로 참조하는 경우 — 참조된 skill도 설치되었는지 확인합니다.

### 4c: Skill 간 교차 참조 확인

일부 skill은 다른 skill을 참조합니다. 이러한 종속성을 검증합니다:
- `django-tdd`는 `django-patterns`를 참조할 수 있음
- `springboot-tdd`는 `springboot-patterns`를 참조할 수 있음
- `continuous-learning-v2`는 `~/.claude/homunculus/` 디렉토리를 참조함
- `python-testing`은 `python-patterns`를 참조할 수 있음
- `golang-testing`은 `golang-patterns`를 참조할 수 있음
- `crosspost`는 `content-engine` 및 `x-api`를 참조함
- `deep-research`는 `exa-search`를 참조함 (상호 보완적인 MCP 도구)
- `fal-ai-media`는 `videodb`를 참조함 (상호 보완적인 미디어 skill)
- `x-api`는 `content-engine` 및 `crosspost`를 참조함
- 언어별 rule은 `common/` 대응 항목을 참조함

### 4d: 문제 보고

발견된 각 문제에 대해 다음을 보고합니다:
1. **파일**: 문제가 있는 참조를 포함하는 파일
2. **라인**: 라인 번호
3. **문제**: 무엇이 잘못되었는지 (예: "python-patterns가 설치되지 않았는데 ~/.claude/skills/python-patterns를 참조함")
4. **권장 수정 사항**: 수행할 작업 (예: "python-patterns skill 설치" 또는 ".claude/skills/로 경로 업데이트")

---

## 5단계: 설치된 파일 최적화 (선택 사항)

`AskUserQuestion`을 사용합니다:

```
질문: "프로젝트를 위해 설치된 파일을 최적화하시겠습니까?"
옵션:
  - "Skill 최적화" — "불필요한 섹션 제거, 경로 조정, 기술 스택에 맞게 조정"
  - "Rule 최적화" — "커버리지 목표 조정, 프로젝트별 패턴 추가, 도구 설정 맞춤화"
  - "둘 다 최적화" — "설치된 모든 파일의 전체 최적화"
  - "건너뛰기" — "그대로 유지"
```

### skill을 최적화하는 경우:
1. 설치된 각 `SKILL.md`를 읽습니다.
2. 사용자에게 프로젝트의 기술 스택이 무엇인지 묻습니다(이미 알고 있지 않은 경우).
3. 각 skill에 대해 관련 없는 섹션의 삭제를 제안합니다.
4. 설치 대상(소스 repo가 아님)에 있는 `SKILL.md` 파일을 직접 수정합니다.
5. 4단계에서 발견된 경로 문제를 수정합니다.

### rule을 최적화하는 경우:
1. 설치된 각 rule `.md` 파일을 읽습니다.
2. 사용자에게 다음 선호 사항을 묻습니다:
   - 테스트 커버리지 목표 (기본값 80%)
   - 선호하는 포맷팅 도구
   - Git workflow 컨벤션
   - 보안 요구 사항
3. 설치 대상에 있는 rule 파일을 직접 수정합니다.

**중요**: 설치 대상(`$TARGET/`)에 있는 파일만 수정하고, 소스 ECC repository(`$ECC_ROOT/`)에 있는 파일은 절대로 수정하지 마십시오.

---

## 6단계: 설치 요약

`/tmp`에서 클론된 repository를 정리합니다:

```bash
rm -rf /tmp/everything-claude-code
```

그런 다음 요약 보고서를 출력합니다:

```
## ECC 설치 완료

### 설치 대상
- 레벨: [사용자 레벨 / 프로젝트 레벨 / 둘 다]
- 경로: [대상 경로]

### 설치된 Skill ([개수])
- skill-1, skill-2, skill-3, ...

### 설치된 Rule ([개수])
- common (8개 파일)
- typescript (5개 파일)
- ...

### 검증 결과
- [개수]개 문제 발견, [개수]개 수정됨
- [남은 문제 목록]

### 적용된 최적화
- [변경 사항 목록 또는 "없음"]
```

---

## 문제 해결

### "Claude Code가 skill을 인식하지 못함"
- skill 디렉토리에 `SKILL.md` 파일이 포함되어 있는지 확인합니다 (낱개의 .md 파일만 있으면 안 됨).
- 사용자 레벨: `~/.claude/skills/<skill-name>/SKILL.md`가 존재하는지 확인합니다.
- 프로젝트 레벨: `.claude/skills/<skill-name>/SKILL.md`가 존재하는지 확인합니다.

### "Rule이 작동하지 않음"
- rule은 하위 디렉토리가 아닌 플랫 파일이어야 합니다: `$TARGET/rules/coding-style.md` (올바름) vs `$TARGET/rules/common/coding-style.md` (플랫 설치 시 잘못됨).
- rule 설치 후 Claude Code를 재시작합니다.

### "프로젝트 레벨 설치 후 경로 참조 오류"
- 일부 skill은 `~/.claude/` 경로를 가정합니다. 4단계 검증을 실행하여 이를 찾아 수정하십시오.
- `continuous-learning-v2`의 경우, `~/.claude/homunculus/` 디렉토리는 항상 사용자 레벨입니다 — 이는 예상된 것이며 오류가 아닙니다.

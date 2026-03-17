---
name: prompt-optimizer
description: >-
  원시 prompt를 분석하고 의도와 누락된 부분을 파악하며, ECC components(skills/commands/agents/hooks)를 매칭하여 즉시 붙여넣을 수 있는 최적화된 prompt를 출력합니다. 자문 역할만 수행하며 작업 자체를 직접 실행하지는 않습니다.
  실행 조건: 사용자가 "optimize prompt", "improve my prompt", "how to write a prompt for", "help me prompt", "rewrite this prompt"라고 말하거나 명시적으로 prompt 품질 향상을 요청할 때. 중국어 대응 문구("优化prompt", "改进prompt", "怎么写prompt", "帮我优化这个指令")에도 반응합니다.
  실행하지 않는 경우: 사용자가 작업을 직접 실행하기를 원하거나 "just do it" / "直接做"라고 말할 때. 사용자가 "优化代码", "优化性能", "optimize performance", "optimize this code"라고 말할 때 — 이들은 prompt optimization이 아닌 refactoring/performance 작업입니다.
origin: community
metadata:
  author: YannJY02
  version: "1.0.0"
---

# Prompt Optimizer

초안 prompt를 분석하고 비평하며, ECC ecosystem components에 매칭하여 사용자가 복사해서 바로 실행할 수 있는 완성된 최적화 prompt를 출력합니다.

## 사용 시점

- 사용자가 "optimize this prompt", "improve my prompt", "rewrite this prompt"라고 말할 때
- 사용자가 "...를 위한 더 나은 prompt 작성을 도와줘"라고 말할 때
- 사용자가 "Claude Code에게 ...를 요청하는 가장 좋은 방법이 뭐야?"라고 말할 때
- 사용자가 "优化prompt", "改进prompt", "怎么写prompt", "帮我优化这个指令"라고 말할 때
- 사용자가 초안 prompt를 붙여넣고 피드백이나 개선을 요청할 때
- 사용자가 "이 작업을 어떻게 prompt로 작성해야 할지 모르겠어"라고 말할 때
- 사용자가 "...를 위해 ECC를 어떻게 사용해야 해?"라고 말할 때
- 사용자가 명시적으로 `/prompt-optimize`를 호출할 때

### 사용하지 않는 경우

- 사용자가 작업을 직접 수행하기를 원할 때 (직접 실행)
- 사용자가 "优化代码", "优化性能", "optimize this code", "optimize performance"라고 말할 때 — 이들은 prompt optimization이 아닌 refactoring 작업입니다.
- 사용자가 ECC 설정에 대해 묻는 경우 (`configure-ecc`를 대신 사용)
- 사용자가 skill 목록을 원하는 경우 (`skill-stocktake`를 대신 사용)
- 사용자가 "just do it" 또는 "直接做"라고 말할 때

## 작동 방식

**자문 전용 — 사용자의 작업을 직접 실행하지 마십시오.**

code를 작성하거나, file을 생성하거나, command를 실행하거나, 어떠한 implementation 조치도 취하지 마십시오. 유일한 결과물은 분석 결과와 최적화된 prompt뿐입니다.

만약 사용자가 "just do it", "直接做", 또는 "don't optimize, just execute"라고 말한다면, 이 skill 내부에서 implementation 모드로 전환하지 마십시오. 사용자에게 이 skill은 최적화된 prompt만 생성한다고 알리고, 실행을 원한다면 일반적인 작업 요청을 하도록 안내하십시오.

다음 6단계 파이프라인을 순차적으로 실행하십시오. 결과는 아래의 출력 형식을 사용하여 제시하십시오.

### 분석 파이프라인

### Phase 0: 프로젝트 감지

prompt를 분석하기 전에 현재 프로젝트 context를 감지합니다:

1. 작업 디렉토리에 `CLAUDE.md`가 있는지 확인 — 프로젝트 컨벤션을 위해 읽어봄
2. 프로젝트 파일에서 tech stack 감지:
   - `package.json` → Node.js / TypeScript / React / Next.js
   - `go.mod` → Go
   - `pyproject.toml` / `requirements.txt` → Python
   - `Cargo.toml` → Rust
   - `build.gradle` / `pom.xml` → Java / Kotlin / Spring Boot
   - `Package.swift` → Swift
   - `Gemfile` → Ruby
   - `composer.json` → PHP
   - `*.csproj` / `*.sln` → .NET
   - `Makefile` / `CMakeLists.txt` → C / C++
   - `cpanfile` / `Makefile.PL` → Perl
3. 감지된 tech stack을 Phase 3 및 Phase 4에서 사용하기 위해 기록

프로젝트 파일을 찾을 수 없는 경우(예: prompt가 추상적이거나 새로운 프로젝트를 위한 것인 경우), 감지를 건너뛰고 Phase 4에서 "tech stack 알 수 없음"으로 표시합니다.

### Phase 1: 의도 감지

사용자의 작업을 하나 이상의 카테고리로 분류합니다:

| 카테고리 | 신호 단어 | 예시 |
|----------|-------------|---------|
| New Feature | build, create, add, implement, 创建, 实现, 添加 | "Build a login page" |
| Bug Fix | fix, broken, not working, error, 修复, 报错 | "Fix the auth flow" |
| Refactor | refactor, clean up, restructure, 重构, 整理 | "Refactor the API layer" |
| Research | how to, what is, explore, investigate, 怎么, 如何 | "How to add SSO" |
| Testing | test, coverage, verify, 测试, 覆盖率 | "Add tests for the cart" |
| Review | review, audit, check, 审查, 检查 | "Review my PR" |
| Documentation | document, update docs, 文档 | "Update the API docs" |
| Infrastructure | deploy, CI, docker, database, 部署, 数据库 | "Set up CI/CD pipeline" |
| Design | design, architecture, plan, 设计, 架构 | "Design the data model" |

### Phase 2: 범위 평가

Phase 0에서 프로젝트를 감지한 경우, codebase 크기를 신호로 사용합니다. 그렇지 않으면 prompt 설명만으로 추정하고 해당 추정치를 불확실함으로 표시합니다.

| 범위 | 휴리스틱 | Orchestration |
|-------|-----------|---------------|
| TRIVIAL | 단일 file, < 50 라인 | 직접 실행 |
| LOW | 단일 component 또는 module | 단일 command 또는 skill |
| MEDIUM | 여러 component, 동일 도메인 | command 체인 + /verify |
| HIGH | 교차 도메인, 5개 이상의 file | /plan 먼저 수행 후 단계별 실행 |
| EPIC | 멀티 session, 멀티 PR, 아키텍처 변경 | 멀티 session 계획을 위해 blueprint skill 사용 |

### Phase 3: ECC Component 매칭

의도 + 범위 + tech stack(Phase 0에서 확인)을 특정 ECC components에 매핑합니다.

#### 의도 유형별

| 의도 | Commands | Skills | Agents |
|--------|----------|--------|--------|
| New Feature | /plan, /tdd, /code-review, /verify | tdd-workflow, verification-loop | planner, tdd-guide, code-reviewer |
| Bug Fix | /tdd, /build-fix, /verify | tdd-workflow | tdd-guide, build-error-resolver |
| Refactor | /refactor-clean, /code-review, /verify | verification-loop | refactor-cleaner, code-reviewer |
| Research | /plan | search-first, iterative-retrieval | — |
| Testing | /tdd, /e2e, /test-coverage | tdd-workflow, e2e-testing | tdd-guide, e2e-runner |
| Review | /code-review | security-review | code-reviewer, security-reviewer |
| Documentation | /update-docs, /update-codemaps | — | doc-updater |
| Infrastructure | /plan, /verify | docker-patterns, deployment-patterns, database-migrations | architect |
| Design (MEDIUM-HIGH) | /plan | — | planner, architect |
| Design (EPIC) | — | blueprint (skill로 호출) | planner, architect |

#### Tech Stack별

| Tech Stack | 추가할 Skills | Agent |
|------------|--------------|-------|
| Python / Django | django-patterns, django-tdd, django-security, django-verification, python-patterns, python-testing | python-reviewer |
| Go | golang-patterns, golang-testing | go-reviewer, go-build-resolver |
| Spring Boot / Java | springboot-patterns, springboot-tdd, springboot-security, springboot-verification, java-coding-standards, jpa-patterns | code-reviewer |
| Kotlin / Android | kotlin-coroutines-flows, compose-multiplatform-patterns, android-clean-architecture | kotlin-reviewer |
| TypeScript / React | frontend-patterns, backend-patterns, coding-standards | code-reviewer |
| Swift / iOS | swiftui-patterns, swift-concurrency-6-2, swift-actor-persistence, swift-protocol-di-testing | code-reviewer |
| PostgreSQL | postgres-patterns, database-migrations | database-reviewer |
| Perl | perl-patterns, perl-testing, perl-security | code-reviewer |
| C++ | cpp-coding-standards, cpp-testing | code-reviewer |
| 기타 / 미등록 | coding-standards (범용) | code-reviewer |

### Phase 4: 누락된 컨텍스트 감지

prompt에서 누락된 중요한 정보를 스캔합니다. 각 항목을 확인하고 Phase 0에서 자동 감지되었는지 또는 사용자가 제공해야 하는지 표시합니다:

- [ ] **Tech stack** — Phase 0에서 감지됨, 아니면 사용자가 지정해야 함?
- [ ] **Target scope** — 언급된 file, directory 또는 module이 있는가?
- [ ] **Acceptance criteria** — 작업 완료를 어떻게 알 수 있는가?
- [ ] **Error handling** — 예외 상황 및 실패 모드가 다뤄졌는가?
- [ ] **Security requirements** — 인증, 입력 검증, secrets?
- [ ] **Testing expectations** — Unit, integration, E2E?
- [ ] **Performance constraints** — 부하, 지연 시간, 리소스 제한?
- [ ] **UI/UX requirements** — 디자인 사양, 반응형, a11y? (프론트엔드인 경우)
- [ ] **Database changes** — Schema, migrations, indexes? (데이터 레이어인 경우)
- [ ] **Existing patterns** — 따를 참조 file이나 컨벤션이 있는가?
- [ ] **Scope boundaries** — 하지 말아야 할 것은 무엇인가?

**3개 이상의 중요 항목이 누락된 경우**, 최적화된 prompt를 생성하기 전에 사용자에게 최대 3개의 명확화 질문을 던지십시오. 그런 다음 답변을 최적화된 prompt에 포함시키십시오.

### Phase 5: Workflow 및 Model 추천

이 prompt가 개발 lifecycle의 어디에 위치하는지 결정합니다:

```
Research → Plan → Implement (TDD) → Review → Verify → Commit
```

MEDIUM 이상의 작업의 경우, 항상 /plan으로 시작하십시오. EPIC 작업의 경우 blueprint skill을 사용하십시오.

**Model 추천** (출력에 포함):

| 범위 | 추천 Model | 근거 |
|-------|------------------|-----------|
| TRIVIAL-LOW | Sonnet 4.6 | 단순 작업에 대해 빠르고 비용 효율적임 |
| MEDIUM | Sonnet 4.6 | 표준 작업에 가장 적합한 코딩 모델 |
| HIGH | Sonnet 4.6 (메인) + Opus 4.6 (기획) | 아키텍처에는 Opus, 구현에는 Sonnet |
| EPIC | Opus 4.6 (blueprint) + Sonnet 4.6 (실행) | 멀티 session 계획을 위한 깊은 추론 능력 |

**Multi-prompt 분할** (HIGH/EPIC 범위용):

단일 session을 초과하는 작업의 경우 순차적 prompt로 나눕니다:
- Prompt 1: Research + Plan (search-first skill 사용 후 /plan)
- Prompt 2-N: prompt당 한 단계씩 구현 (각각 /verify로 종료)
- Final Prompt: 통합 테스트 + 모든 단계에 대한 /code-review
- session 간 context를 유지하기 위해 /save-session 및 /resume-session 사용

---

## 출력 형식

분석 내용을 정확히 이 구조에 맞게 제시하십시오. 사용자가 입력한 언어와 동일한 언어로 응답하십시오.

### Section 1: Prompt 진단

**강점:** 원래 prompt가 잘하고 있는 점을 나열합니다.

**문제점:**

| 문제점 | 영향 | 제안된 수정 사항 |
|-------|--------|---------------|
| (문제) | (결과) | (수정 방법) |

**확인 필요 사항:** 사용자가 답변해야 할 질문의 번호가 매겨진 목록입니다. Phase 0에서 답변을 자동 감지했다면 질문 대신 감지된 내용을 기술하십시오.

### Section 2: 추천 ECC Components

| 유형 | Component | 목적 |
|------|-----------|---------|
| Command | /plan | 코딩 전 아키텍처 계획 |
| Skill | tdd-workflow | TDD 방법론 가이드 |
| Agent | code-reviewer | 구현 후 리뷰 수행 |
| Model | Sonnet 4.6 | 이 범위에 권장됨 |

### Section 3: 최적화된 Prompt — 전체 버전

최적화된 전체 prompt를 단일 fenced code block 안에 제시하십시오. prompt는 자체 포함되어 있어야 하며 복사해서 붙여넣을 준비가 되어 있어야 합니다. 다음을 포함하십시오:
- context가 포함된 명확한 작업 설명
- Tech stack (감지되었거나 지정된 것)
- 적절한 workflow 단계에서의 /command 호출
- Acceptance criteria (수락 기준)
- Verification (검증) 단계
- Scope boundaries (하지 말아야 할 것)

blueprint를 참조하는 항목의 경우, "Use the blueprint skill to..."라고 작성하십시오. (`/blueprint`가 아님. blueprint는 command가 아니라 skill이므로).

### Section 4: 최적화된 Prompt — 요약 버전

숙련된 ECC 사용자를 위한 요약 버전입니다. 의도 유형에 따라 다르게 구성하십시오:

| 의도 | 요약 패턴 |
|--------|--------------|
| New Feature | `/plan [feature]. /tdd to implement. /code-review. /verify.` |
| Bug Fix | `/tdd — write failing test for [bug]. Fix to green. /verify.` |
| Refactor | `/refactor-clean [scope]. /code-review. /verify.` |
| Research | `Use search-first skill for [topic]. /plan based on findings.` |
| Testing | `/tdd [module]. /e2e for critical flows. /test-coverage.` |
| Review | `/code-review. Then use security-reviewer agent.` |
| Docs | `/update-docs. /update-codemaps.` |
| EPIC | `Use blueprint skill for "[objective]". Execute phases with /verify gates.` |

### Section 5: 개선 근거

| 개선 사항 | 이유 |
|-------------|--------|
| (추가된 내용) | (중요한 이유) |

### 푸터

> 원하는 결과가 아니신가요? 조정이 필요한 부분을 말씀해 주시거나, prompt optimization 대신 실행을 원하신다면 일반적인 작업 요청을 해주세요.

---

## 예시

### 실행 트리거 예시

- "Optimize this prompt for ECC"
- "Rewrite this prompt so Claude Code uses the right commands"
- "帮我优化这个指令"
- "How should I prompt ECC for this task?"

### 예시 1: 모호한 중국어 Prompt (프로젝트 감지됨)

**사용자 입력:**
```
帮我写一个用户登录页面
```

**Phase 0 감지 내용:** Next.js 15, TypeScript, Tailwind CSS가 포함된 `package.json`

**최적화된 Prompt (전체):**
```
사용 중인 프로젝트의 기존 tech stack(Next.js 15 + TypeScript + Tailwind CSS)을 사용하여 사용자 로그인 페이지를 구현하십시오.

기술 요구 사항:
- 프로젝트의 기존 component 구조 및 routing 규칙을 준수할 것
- form 검증은 프로젝트의 기존 검증 솔루션(Zod/Yup/기타 사용 여부 확인)을 따를 것
- 인증 방식: 프로젝트의 기존 인증 솔루션을 따를 것 (없는 경우 기본값은 JWT)
- 포함 사항: 이메일/비밀번호 로그인 form, form 검증, 오류 메시지, 로딩 상태, 반응형 레이아웃

Workflow:
1. /plan: 먼저 component 구조와 인증 flow를 계획하고, 기존 페이지 패턴을 참고하십시오.
2. /tdd: 테스트 우선 - 로그인 form의 unit test와 인증 flow의 integration test를 작성하십시오.
3. 로그인 페이지 및 인증 logic을 구현하십시오.
4. /code-review: 구현 내용을 리뷰하십시오.
5. /verify: 모든 테스트가 통과하고 페이지가 정상적으로 렌더링되는지 확인하십시오.

보안 요구 사항:
- 비밀번호를 평문으로 전송하지 말 것
- 무차별 대입 공격 방지(rate limiting)
- XSS 방어
- CSRF token

Acceptance criteria:
- 모든 테스트 통과, 커버리지 80% 이상
- 모바일 및 데스크탑에서 페이지가 정상적으로 렌더링됨
- 로그인 성공 시 dashboard로 이동, 실패 시 오류 메시지 표시

하지 말아야 할 것:
- 회원가입 페이지 구현 금지
- 비밀번호 찾기 기능 구현 금지
- 기존 routing 구조 수정 금지
```

### 예시 2: 중간 규모의 영어 Prompt

**사용자 입력:**
```
Add a REST API endpoint for user profile updates with validation
```

**Phase 0 감지 내용:** Go 1.22, Chi router가 포함된 `go.mod`

**최적화된 Prompt (전체):**
```
사용자 프로필 업데이트를 위한 REST API endpoint(PATCH /api/users/:id)를 추가하십시오.

Tech stack: Go 1.22 + Chi router (프로젝트에서 감지됨)

요구 사항:
- PATCH /api/users/:id — 사용자 프로필의 부분 업데이트
- 필드 입력 검증: name, email, avatar_url, bio
- 인증: 유효한 token 필요, 사용자는 자신의 프로필만 업데이트 가능
- 성공 시 업데이트된 사용자 정보와 함께 200 반환
- 입력이 유효하지 않은 경우 검증 오류와 함께 400 반환
- 인증 실패 시 401/403 반환
- codebase의 기존 API 패턴을 따를 것

Workflow:
1. endpoint 구조, middleware 체인, 검증 logic을 /plan 하십시오.
2. /tdd — 성공, 검증 실패, 인증 실패, 찾을 수 없음(not-found)에 대한 table-driven test를 작성하십시오.
3. 기존 handler 패턴을 따라 구현하십시오.
4. /go-review를 실행하십시오.
5. /verify — 전체 테스트 스위트를 실행하고 regression이 없는지 확인하십시오.

하지 말아야 할 것:
- 기존 endpoint 수정 금지
- 데이터베이스 schema 변경 금지 (기존 user 테이블 사용)
- 기존 dependency를 먼저 확인하지 않고 새로운 것을 추가하지 말 것 (search-first skill 사용)
```

### 예시 3: EPIC 프로젝트

**사용자 입력:**
```
Migrate our monolith to microservices
```

**최적화된 Prompt (전체):**
```
blueprint skill을 사용하여 다음을 계획하십시오: "Migrate monolith to microservices architecture"

실행하기 전에 blueprint에서 다음 질문에 답하십시오:
1. 현재 monolith에 어떤 도메인 경계가 존재하는가?
2. 어떤 서비스를 먼저 추출해야 하는가 (결합도가 가장 낮은 것)?
3. 통신 패턴: REST API, gRPC, 또는 event-driven (Kafka/RabbitMQ)?
4. 데이터베이스 전략: 초기에는 DB 공유 또는 처음부터 서비스별 DB 분리?
5. 배포 대상: Kubernetes, Docker Compose, 또는 serverless?

blueprint는 다음과 같은 단계(phase)를 생성해야 합니다:
- Phase 1: 서비스 경계 식별 및 도메인 맵 생성
- Phase 2: 인프라 설정 (API gateway, service mesh, 서비스별 CI/CD)
- Phase 3: 첫 번째 서비스 추출 (strangler fig 패턴)
- Phase 4: 통합 테스트로 검증 후 다음 서비스 추출
- Phase N: monolith 폐기

각 단계 = 1 PR, 단계 사이에 /verify 게이트를 둘 것.
단계 사이에 /save-session을 사용하고, 계속하려면 /resume-session을 사용하십시오.
의존성이 허용하는 경우 병렬 서비스 추출을 위해 git worktrees를 사용하십시오.

권장 사항: blueprint 계획에는 Opus 4.6, 단계별 실행에는 Sonnet 4.6 사용.
```

---

## 관련 Components

| Component | 참조 시점 |
|-----------|------------------|
| `configure-ecc` | 사용자가 아직 ECC를 설정하지 않았을 때 |
| `skill-stocktake` | 어떤 component가 설치되어 있는지 확인할 때 (하드코딩된 목록 대신 사용) |
| `search-first` | 최적화된 prompt의 Research 단계에서 |
| `blueprint` | EPIC 범위의 최적화된 prompt에서 (command가 아닌 skill로 호출) |
| `strategic-compact` | 긴 session context 관리 시 |
| `cost-aware-llm-pipeline` | Token 최적화 권장 사항 확인 시 |

# Everything Claude Code 롱폼 가이드

![헤더: Everything Claude Code 롱폼 가이드](./assets/images/longform/01-header.png)

---

> **사전 조건**: 이 가이드는 [Everything Claude Code 단축 가이드](./the-shortform-guide.md)를 기반으로 합니다. skill, hook, subagent, MCP, 플러그인을 아직 설정하지 않았다면 먼저 읽어보세요.

![단축 가이드 참조](./assets/images/longform/02-shortform-reference.png)
*단축 가이드 - 먼저 읽어보세요*

단축 가이드에서는 기초 설정을 다뤘습니다: skill과 command, hook, subagent, MCP, 플러그인, 그리고 효과적인 Claude Code 워크플로우의 기반을 이루는 설정 패턴들. 그것은 설정 가이드이자 기본 인프라였습니다.

이 롱폼 가이드는 생산적인 세션과 낭비적인 세션을 구분하는 기술들을 다룹니다. 단축 가이드를 읽지 않았다면, 먼저 돌아가서 설정을 완료하세요. 이하 내용은 skill, agent, hook, MCP가 이미 설정되고 작동 중임을 전제합니다.

여기서 다루는 주제들: 토큰 경제학, 메모리 지속성, 검증 패턴, 병렬화 전략, 그리고 재사용 가능한 워크플로우 구축의 복합 효과. 이것들은 10개월 이상의 매일 사용을 통해 다듬어진 패턴으로, 첫 시간 안에 컨텍스트 부패에 시달리는 것과 시간 동안 생산적인 세션을 유지하는 것의 차이를 만들어냅니다.

단축 가이드와 롱폼 가이드에서 다루는 모든 내용은 GitHub에서 확인할 수 있습니다: `github.com/affaan-m/everything-claude-code`

---

## 팁과 트릭

### 일부 MCP는 대체 가능하며 컨텍스트 윈도우를 확보해 줍니다

버전 관리(GitHub), 데이터베이스(Supabase), 배포(Vercel, Railway) 등의 MCP - 이런 플랫폼들은 대부분 이미 강력한 CLI를 갖추고 있으며 MCP는 본질적으로 그것을 감싸는 래퍼일 뿐입니다. MCP는 좋은 래퍼지만 비용이 따릅니다.

실제로 MCP를 사용하지 않고도(그리고 그에 따른 컨텍스트 윈도우 감소 없이) CLI가 MCP처럼 기능하도록 하려면, 기능을 skill과 command로 묶는 것을 고려하세요. MCP가 노출하는 도구들을 제거하고 그것들을 command로 전환하세요.

예시: GitHub MCP를 항상 로드하는 대신, 선호하는 옵션으로 `gh pr create`를 감싸는 `/gh-pr` command를 만드세요. Supabase MCP가 컨텍스트를 소모하는 대신, Supabase CLI를 직접 사용하는 skill을 만드세요.

레이지 로딩을 사용하면 컨텍스트 윈도우 문제는 대부분 해결됩니다. 하지만 토큰 사용량과 비용은 같은 방식으로 해결되지 않습니다. CLI + skill 접근법은 여전히 토큰 최적화 방법입니다.

---

## 중요 사항

### 컨텍스트 및 메모리 관리

세션 간 메모리를 공유하려면, 진행 상황을 요약하고 확인한 다음 `.claude` 폴더의 `.tmp` 파일에 저장하고 세션이 끝날 때까지 이어서 추가하는 skill이나 command를 사용하는 것이 가장 좋습니다. 다음 날 그것을 컨텍스트로 사용하여 이어서 작업할 수 있으며, 각 세션마다 새 파일을 만들어 이전 컨텍스트가 새 작업에 오염되지 않도록 하세요.

![세션 저장 파일 트리](./assets/images/longform/03-session-storage.png)
*세션 저장 예시 -> <https://github.com/affaan-m/everything-claude-code/tree/main/examples/sessions>*

Claude는 현재 상태를 요약하는 파일을 만듭니다. 검토하고, 필요하면 수정을 요청한 후, 새로 시작하세요. 새 대화에서는 파일 경로만 제공하면 됩니다. 컨텍스트 한계에 부딪혀 복잡한 작업을 계속해야 할 때 특히 유용합니다. 이 파일에는 다음이 포함되어야 합니다:
- 효과가 있었던 접근 방법 (증거로 검증된)
- 시도했지만 효과가 없었던 접근 방법
- 시도하지 않은 접근 방법과 남은 작업

**컨텍스트 전략적 비우기:**

계획이 수립되고 컨텍스트가 비워지면(현재 Claude Code의 계획 모드의 기본 옵션), 계획에서 작업할 수 있습니다. 이는 실행과 더 이상 관련 없는 탐색 컨텍스트가 많이 쌓였을 때 유용합니다. 전략적 압축을 위해 자동 압축을 비활성화하세요. 논리적 간격에서 수동으로 압축하거나, 자동으로 압축하는 skill을 만드세요.

**고급: 동적 시스템 프롬프트 주입**

제가 파악한 패턴 중 하나: 매 세션마다 로드되는 CLAUDE.md(사용자 범위) 또는 `.claude/rules/`(프로젝트 범위)에 모든 것을 넣는 것만으로 만족하지 말고, CLI 플래그를 사용하여 동적으로 컨텍스트를 주입하세요.

```bash
claude --system-prompt "$(cat memory.md)"
```

이를 통해 언제 어떤 컨텍스트가 로드될지 더 정확하게 제어할 수 있습니다. 시스템 프롬프트 내용은 사용자 메시지보다 더 높은 권한을 가지며, 사용자 메시지는 도구 결과보다 더 높은 권한을 가집니다.

**실용적인 설정:**

```bash
# Daily development
alias claude-dev='claude --system-prompt "$(cat ~/.claude/contexts/dev.md)"'

# PR review mode
alias claude-review='claude --system-prompt "$(cat ~/.claude/contexts/review.md)"'

# Research/exploration mode
alias claude-research='claude --system-prompt "$(cat ~/.claude/contexts/research.md)"'
```

**고급: 메모리 지속성 Hook**

대부분의 사람들이 모르는 메모리에 도움이 되는 hook들이 있습니다:

- **PreCompact Hook**: 컨텍스트 압축 전에 중요한 상태를 파일에 저장
- **Stop Hook (세션 종료)**: 세션 종료 시 학습 내용을 파일에 저장
- **SessionStart Hook**: 새 세션 시작 시 이전 컨텍스트 자동 로드

이 hook들을 만들었으며 `github.com/affaan-m/everything-claude-code/tree/main/hooks/memory-persistence`의 저장소에 있습니다.

---

### 지속적 학습 / 메모리

프롬프트를 여러 번 반복해야 했고 Claude가 같은 문제에 부딪히거나 이전에 들어본 응답을 받았다면 - 그 패턴들을 skill에 추가해야 합니다.

**문제점:** 낭비된 토큰, 낭비된 컨텍스트, 낭비된 시간.

**해결책:** Claude Code가 사소하지 않은 것을 발견할 때 - 디버깅 기술, 해결 방법, 프로젝트별 패턴 - 그 지식을 새로운 skill로 저장합니다. 다음에 유사한 문제가 발생하면 skill이 자동으로 로드됩니다.

이를 위한 지속적 학습 skill을 만들었습니다: `github.com/affaan-m/everything-claude-code/tree/main/skills/continuous-learning`

**Stop Hook을 사용하는 이유 (UserPromptSubmit 대신):**

핵심 설계 결정은 UserPromptSubmit 대신 **Stop hook**을 사용하는 것입니다. UserPromptSubmit은 모든 단일 메시지에서 실행 - 모든 프롬프트에 지연을 추가합니다. Stop은 세션 종료 시 한 번만 실행 - 가볍고 세션 중에 속도를 늦추지 않습니다.

---

### 토큰 최적화

**주요 전략: Subagent 아키텍처**

사용하는 도구와 작업에 충분한 가장 저렴한 모델에 위임하도록 설계된 subagent 아키텍처를 최적화하세요.

**모델 선택 빠른 참조:**

![모델 선택 테이블](./assets/images/longform/04-model-selection.png)
*다양한 공통 작업에서 subagent의 가상 설정과 선택의 이유*

| 작업 유형 | 모델 | 이유 |
| ------------------------- | ------ | ------------------------------------------ |
| 탐색/검색 | Haiku | 빠르고, 저렴하며, 파일 찾기에 충분 |
| 단순 편집 | Haiku | 단일 파일 변경, 명확한 지침 |
| 멀티 파일 구현 | Sonnet | 코딩을 위한 최상의 균형 |
| 복잡한 아키텍처 | Opus | 깊은 추론 필요 |
| PR 리뷰 | Sonnet | 컨텍스트 이해, 뉘앙스 포착 |
| 보안 분석 | Opus | 취약점 놓칠 여유 없음 |
| 문서 작성 | Haiku | 구조가 단순 |
| 복잡한 버그 디버깅 | Opus | 전체 시스템을 머릿속에 유지해야 함 |

대부분의 코딩 작업에는 Sonnet을 기본으로 사용하세요. 첫 번째 시도 실패, 5개 이상의 파일 범위, 아키텍처 결정, 보안 핵심 코드의 경우 Opus로 업그레이드하세요.

**가격 참조:**

![Claude 모델 가격](./assets/images/longform/05-pricing-table.png)
*출처: <https://platform.claude.com/docs/en/about-claude/pricing>*

**도구별 최적화:**

mgrep으로 grep 교체 - 전통적인 grep이나 ripgrep에 비해 평균 ~50% 토큰 감소:

![mgrep 벤치마크](./assets/images/longform/06-mgrep-benchmark.png)
*50개 작업 벤치마크에서 mgrep + Claude Code는 비슷하거나 더 나은 품질로 grep 기반 워크플로우보다 약 2배 적은 토큰 사용. 출처: @mixedbread-ai의 mgrep*

**모듈식 코드베이스의 이점:**

수천 줄이 아닌 수백 줄의 메인 파일로 더 모듈식 코드베이스를 구성하면 토큰 최적화 비용과 첫 번째 시도에서 작업을 올바르게 수행하는 데 모두 도움이 됩니다.

---

### 검증 루프 및 평가

**벤치마킹 워크플로우:**

skill이 있는 경우와 없는 경우 동일한 요청을 비교하고 출력 차이를 확인:

대화를 포크하고, 하나에서 skill 없이 새 worktree를 시작한 후, 마지막에 diff를 꺼내어 로그된 것을 확인하세요.

**평가 패턴 유형:**

- **체크포인트 기반 평가**: 명시적 체크포인트 설정, 정의된 기준에 대한 검증, 진행 전 수정
- **지속적 평가**: N분마다 또는 주요 변경 후 실행, 전체 테스트 스위트 + 린트

**핵심 지표:**

```
pass@k: At least ONE of k attempts succeeds
        k=1: 70%  k=3: 91%  k=5: 97%

pass^k: ALL k attempts must succeed
        k=1: 70%  k=3: 34%  k=5: 17%
```

단순히 작동해야 할 때는 **pass@k** 사용. 일관성이 필수적일 때는 **pass^k** 사용.

---

## 병렬화

멀티 Claude 터미널 설정에서 대화를 포크할 때, 포크와 원본 대화에서의 작업 범위가 잘 정의되어 있는지 확인하세요. 코드 변경에 있어 최소한의 중복을 목표로 하세요.

**선호하는 패턴:**

코드 변경은 메인 채팅에서, 포크는 코드베이스와 현재 상태에 대한 질문이나 외부 서비스 연구에 사용.

**임의적인 터미널 수에 대하여:**

![Boris의 병렬 터미널에 대한 의견](./assets/images/longform/07-boris-parallel.png)
*Anthropic의 Boris가 여러 Claude 인스턴스 실행에 대해*

Boris는 병렬화에 대한 팁이 있습니다. 그는 로컬에서 5개, 업스트림에서 5개 Claude 인스턴스를 실행하는 것을 제안했습니다. 임의적인 터미널 수를 설정하지 않도록 권합니다. 터미널 추가는 진정한 필요에 의한 것이어야 합니다.

목표는: **최소한의 병렬화로 얼마나 많은 것을 완료할 수 있는가.**

**병렬 인스턴스를 위한 Git Worktree:**

```bash
# Create worktrees for parallel work
git worktree add ../project-feature-a feature-a
git worktree add ../project-feature-b feature-b
git worktree add ../project-refactor refactor-branch

# Each worktree gets its own Claude instance
cd ../project-feature-a && claude
```

인스턴스를 확장하기 시작하고 서로 겹치는 코드에서 작업하는 여러 Claude 인스턴스가 있다면, git worktree를 사용하고 각각에 대해 잘 정의된 계획을 갖는 것이 필수입니다. `/rename <이름>` 을 사용하여 모든 채팅 이름을 지정하세요.

![두 터미널 설정](./assets/images/longform/08-two-terminals.png)
*시작 설정: 코딩을 위한 왼쪽 터미널, 질문을 위한 오른쪽 터미널 - /rename과 /fork 사용*

**Cascade 방법:**

여러 Claude Code 인스턴스를 실행할 때 "cascade" 패턴으로 구성하세요:

- 오른쪽으로 새 탭에서 새 작업 열기
- 왼쪽에서 오른쪽으로, 가장 오래된 것부터 최신 것으로 처리
- 한 번에 최대 3-4개 작업에 집중

---

## 기반 작업

**두 인스턴스 시작 패턴:**

워크플로우 관리를 위해 2개의 열린 Claude 인스턴스로 빈 저장소를 시작하는 것을 좋아합니다.

**인스턴스 1: 스캐폴딩 Agent**
- 스캐폴드와 기반 작업 구성
- 프로젝트 구조 생성
- 설정 구성 (CLAUDE.md, rules, agents)

**인스턴스 2: 심층 연구 Agent**
- 모든 서비스, 웹 검색에 연결
- 상세 PRD 생성
- 아키텍처 mermaid 다이어그램 생성
- 실제 문서 클립으로 참조 자료 편집

**llms.txt 패턴:**

사용 가능한 경우, 문서 페이지에 도달하면 `/llms.txt`를 추가하여 많은 문서 참조에서 `llms.txt`를 찾을 수 있습니다. 이를 통해 LLM에 최적화된 깔끔한 문서 버전을 얻을 수 있습니다.

**철학: 재사용 가능한 패턴 구축**

@omarsar0로부터: "초기에 재사용 가능한 워크플로우/패턴을 구축하는 데 시간을 투자했습니다. 구축하기 번거롭지만, 모델과 agent harness가 개선됨에 따라 엄청난 복합 효과가 있었습니다."

**투자할 것:**

- Subagent
- Skill
- Command
- 계획 패턴
- MCP 도구
- 컨텍스트 엔지니어링 패턴

---

## Agent 및 Sub-Agent 모범 사례

**Sub-Agent 컨텍스트 문제:**

Sub-agent는 모든 것을 덤프하는 대신 요약을 반환하여 컨텍스트를 절약하기 위해 존재합니다. 하지만 오케스트레이터는 sub-agent가 부족한 의미론적 컨텍스트를 가지고 있습니다. Sub-agent는 요청 뒤의 목적이 아닌 리터럴 쿼리만 알고 있습니다.

**반복적 검색 패턴:**

1. 오케스트레이터가 모든 sub-agent 반환을 평가
2. 수락하기 전에 후속 질문
3. Sub-agent가 소스로 돌아가 답변을 얻어 반환
4. 충분할 때까지 루프 (최대 3 사이클)

**핵심:** 쿼리가 아닌 목표 컨텍스트 전달.

**순차적 단계를 가진 오케스트레이터:**

```markdown
Phase 1: RESEARCH (use Explore agent) → research-summary.md
Phase 2: PLAN (use planner agent) → plan.md
Phase 3: IMPLEMENT (use tdd-guide agent) → code changes
Phase 4: REVIEW (use code-reviewer agent) → review-comments.md
Phase 5: VERIFY (use build-error-resolver if needed) → done or loop back
```

**핵심 규칙:**

1. 각 agent는 하나의 명확한 입력을 받고 하나의 명확한 출력을 생성
2. 출력이 다음 단계의 입력이 됨
3. 단계 건너뛰기 금지
4. Agent 간에 `/clear` 사용
5. 중간 출력을 파일에 저장

---

## 재미있는 것들 / 필수는 아닌 재미있는 팁

### 사용자 지정 상태 표시줄

`/statusline`을 사용하여 설정할 수 있습니다 - 그러면 Claude가 없다고 하지만 설정해 줄 수 있다고 하며 원하는 것을 물어볼 것입니다.

참고: ccstatusline (사용자 지정 Claude Code 상태 표시줄을 위한 커뮤니티 프로젝트)

### 음성 변환

목소리로 Claude Code에 말하세요. 많은 사람들에게 타이핑보다 빠릅니다.

- Mac에서 superwhisper, MacWhisper
- 변환 오류가 있어도 Claude는 의도를 이해

### 터미널 별칭

```bash
alias c='claude'
alias gb='github'
alias co='code'
alias q='cd ~/Desktop/projects'
```

---

## 마일스톤

![25k+ GitHub 스타](./assets/images/longform/09-25k-stars.png)
*일주일 이내에 GitHub 스타 25,000개 이상 달성*

---

## 리소스

**Agent 오케스트레이션:**

- claude-flow — 54개 이상의 전문 agent를 갖춘 커뮤니티 빌드 엔터프라이즈 오케스트레이션 플랫폼

**자기 개선 메모리:**

- 이 저장소의 `skills/continuous-learning/` 참조
- rlancemartin.github.io/2025/12/01/claude_diary/ - 세션 반성 패턴

**시스템 프롬프트 참조:**

- system-prompts-and-models-of-ai-tools — AI 시스템 프롬프트 커뮤니티 컬렉션 (110k+ 스타)

**공식:**

- Anthropic Academy: anthropic.skilljar.com

---

## 참조

- [Anthropic: AI agent를 위한 평가 이해하기](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)
- [YK: 32가지 Claude Code 팁](https://agenticcoding.substack.com/p/32-claude-code-tips-from-basics-to)
- [RLanceMartin: 세션 반성 패턴](https://rlancemartin.github.io/2025/12/01/claude_diary/)
- @PerceptualPeak: Sub-Agent 컨텍스트 협상
- @menhguin: Agent 추상화 티어리스트
- @omarsar0: 복합 효과 철학

---

*단축 가이드와 롱폼 가이드에서 다루는 모든 내용은 GitHub의 [everything-claude-code](https://github.com/affaan-m/everything-claude-code)에서 확인할 수 있습니다*

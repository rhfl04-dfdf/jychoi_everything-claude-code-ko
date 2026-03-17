---
name: blueprint
description: >-
  한 줄의 목표를 다중 세션 및 다중 agent 엔지니어링 프로젝트를 위한 단계별 구축 계획으로 변환합니다. 각 단계에는 독립적인 context 요약이 포함되어 있어 새로운 agent가 사전 지식 없이도 실행할 수 있습니다. adversarial review 게이트, 의존성 그래프, 병렬 단계 감지, 안티 패턴 카탈로그 및 계획 변경 프로토콜이 포함됩니다. 트리거 조건: 사용자가 복잡한 다중 PR 작업에 대한 계획, blueprint 또는 로드맵을 요청하거나 여러 세션이 필요한 작업을 설명할 때. 트리거하지 않는 조건: 작업이 단일 PR 또는 3개 미만의 tool 호출로 완료 가능하거나 사용자가 "그냥 해(just do it)"라고 말할 때.
origin: community
---

# Blueprint — 구축 계획 생성기

한 줄의 목표를 어떤 코딩 agent라도 사전 지식 없이 실행할 수 있는 단계별 구축 계획으로 변환합니다.

## 사용 시기

- 명확한 의존성 순서에 따라 큰 기능을 여러 PR로 나눌 때
- 여러 세션에 걸친 refactor 또는 마이그레이션을 계획할 때
- 서브 agent 간의 병렬 작업 흐름을 조정할 때
- 세션 간의 context 손실로 인해 재작업이 발생할 수 있는 모든 작업

단일 PR, 3개 미만의 tool 호출로 완료 가능한 작업 또는 사용자가 "그냥 해(just do it)"라고 말하는 경우에는 **사용하지 마세요**.

## 작동 방식

Blueprint는 5단계 파이프라인을 실행합니다:

1. **Research** — 사전 점검(git, gh auth, remote, default branch) 후 프로젝트 구조, 기존 계획 및 메모리 파일을 읽어 context를 수집합니다.
2. **Design** — 목표를 PR 규모의 단계(일반적으로 3~12개)로 나눕니다. 각 단계별로 의존성 관계, 병렬/직렬 순서, 모델 티어(strongest vs default) 및 롤백 전략을 할당합니다.
3. **Draft** — 독립적인 마크다운 계획 파일을 `plans/`에 작성합니다. 모든 단계에는 context 요약, 작업 목록, 검증 명령 및 종료 기준이 포함되어 있어, 새로운 agent가 이전 단계를 읽지 않고도 모든 단계를 실행할 수 있습니다.
4. **Review** — 체크리스트 및 안티 패턴 카탈로그를 기반으로 adversarial review를 strongest-model 서브 agent(예: Opus)에게 위임합니다. 확정하기 전에 모든 치명적인 발견 사항을 수정합니다.
5. **Register** — 계획을 저장하고 메모리 인덱스를 업데이트하며, 사용자에게 단계 수와 병렬 처리 요약을 제시합니다.

Blueprint는 git/gh 가용성을 자동으로 감지합니다. git + GitHub CLI가 있으면 전체 branch/PR/CI 워크플로우 계획을 생성합니다. 없으면 직접 모드(브랜치 없이 제자리 수정)로 전환합니다.

## 예시

### 기본 사용법

```
/blueprint myapp "migrate database to PostgreSQL"
```

다음과 같은 단계가 포함된 `plans/myapp-migrate-database-to-postgresql.md`를 생성합니다:
- Step 1: Add PostgreSQL driver and connection config
- Step 2: Create migration scripts for each table
- Step 3: Update repository layer to use new driver
- Step 4: Add integration tests against PostgreSQL
- Step 5: Remove old database code and config

### 다중 agent 프로젝트

```
/blueprint chatbot "extract LLM providers into a plugin system"
```

가능한 경우 병렬 단계(예: 플러그인 인터페이스 단계 완료 후 "Anthropic 플러그인 구현" 및 "OpenAI 플러그인 구현" 병렬 실행), 모델 티어 할당(인터페이스 설계 단계는 strongest, 구현은 default), 그리고 매 단계 후 검증되는 불변성(예: "모든 기존 테스트 통과", "코어에 provider import 없음")이 포함된 계획을 생성합니다.

## 주요 기능

- **사전 지식 없는 실행(Cold-start execution)** — 모든 단계에는 독립적인 context 요약이 포함됩니다. 이전 context가 필요하지 않습니다.
- **Adversarial review 게이트** — 모든 계획은 완성도, 의존성 정확성 및 안티 패턴 감지를 다루는 체크리스트를 기반으로 strongest-model 서브 agent에 의해 검토됩니다.
- **Branch/PR/CI 워크플로우** — 모든 단계에 내장되어 있습니다. git/gh가 없으면 직접 모드로 자연스럽게 전환됩니다.
- **병렬 단계 감지** — 의존성 그래프를 통해 공유 파일이나 출력 의존성이 없는 단계를 식별합니다.
- **계획 변경 프로토콜** — 공식 프로토콜과 감사 추적(audit trail)을 통해 단계를 분할, 삽입, 건너뛰기, 재정렬 또는 중단할 수 있습니다.
- **런타임 위험 제로** — 순수 마크다운 skill입니다. 전체 저장소에는 `.md` 파일만 포함되어 있으며 hook, 쉘 스크립트, 실행 코드, `package.json`, 빌드 단계가 없습니다. 설치 또는 호출 시 Claude Code의 기본 마크다운 skill 로더 외에는 아무것도 실행되지 않습니다.

## 설치

이 skill은 Everything Claude Code와 함께 제공됩니다. ECC가 설치되어 있다면 별도의 설치가 필요하지 않습니다.

### 전체 ECC 설치

ECC 저장소 체크아웃에서 작업 중인 경우, 다음 명령으로 skill이 있는지 확인하세요:

```bash
test -f skills/blueprint/SKILL.md
```

나중에 업데이트하려면 업데이트 전 ECC diff를 검토하세요:

```bash
cd /path/to/everything-claude-code
git fetch origin main
git log --oneline HEAD..origin/main       # review new commits before updating
git checkout <reviewed-full-sha>          # pin to a specific reviewed commit
```

### 벤더링된 단독 설치

전체 ECC 설치 외에 이 skill만 벤더링하는 경우, 검토된 파일을 ECC 저장소에서 `~/.claude/skills/blueprint/SKILL.md`로 복사하세요. 벤더링된 복사본에는 git remote가 없으므로 `git pull`을 실행하는 대신 검토된 ECC 커밋에서 파일을 다시 복사하여 업데이트하세요.

## 요구 사항

- Claude Code (`/blueprint` 슬래시 명령용)
- Git + GitHub CLI (선택 사항 — 전체 branch/PR/CI 워크플로우를 활성화하며, Blueprint는 부재를 감지하고 자동으로 직접 모드로 전환함)

## 출처

antbotlab/blueprint에서 영감을 받음 — 업스트림 프로젝트 및 참조 디자인.

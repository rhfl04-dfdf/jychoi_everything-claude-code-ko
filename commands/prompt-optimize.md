---
description: 초안 prompt를 분석하여 붙여넣고 바로 실행할 수 있는 최적화된 ECC-enriched 버전을 출력합니다. 작업을 수행하지 않으며, 자문 분석 결과만 출력합니다.
---

# /prompt-optimize

최대한의 ECC 활용을 위해 다음 prompt를 분석하고 최적화합니다.

## Your Task

아래의 사용자 입력에 **prompt-optimizer** skill을 적용하세요. 6단계 분석 파이프라인을 따릅니다:

0. **Project Detection** — CLAUDE.md를 읽고, 프로젝트 파일(package.json, go.mod, pyproject.toml 등)에서 기술 스택을 감지합니다.
1. **Intent Detection** — 작업 유형(new feature, bug fix, refactor, research, testing, review, documentation, infrastructure, design)을 분류합니다.
2. **Scope Assessment** — 감지된 경우 codebase 크기를 신호로 사용하여 복잡도(TRIVIAL / LOW / MEDIUM / HIGH / EPIC)를 평가합니다.
3. **ECC Component Matching** — 특정 skill, command, agent 및 model tier에 매핑합니다.
4. **Missing Context Detection** — 누락된 정보를 식별합니다. 3개 이상의 중요한 항목이 누락된 경우, 생성하기 전에 사용자에게 확인을 요청하세요.
5. **Workflow & Model** — 수명 주기 위치를 결정하고, model tier를 추천하며, HIGH/EPIC인 경우 여러 prompt로 분할합니다.

## Output Requirements

- prompt-optimizer skill의 출력 형식을 사용하여 진단, 권장 ECC component 및 최적화된 prompt를 제시합니다.
- **Full Version**(상세) 및 **Quick Version**(압축, intent 유형에 따라 다름)을 모두 제공합니다.
- 사용자의 입력과 동일한 언어로 응답합니다.
- 최적화된 prompt는 완성된 형태여야 하며 새 세션에 바로 복사하여 붙여넣을 수 있어야 합니다.
- 수정을 제안하거나 별도의 실행 요청을 시작하기 위한 명확한 다음 단계를 안내하는 푸터로 마무리합니다.

## CRITICAL

사용자의 작업을 실행하지 마십시오. 분석 결과와 최적화된 prompt만 출력하십시오.
사용자가 직접 실행을 요청하는 경우, `/prompt-optimize`는 자문 출력만 생성한다는 점을 설명하고 대신 일반적인 작업 요청을 시작하도록 안내하십시오.

참고: `blueprint`는 slash command가 아니라 **skill**입니다. 이를 `/...` 명령어 형태로 제시하는 대신 "blueprint skill을 사용하세요"라고 작성하십시오.

## User Input

$ARGUMENTS

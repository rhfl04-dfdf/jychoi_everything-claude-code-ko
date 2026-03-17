# Harness Audit 커맨드

현재 저장소의 에이전트 harness 설정을 감사하고 우선순위가 지정된 스코어카드를 반환합니다.

## 사용법

`/harness-audit [scope] [--format text|json]`

- `scope` (선택 사항): `repo` (기본값), `hooks`, `skills`, `commands`, `agents`
- `--format`: 출력 스타일 (`text` 기본값, 자동화를 위한 `json`)

## 평가 항목

각 범주를 `0`에서 `10`으로 점수를 매깁니다:

1. Tool Coverage
2. Context Efficiency
3. Quality Gates
4. Memory Persistence
5. Eval Coverage
6. Security Guardrails
7. Cost Efficiency

## 출력 계약

반환:

1. 70점 만점의 `overall_score`
2. 범주별 점수와 구체적인 발견 사항
3. 정확한 파일 경로가 있는 상위 3가지 액션
4. 다음으로 적용할 ECC skill 제안

## 체크리스트

- `hooks/hooks.json`, `scripts/hooks/`, hook 테스트를 검사합니다.
- `skills/`, command 커버리지, agent 커버리지를 검사합니다.
- `.cursor/`, `.opencode/`, `.codex/`에 대한 cross-harness 패리티를 확인합니다.
- 깨지거나 오래된 참조를 플래그합니다.

## 결과 예시

```text
Harness Audit (repo): 52/70
- Quality Gates: 9/10
- Eval Coverage: 6/10
- Cost Efficiency: 4/10

Top 3 Actions:
1) Add cost tracking hook in scripts/hooks/cost-tracker.js
2) Add pass@k docs and templates in skills/eval-harness/SKILL.md
3) Add command parity for /harness-audit in .opencode/commands/
```

## 인수

$ARGUMENTS:
- `repo|hooks|skills|commands|agents` (선택 사항 scope)
- `--format text|json` (선택 사항 출력 형식)

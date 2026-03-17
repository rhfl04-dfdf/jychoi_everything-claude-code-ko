---
name: continuous-agent-loop
description: quality gates, evals 및 recovery controls를 갖춘 지속적인 autonomous agent loop 패턴.
origin: ECC
---

# Continuous Agent Loop

이것은 v1.8+ canonical loop skill 이름입니다. 한 릴리스 동안 호환성을 유지하면서 `autonomous-loops`를 대체합니다.

## Loop Selection Flow

```text
Start
  |
  +-- Need strict CI/PR control? -- yes --> continuous-pr
  |                                    
  +-- Need RFC decomposition? -- yes --> rfc-dag
  |
  +-- Need exploratory parallel generation? -- yes --> infinite
  |
  +-- default --> sequential
```

## Combined Pattern

권장되는 production stack:
1. RFC decomposition (`ralphinho-rfc-pipeline`)
2. quality gates (`plankton-code-quality` + `/quality-gate`)
3. eval loop (`eval-harness`)
4. session persistence (`nanoclaw-repl`)

## Failure Modes

- 측정 가능한 진전 없는 loop churn
- 동일한 근본 원인으로 인한 반복적인 retries
- merge queue stalls
- 제한 없는 에스컬레이션으로 인한 cost drift

## Recovery

- freeze loop
- `/harness-audit` 실행
- 실패하는 단위로 scope 축소
- 명시적인 acceptance criteria로 replay

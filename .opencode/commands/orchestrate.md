---
description: 복잡한 작업을 위한 다중 agent 오케스트레이션
agent: planner
subtask: true
---

# Orchestrate 커맨드

이 복잡한 작업을 위해 여러 전문 agent를 오케스트레이션: $ARGUMENTS

## 작업 내용

1. **작업 복잡도 분석** 후 하위 작업으로 분해
2. **각 하위 작업에 최적의 agent 식별**
3. **의존성이 포함된 실행 계획 수립**
4. **실행 조율** - 가능한 경우 병렬 처리
5. **결과 종합** 후 통합된 출력 생성

## 사용 가능한 Agent

| Agent | 전문 분야 | 사용 목적 |
|-------|-----------|---------|
| planner | 구현 계획 | 복잡한 기능 설계 |
| architect | 시스템 설계 | 아키텍처 결정 |
| code-reviewer | 코드 품질 | 변경 사항 리뷰 |
| security-reviewer | 보안 분석 | 취약점 탐지 |
| tdd-guide | 테스트 주도 개발 | 기능 구현 |
| build-error-resolver | 빌드 수정 | TypeScript/빌드 에러 |
| e2e-runner | E2E 테스트 | 사용자 흐름 테스트 |
| doc-updater | 문서화 | 문서 업데이트 |
| refactor-cleaner | 코드 정리 | 죽은 코드 제거 |
| go-reviewer | Go 코드 | Go 전용 리뷰 |
| go-build-resolver | Go 빌드 | Go 빌드 에러 |
| database-reviewer | 데이터베이스 | 쿼리 최적화 |

## 오케스트레이션 패턴

### 순차 실행
```
planner → tdd-guide → code-reviewer → security-reviewer
```
사용 시점: 후속 작업이 이전 결과에 의존하는 경우

### 병렬 실행
```
┌→ security-reviewer
planner →├→ code-reviewer
└→ architect
```
사용 시점: 작업이 독립적인 경우

### Fan-Out/Fan-In
```
         ┌→ agent-1 ─┐
planner →├→ agent-2 ─┼→ synthesizer
         └→ agent-3 ─┘
```
사용 시점: 다양한 관점이 필요한 경우

## 실행 계획 형식

### 단계 1: [이름]
- Agent: [agent-name]
- 작업: [구체적 작업]
- 의존성: [없음 또는 이전 단계]

### 단계 2: [이름] (병렬)
- Agent A: [agent-name]
  - 작업: [구체적 작업]
- Agent B: [agent-name]
  - 작업: [구체적 작업]
- 의존성: 단계 1

### 단계 3: 종합
- 단계 2의 결과 결합
- 통합된 출력 생성

## 조율 규칙

1. **실행 전 계획** - 먼저 전체 실행 계획 수립
2. **핸드오프 최소화** - 컨텍스트 전환 줄이기
3. **가능하면 병렬화** - 독립적인 작업은 병렬 처리
4. **명확한 경계** - 각 agent는 특정 범위를 담당
5. **단일 진실 공급원** - 하나의 agent가 각 산출물을 소유

---

**참고**: 복잡한 작업은 다중 agent 오케스트레이션의 혜택을 받습니다. 단순한 작업은 단일 agent를 직접 사용하세요.

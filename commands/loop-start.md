# Loop Start 커맨드

안전 기본값을 갖춘 관리된 자율 루프 패턴을 시작합니다.

## 사용법

`/loop-start [pattern] [--mode safe|fast]`

- `pattern`: `sequential`, `continuous-pr`, `rfc-dag`, `infinite`
- `--mode`:
  - `safe` (기본값): 엄격한 품질 게이트와 체크포인트
  - `fast`: 속도를 위한 축소된 게이트

## 흐름

1. 저장소 상태와 브랜치 전략을 확인합니다.
2. 루프 패턴과 모델 티어 전략을 선택합니다.
3. 선택한 모드에 필요한 hook/프로파일을 활성화합니다.
4. 루프 계획을 작성하고 `.claude/plans/` 하위에 runbook을 저장합니다.
5. 루프를 시작하고 모니터링할 커맨드를 출력합니다.

## 필수 안전 확인

- 첫 번째 루프 반복 전에 테스트가 통과하는지 확인합니다.
- `ECC_HOOK_PROFILE`이 전역적으로 비활성화되어 있지 않은지 확인합니다.
- 루프에 명시적인 중단 조건이 있는지 확인합니다.

## 인수

$ARGUMENTS:
- `<pattern>` 선택 사항 (`sequential|continuous-pr|rfc-dag|infinite`)
- `--mode safe|fast` 선택 사항

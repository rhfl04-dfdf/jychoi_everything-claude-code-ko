# Model Route 커맨드

복잡도와 예산에 따라 현재 작업에 가장 적합한 모델 티어를 추천합니다.

## 사용법

`/model-route [task-description] [--budget low|med|high]`

## 라우팅 휴리스틱

- `haiku`: 결정론적이고 위험이 낮은 기계적 변경
- `sonnet`: 구현 및 리팩토링의 기본값
- `opus`: 아키텍처, 심층 리뷰, 모호한 요구사항

## 필수 출력

- 추천 모델
- 신뢰도 수준
- 이 모델이 적합한 이유
- 첫 번째 시도 실패 시 fallback 모델

## 인수

$ARGUMENTS:
- `[task-description]` 선택 사항 자유 텍스트
- `--budget low|med|high` 선택 사항

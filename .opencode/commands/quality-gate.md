# Quality Gate Command

파일 또는 프로젝트 범위에 대해 요청 시 ECC quality pipeline을 실행합니다.

## Usage

`/quality-gate [path|.] [--fix] [--strict]`

- 기본 대상: 현재 디렉토리 (`.`)
- `--fix`: 설정된 경우 auto-format/fix를 허용합니다.
- `--strict`: 지원되는 경우 warning 발생 시 실패로 처리합니다.

## Pipeline

1. 대상의 언어/tooling을 감지합니다.
2. formatter 체크를 실행합니다.
3. 가능한 경우 lint/type 체크를 실행합니다.
4. 간결한 remediation 리스트를 생성합니다.

## Notes

이 command는 hook 동작을 반영하지만 operator에 의해 호출됩니다.

## Arguments

$ARGUMENTS:
- `[path|.]` 선택적 대상 경로
- `--fix` 선택 사항
- `--strict` 선택 사항

---
description: NanoClaw v2를 시작합니다 — model routing, skill 핫로드, 브랜칭, 압축, 내보내기, 메트릭을 갖춘 ECC의 지속적이고 의존성 없는 REPL입니다.
---

# Claw 커맨드

지속적인 마크다운 히스토리와 운영 제어 기능을 갖춘 인터랙티브 AI 에이전트 세션을 시작합니다.

## 사용법

```bash
node scripts/claw.js
```

또는 npm을 통해:

```bash
npm run claw
```

## 환경 변수

| 변수 | 기본값 | 설명 |
|----------|---------|-------------|
| `CLAW_SESSION` | `default` | 세션 이름 (영숫자 + 하이픈) |
| `CLAW_SKILLS` | *(비어 있음)* | 시작 시 로드되는 쉼표로 구분된 skill 목록 |
| `CLAW_MODEL` | `sonnet` | 세션의 기본 모델 |

## REPL 커맨드

```text
/help                          도움말 표시
/clear                         현재 세션 히스토리 삭제
/history                       전체 대화 히스토리 출력
/sessions                      저장된 세션 목록
/model [name]                  모델 표시/설정
/load <skill-name>             skill을 컨텍스트에 핫로드
/branch <session-name>         현재 세션 브랜치
/search <query>                세션 전체에서 쿼리 검색
/compact                       오래된 턴 압축, 최근 컨텍스트 유지
/export <md|json|txt> [path]   세션 내보내기
/metrics                       세션 메트릭 표시
exit                           종료
```

## 참고 사항

- NanoClaw는 의존성이 없습니다.
- 세션은 `~/.claude/claw/<session>.md`에 저장됩니다.
- 압축은 가장 최근 턴을 유지하고 압축 헤더를 작성합니다.
- 내보내기는 마크다운, JSON 턴, 평문 텍스트를 지원합니다.

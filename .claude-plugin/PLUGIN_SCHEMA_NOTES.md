# 플러그인 매니페스트 스키마 노트

이 문서는 Claude Code 플러그인 매니페스트 검증기의 **문서화되지 않았지만 강제되는 제약사항**을 기록합니다.

이 규칙들은 실제 설치 실패, 검증기 동작, 알려진 작동 플러그인과의 비교를 기반으로 합니다.
조용한 오류와 반복되는 회귀를 방지하기 위해 존재합니다.

`.claude-plugin/plugin.json`을 편집하기 전에 이 문서를 먼저 읽으세요.

---

## 요약 (먼저 읽으세요)

Claude 플러그인 매니페스트 검증기는 **엄격하고 독단적**입니다.
공개 스키마 참고자료에 완전히 문서화되지 않은 규칙을 강제합니다.

가장 일반적인 실패 모드는:

> 매니페스트가 합리적으로 보이지만, 검증기가 다음과 같은 모호한 오류로 거부합니다
> `agents: Invalid input`

이 문서는 그 이유를 설명합니다.

---

## 필수 필드

### `version` (필수)

`version` 필드는 일부 예시에서 생략되더라도 검증기가 요구합니다.

없으면 마켓플레이스 설치 또는 CLI 검증 중에 설치가 실패할 수 있습니다.

예시:

```json
{
  "version": "1.1.0"
}
```

---

## 필드 형태 규칙

다음 필드는 **항상 배열이어야 합니다**:

* `agents`
* `commands`
* `skills`
* `hooks` (있는 경우)

항목이 하나뿐이어도 **문자열은 허용되지 않습니다**.

### 잘못된 예

```json
{
  "agents": "./agents"
}
```

### 올바른 예

```json
{
  "agents": ["./agents/planner.md"]
}
```

이것은 모든 컴포넌트 경로 필드에 일관되게 적용됩니다.

---

## 경로 해석 규칙 (중요)

### Agent는 반드시 명시적인 파일 경로를 사용해야 합니다

검증기는 **`agents`에 대한 디렉토리 경로를 허용하지 않습니다**.

다음도 실패합니다:

```json
{
  "agents": ["./agents/"]
}
```

대신 agent 파일을 명시적으로 나열해야 합니다:

```json
{
  "agents": [
    "./agents/planner.md",
    "./agents/architect.md",
    "./agents/code-reviewer.md"
  ]
}
```

이것이 검증 오류의 가장 일반적인 원인입니다.

### Command와 Skill

* `commands`와 `skills`는 **배열로 감싼 경우에만** 디렉토리 경로를 허용합니다
* 명시적인 파일 경로가 가장 안전하고 미래 지향적입니다

---

## 검증기 동작 참고사항

* `claude plugin validate`는 일부 마켓플레이스 미리보기보다 더 엄격합니다
* 검증이 로컬에서는 통과하지만 경로가 모호한 경우 설치 중 실패할 수 있습니다
* 오류는 종종 일반적(`Invalid input`)이며 근본 원인을 나타내지 않습니다
* 크로스 플랫폼 설치 (특히 Windows)는 경로 가정에 더 엄격합니다

검증기가 적대적이고 문자 그대로라고 가정하세요.

---

## `hooks` 필드: 추가하지 마세요

> ⚠️ **중요:** `plugin.json`에 `"hooks"` 필드를 추가하지 마세요. 이것은 회귀 테스트로 강제됩니다.

### 이것이 중요한 이유

Claude Code v2.1+는 컨벤션에 따라 설치된 플러그인에서 `hooks/hooks.json`을 **자동으로 로드**합니다. `plugin.json`에도 선언하면 다음 오류가 발생합니다:

```
Duplicate hooks file detected: ./hooks/hooks.json resolves to already-loaded file.
The standard hooks/hooks.json is loaded automatically, so manifest.hooks should
only reference additional hook files.
```

### 반복된 수정 이력

이로 인해 이 저장소에서 수정/복원 사이클이 반복되었습니다:

| 커밋 | 액션 | 트리거 |
|--------|--------|---------|
| `22ad036` | hooks 추가 | 사용자들이 "hooks가 로드되지 않음" 보고 |
| `a7bc5f2` | hooks 제거 | 사용자들이 "중복 hooks 오류" 보고 (#52) |
| `779085e` | hooks 추가 | 사용자들이 "agents가 로드되지 않음" 보고 (#88) |
| `e3a1306` | hooks 제거 | 사용자들이 "중복 hooks 오류" 보고 (#103) |

**근본 원인:** Claude Code CLI가 버전 간에 동작을 변경했습니다:
- v2.1 이전: 명시적 `hooks` 선언 필요
- v2.1+: 컨벤션에 따라 자동 로드, 중복 시 오류 발생

### 현재 규칙 (테스트로 강제)

`tests/hooks/hooks.test.js`의 `plugin.json does NOT have explicit hooks declaration` 테스트가 이것의 재도입을 방지합니다.

**추가 hook 파일을 추가하는 경우** (`hooks/hooks.json`이 아닌), 선언할 수 있습니다. 하지만 표준 `hooks/hooks.json`은 선언해서는 안 됩니다.

---

## 알려진 안티패턴

올바르게 보이지만 거부되는 것들:

* 배열 대신 문자열 값
* `agents`의 디렉토리 배열
* 누락된 `version`
* 추론된 경로에 의존
* 마켓플레이스 동작이 로컬 검증과 동일하다고 가정
* **`"hooks": "./hooks/hooks.json"` 추가** — 컨벤션에 따라 자동 로드되며 중복 오류를 유발

영리함을 피하세요. 명시적으로 작성하세요.

---

## 최소 검증된 정상 예시

```json
{
  "version": "1.1.0",
  "agents": [
    "./agents/planner.md",
    "./agents/code-reviewer.md"
  ],
  "commands": ["./commands/"],
  "skills": ["./skills/"]
}
```

이 구조는 Claude 플러그인 검증기에 대해 검증되었습니다.

**중요:** `"hooks"` 필드가 없음을 확인하세요. `hooks/hooks.json` 파일은 컨벤션에 따라 자동으로 로드됩니다. 명시적으로 추가하면 중복 오류가 발생합니다.

---

## 기여자를 위한 권고사항

`plugin.json`을 변경하는 PR을 제출하기 전에:

1. agent에 명시적인 파일 경로 사용
2. 모든 컴포넌트 필드가 배열인지 확인
3. `version` 포함
4. 다음 실행:

```bash
claude plugin validate .claude-plugin/plugin.json
```

확신이 없으면 편리함보다 자세함을 선택하세요.

---

## 이 파일이 존재하는 이유

이 저장소는 널리 포크되고 참조 구현으로 사용됩니다.

검증기 특이사항을 여기에 문서화하면:

* 반복되는 문제를 방지합니다
* 기여자의 좌절감을 줄입니다
* 생태계가 발전함에 따라 플러그인 안정성을 보존합니다

검증기가 변경되면, 이 문서를 먼저 업데이트하세요.

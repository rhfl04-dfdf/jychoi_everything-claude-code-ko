# Everything Claude Code

`everything-claude-code` 저장소 내에서 작업할 때 일반적인 코딩 조언 대신 저장소 특화 가이드가 필요한 경우 이 skill을 사용하세요.

선택적 동반 instinct는 `continuous-learning-v2`를 사용하는 팀을 위해 `.claude/homunculus/instincts/inherited/everything-claude-code-instincts.yaml`에 있습니다.

## 사용 시점

다음 영역 중 하나 이상을 다루는 작업에서 이 skill을 활성화하세요:
- Claude Code, Cursor, Codex, OpenCode 전반의 크로스 플랫폼 패리티
- hook 스크립트, hook 문서, 또는 hook 테스트
- 여러 플랫폼에 걸쳐 동기화를 유지해야 하는 skill, command, agent, rule
- 버전 업그레이드, changelog 업데이트, 플러그인 메타데이터 업데이트 등 릴리즈 작업
- 이 저장소 내의 continuous-learning 또는 instinct 워크플로우

## 작동 방식

### 1. 저장소의 개발 계약 따르기

- `feat:`, `fix:`, `docs:`, `test:`, `chore:` 등의 conventional commit 사용
- 커밋 제목은 저장소 기준인 약 70자로 간결하게 유지
- JavaScript와 TypeScript 모듈 파일명은 camelCase 선호
- skill 디렉토리와 command 파일명은 kebab-case 사용
- 기존 `*.test.js` 패턴으로 테스트 파일 유지

### 2. 루트 저장소를 진실 소스로 취급

루트 구현에서 시작하고, 의도적으로 배포되는 곳에만 변경사항을 미러링하세요.

일반적인 미러 대상:
- `.cursor/`
- `.codex/`
- `.opencode/`
- `.agents/`

모든 `.claude/` 아티팩트가 크로스 플랫폼 사본이 필요하다고 가정하지 마세요. 배포된 멀티플랫폼 표면의 일부인 파일만 미러링하세요.

### 3. 테스트와 문서와 함께 hook 업데이트

hook 동작을 변경할 때:
1. `hooks/hooks.json` 또는 `scripts/hooks/`의 관련 스크립트를 업데이트
2. `tests/hooks/` 또는 `tests/integration/`의 일치하는 테스트를 업데이트
3. 동작이나 설정이 변경된 경우 `hooks/README.md` 업데이트
4. 해당되는 경우 `.cursor/hooks/` 및 `.opencode/plugins/`의 패리티 확인

### 4. 릴리즈 메타데이터 동기화 유지

릴리즈를 준비할 때, 같은 버전이 노출되는 모든 곳에 반영되었는지 확인하세요:
- `package.json`
- `.claude-plugin/plugin.json`
- `.claude-plugin/marketplace.json`
- `.opencode/package.json`
- 릴리즈 프로세스에서 기대하는 릴리즈 노트 또는 changelog 항목

### 5. continuous-learning 변경사항에 대해 명시적으로

작업이 `skills/continuous-learning-v2/` 또는 임포트된 instinct를 다루는 경우:
- 자동 생성된 대량 출력보다 정확하고 노이즈가 낮은 instinct를 선호
- instinct 파일을 `instinct-cli.py`로 임포트 가능하게 유지
- 더 많은 가이드를 쌓는 대신 중복되거나 모순된 instinct를 제거

## 예시

### 명명 예시

```text
skills/continuous-learning-v2/SKILL.md
commands/update-docs.md
scripts/hooks/session-start.js
tests/hooks/hooks.test.js
```

### 커밋 예시

```text
fix: harden session summary extraction on Stop hook
docs: align Codex config examples with current schema
test: cover Windows formatter fallback behavior
```

### Skill 업데이트 체크리스트

```text
1. Update the root skill or command.
2. Mirror it only where that surface is shipped.
3. Run targeted tests first, then the broader suite if behavior changed.
4. Review docs and release notes for user-visible changes.
```

### 릴리즈 체크리스트

```text
1. Bump package and plugin versions.
2. Run npm test.
3. Verify platform-specific manifests.
4. Publish the release notes with a human-readable summary.
```

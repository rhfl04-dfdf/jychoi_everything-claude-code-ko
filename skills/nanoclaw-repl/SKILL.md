---
name: nanoclaw-repl
description: NanoClaw v2를 조작하고 확장합니다. ECC의 claude -p를 기반으로 구축된 zero-dependency session-aware REPL입니다.
origin: ECC
---

# NanoClaw REPL

`scripts/claw.js`를 실행하거나 확장할 때 이 skill을 사용하세요.

## Capabilities

- 지속적인 markdown-backed sessions
- `/model`을 통한 model switching
- `/load`를 통한 동적 skill loading
- `/branch`를 통한 session branching
- `/search`를 통한 cross-session search
- `/compact`를 통한 history compaction
- `/export`를 통해 md/json/txt로 export
- `/metrics`를 통한 session metrics

## Operating Guidance

1. sessions를 작업 중심으로 유지하세요.
2. 위험도가 높은 변경 전에는 branch 하세요.
3. 주요 milestone 이후에는 compact 하세요.
4. 공유하거나 보관하기 전에 export 하세요.

## Extension Rules

- 외부 runtime dependencies를 0으로 유지하세요.
- markdown-as-database 호환성을 유지하세요.
- command handlers를 deterministic하고 local하게 유지하세요.

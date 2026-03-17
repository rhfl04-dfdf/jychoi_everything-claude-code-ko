# Everything Claude Code에 기여하기

기여에 관심을 가져주셔서 감사합니다! 이 저장소는 Claude Code 사용자를 위한 커뮤니티 리소스입니다.

## 목차

- [우리가 찾는 것](#우리가-찾는-것)
- [빠른 시작](#빠른-시작)
- [스킬 기여하기](#스킬-기여하기)
- [에이전트 기여하기](#에이전트-기여하기)
- [훅 기여하기](#훅-기여하기)
- [커맨드 기여하기](#커맨드-기여하기)
- [Pull Request 프로세스](#pull-request-프로세스)

---

## 우리가 찾는 것

### 에이전트
특정 작업을 잘 처리하는 새로운 에이전트:
- 언어별 리뷰어 (Python, Go, Rust)
- 프레임워크 전문가 (Django, Rails, Laravel, Spring)
- DevOps 전문가 (Kubernetes, Terraform, CI/CD)
- 도메인 전문가 (ML 파이프라인, 데이터 엔지니어링, 모바일)

### 스킬
워크플로우 정의와 도메인 지식:
- 언어 모범 사례
- 프레임워크 패턴
- 테스팅 전략
- 아키텍처 가이드

### 훅
유용한 자동화:
- 린팅/포매팅 훅
- 보안 검사
- 유효성 검증 훅
- 알림 훅

### 커맨드
유용한 워크플로우를 호출하는 슬래시 커맨드:
- 배포 커맨드
- 테스팅 커맨드
- 코드 생성 커맨드

---

## 빠른 시작

```bash
# 1. Fork and clone
gh repo fork affaan-m/everything-claude-code --clone
cd everything-claude-code

# 2. Create a branch
git checkout -b feat/my-contribution

# 3. Add your contribution (see sections below)

# 4. Test locally
cp -r skills/my-skill ~/.claude/skills/  # for skills
# Then test with Claude Code

# 5. Submit PR
git add . && git commit -m "feat: add my-skill" && git push -u origin feat/my-contribution
```

---

## 스킬 기여하기

스킬은 Claude Code가 컨텍스트에 따라 로드하는 지식 모듈입니다.

### 디렉토리 구조

```
skills/
└── your-skill-name/
    └── SKILL.md
```

### SKILL.md 템플릿

```markdown
---
name: your-skill-name
description: Brief description shown in skill list
origin: ECC
---

# Your Skill Title

Brief overview of what this skill covers.

## Core Concepts

Explain key patterns and guidelines.

## Code Examples

\`\`\`typescript
// Include practical, tested examples
function example() {
  // Well-commented code
}
\`\`\`

## Best Practices

- Actionable guidelines
- Do's and don'ts
- Common pitfalls to avoid

## When to Use

Describe scenarios where this skill applies.
```

### 스킬 체크리스트

- [ ] 하나의 도메인/기술에 집중
- [ ] 실용적인 코드 예제 포함
- [ ] 500줄 미만
- [ ] 명확한 섹션 헤더 사용
- [ ] Claude Code에서 테스트 완료

### 스킬 예시

| 스킬 | 용도 |
|------|------|
| `coding-standards/` | TypeScript/JavaScript 패턴 |
| `frontend-patterns/` | React와 Next.js 모범 사례 |
| `backend-patterns/` | API와 데이터베이스 패턴 |
| `security-review/` | 보안 체크리스트 |

---

## 에이전트 기여하기

에이전트는 Task 도구를 통해 호출되는 전문 어시스턴트입니다.

### 파일 위치

```
agents/your-agent-name.md
```

### 에이전트 템플릿

```markdown
---
name: your-agent-name
description: What this agent does and when Claude should invoke it. Be specific!
tools: ["Read", "Write", "Edit", "Bash", "Grep", "Glob"]
model: sonnet
---

You are a [role] specialist.

## Your Role

- Primary responsibility
- Secondary responsibility
- What you DO NOT do (boundaries)

## Workflow

### Step 1: Understand
How you approach the task.

### Step 2: Execute
How you perform the work.

### Step 3: Verify
How you validate results.

## Output Format

What you return to the user.

## Examples

### Example: [Scenario]
Input: [what user provides]
Action: [what you do]
Output: [what you return]
```

### 에이전트 필드

| 필드 | 설명 | 옵션 |
|------|------|------|
| `name` | 소문자, 하이픈 연결 | `code-reviewer` |
| `description` | 호출 시점 결정에 사용 | 구체적으로 작성! |
| `tools` | 필요한 것만 포함 | `Read, Write, Edit, Bash, Grep, Glob, WebFetch, Task` |
| `model` | 복잡도 수준 | `haiku` (단순), `sonnet` (코딩), `opus` (복잡) |

### 예시 에이전트

| 에이전트 | 용도 |
|----------|------|
| `tdd-guide.md` | 테스트 주도 개발 |
| `code-reviewer.md` | 코드 리뷰 |
| `security-reviewer.md` | 보안 점검 |
| `build-error-resolver.md` | 빌드 오류 수정 |

---

## 훅 기여하기

훅은 Claude Code 이벤트에 의해 트리거되는 자동 동작입니다.

### 파일 위치

```
hooks/hooks.json
```

### 훅 유형

| 유형 | 트리거 시점 | 사용 사례 |
|------|-----------|----------|
| `PreToolUse` | 도구 실행 전 | 유효성 검증, 경고, 차단 |
| `PostToolUse` | 도구 실행 후 | 포매팅, 검사, 알림 |
| `SessionStart` | 세션 시작 시 | 컨텍스트 로딩 |
| `Stop` | 세션 종료 시 | 정리, 감사 |

### 훅 형식

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "tool == \"Bash\" && tool_input.command matches \"rm -rf /\"",
        "hooks": [
          {
            "type": "command",
            "command": "echo '[Hook] BLOCKED: Dangerous command' && exit 1"
          }
        ],
        "description": "Block dangerous rm commands"
      }
    ]
  }
}
```

### Matcher 문법

```javascript
// Match specific tools
tool == "Bash"
tool == "Edit"
tool == "Write"

// Match input patterns
tool_input.command matches "npm install"
tool_input.file_path matches "\\.tsx?$"

// Combine conditions
tool == "Bash" && tool_input.command matches "git push"
```

### 훅 예시

```json
// Block dev servers outside tmux
{
  "matcher": "tool == \"Bash\" && tool_input.command matches \"npm run dev\"",
  "hooks": [{"type": "command", "command": "echo 'Use tmux for dev servers' && exit 1"}],
  "description": "Ensure dev servers run in tmux"
}

// Auto-format after editing TypeScript
{
  "matcher": "tool == \"Edit\" && tool_input.file_path matches \"\\.tsx?$\"",
  "hooks": [{"type": "command", "command": "npx prettier --write \"$file_path\""}],
  "description": "Format TypeScript files after edit"
}

// Warn before git push
{
  "matcher": "tool == \"Bash\" && tool_input.command matches \"git push\"",
  "hooks": [{"type": "command", "command": "echo '[Hook] Review changes before pushing'"}],
  "description": "Reminder to review before push"
}
```

### 훅 체크리스트

- [ ] Matcher가 구체적 (너무 광범위하지 않게)
- [ ] 명확한 오류/정보 메시지 포함
- [ ] 올바른 종료 코드 사용 (`exit 1`은 차단, `exit 0`은 허용)
- [ ] 충분한 테스트 완료
- [ ] 설명 포함

---

## 커맨드 기여하기

커맨드는 `/command-name`으로 사용자가 호출하는 액션입니다.

### 파일 위치

```
commands/your-command.md
```

### 커맨드 템플릿

```markdown
---
description: Brief description shown in /help
---

# Command Name

## Purpose

What this command does.

## Usage

\`\`\`
/your-command [args]
\`\`\`

## Workflow

1. First step
2. Second step
3. Final step

## Output

What the user receives.
```

### 커맨드 예시

| 커맨드 | 용도 |
|--------|------|
| `commit.md` | Git 커밋 생성 |
| `code-review.md` | 코드 변경사항 리뷰 |
| `tdd.md` | TDD 워크플로우 |
| `e2e.md` | E2E 테스팅 |

---

## 크로스-하네스 및 번역

### 스킬 서브셋 (Codex 및 Cursor)

ECC는 다른 하네스를 위한 스킬 서브셋도 제공합니다:

- **Codex:** `.agents/skills/` — `agents/openai.yaml`에 나열된 스킬이 Codex에서 로드됩니다.
- **Cursor:** `.cursor/skills/` — Cursor용 스킬 서브셋이 별도로 포함됩니다.

Codex 또는 Cursor에서도 제공해야 하는 **새 스킬**을 추가한다면:

1. 먼저 `skills/your-skill-name/` 아래에 일반적인 ECC 스킬로 추가합니다.
2. **Codex**에서도 제공해야 하면 `.agents/skills/`에 반영하고, 필요하면 `agents/openai.yaml`에도 참조를 추가합니다.
3. **Cursor**에서도 제공해야 하면 Cursor 레이아웃에 맞게 `.cursor/skills/` 아래에 추가합니다.

기존 디렉터리의 구조를 확인한 뒤 같은 패턴을 따르세요. 이 서브셋 동기화는 수동이므로 PR 설명에 반영 여부를 적어 두는 것이 좋습니다.

### 번역

번역 문서는 `docs/` 아래에 있습니다. 예: `docs/zh-CN`, `docs/zh-TW`, `docs/ja-JP`.

번역된 에이전트, 커맨드, 스킬을 변경한다면:

- 대응하는 번역 파일도 함께 업데이트하거나
- 유지보수자/번역자가 후속 작업을 할 수 있도록 이슈를 열어 주세요.

---

## Pull Request 프로세스

### 1. PR 제목 형식

```
feat(skills): add rust-patterns skill
feat(agents): add api-designer agent
feat(hooks): add auto-format hook
fix(skills): update React patterns
docs: improve contributing guide
```

### 2. PR 설명

```markdown
## Summary
What you're adding and why.

## Type
- [ ] Skill
- [ ] Agent
- [ ] Hook
- [ ] Command

## Testing
How you tested this.

## Checklist
- [ ] Follows format guidelines
- [ ] Tested with Claude Code
- [ ] No sensitive info (API keys, paths)
- [ ] Clear descriptions
```

### 3. 리뷰 프로세스

1. 메인테이너가 48시간 이내에 리뷰
2. 피드백이 있으면 수정 반영
3. 승인되면 main에 머지

---

## 가이드라인

### 해야 할 것
- 기여를 집중적이고 모듈화되게 유지
- 명확한 설명 포함
- 제출 전 테스트
- 기존 패턴 따르기
- 의존성 문서화

### 하지 말아야 할 것
- 민감한 데이터 포함 (API 키, 토큰, 경로)
- 지나치게 복잡하거나 특수한 설정 추가
- 테스트하지 않은 기여 제출
- 기존 기능과 중복되는 것 생성

---

## 파일 이름 규칙

- 소문자에 하이픈 사용: `python-reviewer.md`
- 설명적으로 작성: `workflow.md`가 아닌 `tdd-workflow.md`
- name과 파일명을 일치시키기

---

## 질문이 있으신가요?

- **이슈:** [github.com/affaan-m/everything-claude-code/issues](https://github.com/affaan-m/everything-claude-code/issues)
- **X/Twitter:** [@affaanmustafa](https://x.com/affaanmustafa)

---

기여해 주셔서 감사합니다! 함께 훌륭한 리소스를 만들어 갑시다.

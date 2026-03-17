# Claude Code 완벽 가이드 (요약본)

![Header: Anthropic Hackathon Winner - Tips & Tricks for Claude Code](./assets/images/shortform/00-header.png)

---

**2월 실험적 출시 이후 Claude Code를 열정적으로 사용해왔으며, [@DRodriguezFX](https://x.com/DRodriguezFX)와 함께 [zenith.chat](https://zenith.chat)으로 Anthropic x Forum Ventures 해커톤에서 우승했습니다 - 전적으로 Claude Code만 사용했습니다.**

10개월간 매일 사용한 후의 완전한 설정을 공유합니다: skill, hook, subagent, MCP, plugin, 그리고 실제로 효과가 있는 것들.

---

## Skill과 Command

Skill은 특정 범위와 워크플로우에 제한된 규칙처럼 작동합니다. 특정 워크플로우를 실행해야 할 때 프롬프트의 축약형입니다.

Opus 4.5로 긴 코딩 세션을 마친 후 죽은 코드와 불필요한 .md 파일을 정리하고 싶으신가요? `/refactor-clean`을 실행하세요. 테스트가 필요하신가요? `/tdd`, `/e2e`, `/test-coverage`. Skill에는 코드맵도 포함될 수 있습니다 - Claude가 탐색에 컨텍스트를 소모하지 않고 코드베이스를 빠르게 탐색할 수 있는 방법입니다.

![Terminal showing chained commands](./assets/images/shortform/02-chaining-commands.jpeg)
*command를 연쇄적으로 실행하는 모습*

Command는 slash command를 통해 실행되는 skill입니다. 겹치는 부분이 있지만 저장 위치가 다릅니다:

- **Skill**: `~/.claude/skills/` - 더 넓은 범위의 워크플로우 정의
- **Command**: `~/.claude/commands/` - 빠르게 실행 가능한 프롬프트

```bash
# Example skill structure
~/.claude/skills/
  pmx-guidelines.md      # Project-specific patterns
  coding-standards.md    # Language best practices
  tdd-workflow/          # Multi-file skill with README.md
  security-review/       # Checklist-based skill
```

---

## Hook

Hook은 특정 이벤트에 발동하는 트리거 기반 자동화입니다. Skill과 달리 도구 호출과 라이프사이클 이벤트에 제한됩니다.

**Hook 유형:**

1. **PreToolUse** - 도구 실행 전 (유효성 검사, 리마인더)
2. **PostToolUse** - 도구 완료 후 (포맷팅, 피드백 루프)
3. **UserPromptSubmit** - 메시지를 보낼 때
4. **Stop** - Claude가 응답을 완료했을 때
5. **PreCompact** - 컨텍스트 압축 전
6. **Notification** - 권한 요청

**예시: 장시간 실행 명령 전 tmux 리마인더**

```json
{
  "PreToolUse": [
    {
      "matcher": "tool == \"Bash\" && tool_input.command matches \"(npm|pnpm|yarn|cargo|pytest)\"",
      "hooks": [
        {
          "type": "command",
          "command": "if [ -z \"$TMUX\" ]; then echo '[Hook] Consider tmux for session persistence' >&2; fi"
        }
      ]
    }
  ]
}
```

![PostToolUse hook feedback](./assets/images/shortform/03-posttooluse-hook.png)
*PostToolUse hook 실행 시 Claude Code에서 받는 피드백 예시*

**꿀팁:** `hookify` plugin을 사용하면 JSON을 직접 작성하는 대신 대화형으로 hook을 생성할 수 있습니다. `/hookify`를 실행하고 원하는 것을 설명하세요.

---

## Subagent

Subagent는 오케스트레이터(메인 Claude)가 제한된 범위로 작업을 위임할 수 있는 프로세스입니다. 백그라운드 또는 포그라운드에서 실행되어 메인 agent의 컨텍스트를 확보해줍니다.

Subagent는 skill과 잘 어울립니다 - skill의 하위 집합을 실행할 수 있는 subagent에 작업을 위임하면 해당 skill을 자율적으로 사용할 수 있습니다. 또한 특정 도구 권한으로 샌드박스화할 수 있습니다.

```bash
# Example subagent structure
~/.claude/agents/
  planner.md           # Feature implementation planning
  architect.md         # System design decisions
  tdd-guide.md         # Test-driven development
  code-reviewer.md     # Quality/security review
  security-reviewer.md # Vulnerability analysis
  build-error-resolver.md
  e2e-runner.md
  refactor-cleaner.md
```

subagent별로 허용된 도구, MCP, 권한을 설정하여 적절한 범위를 지정하세요.

---

## 규칙과 메모리

`.rules` 폴더에는 Claude가 **항상** 따라야 할 모범 사례가 담긴 `.md` 파일이 있습니다. 두 가지 접근법:

1. **단일 CLAUDE.md** - 하나의 파일에 모든 것 (사용자 또는 프로젝트 수준)
2. **규칙 폴더** - 관심사별로 그룹화된 모듈식 `.md` 파일

```bash
~/.claude/rules/
  security.md      # No hardcoded secrets, validate inputs
  coding-style.md  # Immutability, file organization
  testing.md       # TDD workflow, 80% coverage
  git-workflow.md  # Commit format, PR process
  agents.md        # When to delegate to subagents
  performance.md   # Model selection, context management
```

**규칙 예시:**

- 코드베이스에 이모지 금지
- 프론트엔드에서 보라색 계열 자제
- 배포 전 항상 코드 테스트
- 대형 파일보다 모듈식 코드 우선
- console.log 커밋 금지

---

## MCP (Model Context Protocol)

MCP는 Claude를 외부 서비스에 직접 연결합니다. API의 대체가 아닌 프롬프트 기반 래퍼로, 정보 탐색에 더 많은 유연성을 제공합니다.

**예시:** Supabase MCP를 사용하면 Claude가 특정 데이터를 가져오고, 복사-붙여넣기 없이 직접 업스트림으로 SQL을 실행할 수 있습니다. 데이터베이스, 배포 플랫폼 등도 마찬가지입니다.

![Supabase MCP listing tables](./assets/images/shortform/04-supabase-mcp.jpeg)
*Supabase MCP가 public 스키마 내 테이블을 나열하는 예시*

**Claude 내장 Chrome:** Claude가 브라우저를 자율적으로 제어할 수 있는 내장 plugin MCP입니다 - 클릭하며 작동 방식을 확인합니다.

**중요: 컨텍스트 윈도우 관리**

MCP는 선별적으로 사용하세요. 모든 MCP를 사용자 설정에 넣되 **사용하지 않는 것은 모두 비활성화**합니다. `/plugins`로 이동하여 스크롤하거나 `/mcp`를 실행하세요.

![/plugins interface](./assets/images/shortform/05-plugins-interface.jpeg)
*/plugins를 사용하여 MCP로 이동, 현재 설치된 MCP와 상태 확인*

압축 전 200k 컨텍스트 윈도우가 너무 많은 도구가 활성화되면 70k에 불과할 수 있습니다. 성능이 크게 저하됩니다.

**경험 법칙:** 설정에 20-30개의 MCP를 두되, 활성화는 10개 미만 / 활성 도구는 80개 미만으로 유지하세요.

```bash
# Check enabled MCPs
/mcp

# Disable unused ones in ~/.claude.json under projects.disabledMcpServers
```

---

## Plugin

Plugin은 번거로운 수동 설정 대신 도구를 쉽게 설치할 수 있도록 패키징합니다. Plugin은 skill + MCP의 조합이거나 hook/도구를 함께 번들링한 것일 수 있습니다.

**Plugin 설치:**

```bash
# Add a marketplace
# mgrep plugin by @mixedbread-ai
claude plugin marketplace add https://github.com/mixedbread-ai/mgrep

# Open Claude, run /plugins, find new marketplace, install from there
```

![Marketplaces tab showing mgrep](./assets/images/shortform/06-marketplaces-mgrep.jpeg)
*새로 설치된 Mixedbread-Grep 마켓플레이스 표시*

**LSP Plugin**은 에디터 밖에서 Claude Code를 자주 실행하는 경우 특히 유용합니다. Language Server Protocol은 IDE를 열지 않고도 Claude에게 실시간 타입 검사, 정의로 이동, 지능형 자동 완성을 제공합니다.

```bash
# Enabled plugins example
typescript-lsp@claude-plugins-official  # TypeScript intelligence
pyright-lsp@claude-plugins-official     # Python type checking
hookify@claude-plugins-official         # Create hooks conversationally
mgrep@Mixedbread-Grep                   # Better search than ripgrep
```

MCP와 동일한 주의사항 - 컨텍스트 윈도우를 주시하세요.

---

## 팁과 트릭

### 키보드 단축키

- `Ctrl+U` - 전체 줄 삭제 (백스페이스 연타보다 빠름)
- `!` - 빠른 bash 명령어 접두사
- `@` - 파일 검색
- `/` - slash command 시작
- `Shift+Enter` - 여러 줄 입력
- `Tab` - 사고 과정 표시 토글
- `Esc Esc` - Claude 중단 / 코드 복원

### 병렬 워크플로우

- **Fork** (`/fork`) - 대화를 포크하여 겹치지 않는 작업을 병렬로 처리 (대기열 메시지 연타 대신)
- **Git Worktree** - 충돌 없이 여러 Claude를 병렬로 실행. 각 worktree는 독립적인 체크아웃

```bash
git worktree add ../feature-branch feature-branch
# Now run separate Claude instances in each worktree
```

### 장시간 실행 명령을 위한 tmux

Claude가 실행하는 로그/bash 프로세스를 스트리밍하고 모니터링하세요:

<https://github.com/user-attachments/assets/shortform/07-tmux-video.mp4>

```bash
tmux new -s dev
# Claude runs commands here, you can detach and reattach
tmux attach -t dev
```

### mgrep > grep

`mgrep`은 ripgrep/grep 대비 상당한 개선입니다. plugin 마켓플레이스를 통해 설치한 후 `/mgrep` skill을 사용하세요. 로컬 검색과 웹 검색 모두 지원합니다.

```bash
mgrep "function handleSubmit"  # Local search
mgrep --web "Next.js 15 app router changes"  # Web search
```

### 기타 유용한 Command

- `/rewind` - 이전 상태로 되돌리기
- `/statusline` - 브랜치, 컨텍스트 %, 할일 등으로 커스터마이즈
- `/checkpoints` - 파일 수준 실행 취소 지점
- `/compact` - 수동으로 컨텍스트 압축 트리거

### GitHub Actions CI/CD

GitHub Actions로 PR에 대한 코드 리뷰를 설정하세요. Claude는 설정 시 PR을 자동으로 리뷰할 수 있습니다.

![Claude bot approving a PR](./assets/images/shortform/08-github-pr-review.jpeg)
*Claude가 버그 수정 PR을 승인하는 모습*

### 샌드박싱

위험한 작업에는 샌드박스 모드를 사용하세요 - Claude가 실제 시스템에 영향을 주지 않는 제한된 환경에서 실행됩니다.

---

## 에디터에 대하여

에디터 선택은 Claude Code 워크플로우에 큰 영향을 미칩니다. Claude Code는 어떤 터미널에서든 작동하지만, 유능한 에디터와 함께 사용하면 실시간 파일 추적, 빠른 탐색, 통합 명령 실행이 가능해집니다.

### Zed (개인적 선호)

[Zed](https://zed.dev)를 사용합니다 - Rust로 작성되어 진짜 빠릅니다. 즉시 열리고, 대규모 코드베이스도 끄떡없이 처리하며, 시스템 리소스를 거의 사용하지 않습니다.

**Zed + Claude Code가 좋은 조합인 이유:**

- **속도** - Rust 기반 성능으로 Claude가 빠르게 파일을 편집할 때 지연 없음. 에디터가 따라갑니다
- **Agent 패널 통합** - Zed의 Claude 통합으로 Claude가 편집하는 파일 변경사항을 실시간 추적 가능. 에디터를 떠나지 않고 Claude가 참조하는 파일 간 이동
- **CMD+Shift+R 명령 팔레트** - 모든 커스텀 slash command, 디버거, 빌드 스크립트를 검색 가능한 UI로 빠르게 접근
- **최소한의 리소스 사용** - 무거운 작업 중 Claude와 RAM/CPU를 놓고 경쟁하지 않음. Opus 실행 시 중요
- **Vim 모드** - 원하신다면 전체 vim 키바인딩 지원

![Zed Editor with custom commands](./assets/images/shortform/09-zed-editor.jpeg)
*CMD+Shift+R을 사용한 커스텀 command 드롭다운이 있는 Zed Editor. 우측 하단의 과녁이 팔로우 모드.*

**에디터에 구애받지 않는 팁:**

1. **화면 분할** - 한쪽에는 Claude Code가 있는 터미널, 다른 쪽에는 에디터
2. **Ctrl + G** - Zed에서 Claude가 현재 작업 중인 파일을 빠르게 열기
3. **자동 저장** - 자동 저장을 활성화하여 Claude의 파일 읽기가 항상 최신 상태가 되도록
4. **Git 통합** - 에디터의 git 기능을 사용하여 커밋 전 Claude의 변경사항 검토
5. **파일 감시** - 대부분의 에디터는 변경된 파일을 자동으로 다시 로드하므로, 활성화되어 있는지 확인

### VSCode / Cursor

이것도 훌륭한 선택이며 Claude Code와 잘 작동합니다. `\ide`를 사용하여 에디터와 자동 동기화되는 터미널 형식으로 사용할 수 있으며 LSP 기능을 활성화합니다 (plugin으로 인해 다소 중복). 또는 에디터에 더 통합되고 매칭되는 UI를 가진 확장 프로그램을 선택할 수 있습니다.

![VS Code Claude Code Extension](./assets/images/shortform/10-vscode-extension.jpeg)
*VS Code 확장 프로그램은 IDE에 직접 통합된 네이티브 그래픽 인터페이스를 Claude Code에 제공합니다.*

---

## 나의 설정

### Plugin

**설치됨:** (보통 한 번에 4-5개만 활성화합니다)

```markdown
ralph-wiggum@claude-code-plugins       # Loop automation
frontend-design@claude-code-plugins    # UI/UX patterns
commit-commands@claude-code-plugins    # Git workflow
security-guidance@claude-code-plugins  # Security checks
pr-review-toolkit@claude-code-plugins  # PR automation
typescript-lsp@claude-plugins-official # TS intelligence
hookify@claude-plugins-official        # Hook creation
code-simplifier@claude-plugins-official
feature-dev@claude-code-plugins
explanatory-output-style@claude-code-plugins
code-review@claude-code-plugins
context7@claude-plugins-official       # Live documentation
pyright-lsp@claude-plugins-official    # Python types
mgrep@Mixedbread-Grep                  # Better search
```

### MCP 서버

**설정됨 (사용자 수준):**

```json
{
  "github": { "command": "npx", "args": ["-y", "@modelcontextprotocol/server-github"] },
  "firecrawl": { "command": "npx", "args": ["-y", "firecrawl-mcp"] },
  "supabase": {
    "command": "npx",
    "args": ["-y", "@supabase/mcp-server-supabase@latest", "--project-ref=YOUR_REF"]
  },
  "memory": { "command": "npx", "args": ["-y", "@modelcontextprotocol/server-memory"] },
  "sequential-thinking": {
    "command": "npx",
    "args": ["-y", "@modelcontextprotocol/server-sequential-thinking"]
  },
  "vercel": { "type": "http", "url": "https://mcp.vercel.com" },
  "railway": { "command": "npx", "args": ["-y", "@railway/mcp-server"] },
  "cloudflare-docs": { "type": "http", "url": "https://docs.mcp.cloudflare.com/mcp" },
  "cloudflare-workers-bindings": {
    "type": "http",
    "url": "https://bindings.mcp.cloudflare.com/mcp"
  },
  "clickhouse": { "type": "http", "url": "https://mcp.clickhouse.cloud/mcp" },
  "AbletonMCP": { "command": "uvx", "args": ["ableton-mcp"] },
  "magic": { "command": "npx", "args": ["-y", "@magicuidesign/mcp@latest"] }
}
```

핵심은 이것입니다 - 14개의 MCP를 설정해두었지만 프로젝트당 ~5-6개만 활성화합니다. 컨텍스트 윈도우를 건강하게 유지합니다.

### 주요 Hook

```json
{
  "PreToolUse": [
    { "matcher": "npm|pnpm|yarn|cargo|pytest", "hooks": ["tmux reminder"] },
    { "matcher": "Write && .md file", "hooks": ["block unless README/CLAUDE"] },
    { "matcher": "git push", "hooks": ["open editor for review"] }
  ],
  "PostToolUse": [
    { "matcher": "Edit && .ts/.tsx/.js/.jsx", "hooks": ["prettier --write"] },
    { "matcher": "Edit && .ts/.tsx", "hooks": ["tsc --noEmit"] },
    { "matcher": "Edit", "hooks": ["grep console.log warning"] }
  ],
  "Stop": [
    { "matcher": "*", "hooks": ["check modified files for console.log"] }
  ]
}
```

### 커스텀 상태 표시줄

사용자, 디렉토리, dirty 표시가 있는 git 브랜치, 남은 컨텍스트 %, 모델, 시간, 할일 개수를 표시합니다:

![Custom status line](./assets/images/shortform/11-statusline.jpeg)
*Mac 루트 디렉토리에서의 상태 표시줄 예시*

```
affoon:~ ctx:65% Opus 4.5 19:52
▌▌ plan mode on (shift+tab to cycle)
```

### 규칙 구조

```
~/.claude/rules/
  security.md      # Mandatory security checks
  coding-style.md  # Immutability, file size limits
  testing.md       # TDD, 80% coverage
  git-workflow.md  # Conventional commits
  agents.md        # Subagent delegation rules
  patterns.md      # API response formats
  performance.md   # Model selection (Haiku vs Sonnet vs Opus)
  hooks.md         # Hook documentation
```

### Subagent

```
~/.claude/agents/
  planner.md           # Break down features
  architect.md         # System design
  tdd-guide.md         # Write tests first
  code-reviewer.md     # Quality review
  security-reviewer.md # Vulnerability scan
  build-error-resolver.md
  e2e-runner.md        # Playwright tests
  refactor-cleaner.md  # Dead code removal
  doc-updater.md       # Keep docs synced
```

---

## 핵심 요약

1. **과도하게 복잡하게 만들지 마세요** - 설정은 아키텍처가 아니라 미세 조정처럼 다루세요
2. **컨텍스트 윈도우는 소중합니다** - 사용하지 않는 MCP와 plugin은 비활성화하세요
3. **병렬 실행** - 대화를 포크하고 git worktree를 사용하세요
4. **반복 작업을 자동화하세요** - 포맷팅, 린팅, 리마인더를 위한 hook
5. **subagent의 범위를 제한하세요** - 제한된 도구 = 집중된 실행

---

## 참고 자료

- [Plugin 레퍼런스](https://code.claude.com/docs/en/plugins-reference)
- [Hook 문서](https://code.claude.com/docs/en/hooks)
- [체크포인팅](https://code.claude.com/docs/en/checkpointing)
- [인터랙티브 모드](https://code.claude.com/docs/en/interactive-mode)
- [메모리 시스템](https://code.claude.com/docs/en/memory)
- [Subagent](https://code.claude.com/docs/en/sub-agents)
- [MCP 개요](https://code.claude.com/docs/en/mcp-overview)

---

**참고:** 이것은 세부 내용의 일부입니다. 고급 패턴은 [상세 가이드](./the-longform-guide.md)를 참조하세요.

---

*NYC에서 [@DRodriguezFX](https://x.com/DRodriguezFX)와 함께 [zenith.chat](https://zenith.chat)을 만들어 Anthropic x Forum Ventures 해커톤에서 우승했습니다*

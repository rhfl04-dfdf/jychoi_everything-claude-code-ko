# Hook

Hook은 Claude Code 도구 실행 전후에 발생하는 이벤트 기반 자동화입니다. 코드 품질을 강제하고, 실수를 조기에 발견하며, 반복적인 검사를 자동화합니다.

## Hook 작동 방식

```
User request → Claude picks a tool → PreToolUse hook runs → Tool executes → PostToolUse hook runs
```

- **PreToolUse** hook은 도구 실행 전에 실행됩니다. **차단**(종료 코드 2) 또는 **경고**(차단 없이 stderr) 할 수 있습니다.
- **PostToolUse** hook은 도구 완료 후에 실행됩니다. 출력을 분석할 수 있지만 차단할 수는 없습니다.
- **Stop** hook은 각 Claude 응답 후에 실행됩니다.
- **SessionStart/SessionEnd** hook은 세션 생명주기 경계에서 실행됩니다.
- **PreCompact** hook은 컨텍스트 압축 전에 실행되며, 상태 저장에 유용합니다.

## 이 플러그인의 Hook

### PreToolUse Hook

| Hook | Matcher | 동작 | 종료 코드 |
|------|---------|----------|-----------|
| **개발 서버 차단** | `Bash` | tmux 외부에서 `npm run dev` 등을 차단 — 로그 접근 보장 | 2 (차단) |
| **Tmux 안내** | `Bash` | 장시간 실행 명령어(npm test, cargo build, docker)에 tmux 제안 | 0 (경고) |
| **Git push 안내** | `Bash` | `git push` 전 변경사항 검토 안내 | 0 (경고) |
| **문서 파일 경고** | `Write` | 비표준 `.md`/`.txt` 파일 경고 (README, CLAUDE, CONTRIBUTING, CHANGELOG, LICENSE, SKILL, docs/, skills/ 허용); 크로스 플랫폼 경로 처리 | 0 (경고) |
| **전략적 compact** | `Edit\|Write` | 논리적 간격(약 50회 도구 호출마다)으로 수동 `/compact` 제안 | 0 (경고) |
| **InsAIts 보안 모니터 (opt-in)** | `Bash\|Write\|Edit\|MultiEdit` | 고신호 도구 입력에 대한 선택적 보안 스캔. `ECC_ENABLE_INSAITS=1`이 아니면 비활성화. 심각한 발견 시 차단, 비심각 시 경고, `.insaits_audit_session.jsonl`에 감사 로그 기록. `pip install insa-its` 필요. [세부 정보](../scripts/hooks/insaits-security-monitor.py) | 2 (심각 차단) / 0 (경고) |

### PostToolUse Hook

| Hook | Matcher | 동작 |
|------|---------|-------------|
| **PR 로거** | `Bash` | `gh pr create` 후 PR URL과 리뷰 명령어 로깅 |
| **빌드 분석** | `Bash` | 빌드 명령어 후 백그라운드 분석 (비동기, 비차단) |
| **품질 게이트** | `Edit\|Write\|MultiEdit` | 편집 후 빠른 품질 검사 실행 |
| **Prettier 포맷** | `Edit` | 편집 후 Prettier로 JS/TS 파일 자동 포맷 |
| **TypeScript 검사** | `Edit` | `.ts`/`.tsx` 파일 편집 후 `tsc --noEmit` 실행 |
| **console.log 경고** | `Edit` | 편집된 파일의 `console.log` 문 경고 |

### 생명주기 Hook

| Hook | 이벤트 | 동작 |
|------|-------|-------------|
| **세션 시작** | `SessionStart` | 이전 컨텍스트 로드 및 패키지 매니저 감지 |
| **Pre-compact** | `PreCompact` | 컨텍스트 압축 전 상태 저장 |
| **Console.log 감사** | `Stop` | 각 응답 후 수정된 모든 파일에서 `console.log` 확인 |
| **세션 요약** | `Stop` | 트랜스크립트 경로가 있을 때 세션 상태 지속 |
| **패턴 추출** | `Stop` | 추출 가능한 패턴에 대한 세션 평가 (지속적 학습) |
| **비용 추적기** | `Stop` | 경량 실행 비용 텔레메트리 마커 발생 |
| **세션 종료 마커** | `SessionEnd` | 생명주기 마커 및 정리 로그 |

## Hook 커스터마이징

### Hook 비활성화

`hooks.json`에서 hook 항목을 제거하거나 주석 처리합니다. 플러그인으로 설치된 경우, `~/.claude/settings.json`에서 오버라이드합니다:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write",
        "hooks": [],
        "description": "Override: allow all .md file creation"
      }
    ]
  }
}
```

### 런타임 Hook 제어 (권장)

`hooks.json`을 편집하지 않고 환경 변수로 hook 동작을 제어합니다:

```bash
# minimal | standard | strict (default: standard)
export ECC_HOOK_PROFILE=standard

# Disable specific hook IDs (comma-separated)
export ECC_DISABLED_HOOKS="pre:bash:tmux-reminder,post:edit:typecheck"
```

프로파일:
- `minimal` — 필수 생명주기 및 안전 hook만 유지.
- `standard` — 기본값; 균형 잡힌 품질 + 안전 검사.
- `strict` — 추가 안내 및 더 엄격한 가드레일 활성화.

### 직접 Hook 작성

Hook은 stdin으로 도구 입력을 JSON으로 받고 stdout으로 JSON을 출력해야 하는 셸 명령어입니다.

**기본 구조:**

```javascript
// my-hook.js
let data = '';
process.stdin.on('data', chunk => data += chunk);
process.stdin.on('end', () => {
  const input = JSON.parse(data);

  // Access tool info
  const toolName = input.tool_name;        // "Edit", "Bash", "Write", etc.
  const toolInput = input.tool_input;      // Tool-specific parameters
  const toolOutput = input.tool_output;    // Only available in PostToolUse

  // Warn (non-blocking): write to stderr
  console.error('[Hook] Warning message shown to Claude');

  // Block (PreToolUse only): exit with code 2
  // process.exit(2);

  // Always output the original data to stdout
  console.log(data);
});
```

**종료 코드:**
- `0` — 성공 (실행 계속)
- `2` — 도구 호출 차단 (PreToolUse만 가능)
- 기타 비제로 — 오류 (로깅되지만 차단하지 않음)

### Hook 입력 스키마

```typescript
interface HookInput {
  tool_name: string;          // "Bash", "Edit", "Write", "Read", etc.
  tool_input: {
    command?: string;         // Bash: the command being run
    file_path?: string;       // Edit/Write/Read: target file
    old_string?: string;      // Edit: text being replaced
    new_string?: string;      // Edit: replacement text
    content?: string;         // Write: file content
  };
  tool_output?: {             // PostToolUse only
    output?: string;          // Command/tool output
  };
}
```

### 비동기 Hook

메인 흐름을 차단하지 않아야 하는 hook(예: 백그라운드 분석)의 경우:

```json
{
  "type": "command",
  "command": "node my-slow-hook.js",
  "async": true,
  "timeout": 30
}
```

비동기 hook은 백그라운드에서 실행됩니다. 도구 실행을 차단할 수 없습니다.

## 일반적인 Hook 레시피

### TODO 주석 경고

```json
{
  "matcher": "Edit",
  "hooks": [{
    "type": "command",
    "command": "node -e \"let d='';process.stdin.on('data',c=>d+=c);process.stdin.on('end',()=>{const i=JSON.parse(d);const ns=i.tool_input?.new_string||'';if(/TODO|FIXME|HACK/.test(ns)){console.error('[Hook] New TODO/FIXME added - consider creating an issue')}console.log(d)})\""
  }],
  "description": "Warn when adding TODO/FIXME comments"
}
```

### 대용량 파일 생성 차단

```json
{
  "matcher": "Write",
  "hooks": [{
    "type": "command",
    "command": "node -e \"let d='';process.stdin.on('data',c=>d+=c);process.stdin.on('end',()=>{const i=JSON.parse(d);const c=i.tool_input?.content||'';const lines=c.split('\\n').length;if(lines>800){console.error('[Hook] BLOCKED: File exceeds 800 lines ('+lines+' lines)');console.error('[Hook] Split into smaller, focused modules');process.exit(2)}console.log(d)})\""
  }],
  "description": "Block creation of files larger than 800 lines"
}
```

### ruff로 Python 파일 자동 포맷

```json
{
  "matcher": "Edit",
  "hooks": [{
    "type": "command",
    "command": "node -e \"let d='';process.stdin.on('data',c=>d+=c);process.stdin.on('end',()=>{const i=JSON.parse(d);const p=i.tool_input?.file_path||'';if(/\\.py$/.test(p)){const{execFileSync}=require('child_process');try{execFileSync('ruff',['format',p],{stdio:'pipe'})}catch(e){}}console.log(d)})\""
  }],
  "description": "Auto-format Python files with ruff after edits"
}
```

### 새 소스 파일 옆에 테스트 파일 요구

```json
{
  "matcher": "Write",
  "hooks": [{
    "type": "command",
    "command": "node -e \"const fs=require('fs');let d='';process.stdin.on('data',c=>d+=c);process.stdin.on('end',()=>{const i=JSON.parse(d);const p=i.tool_input?.file_path||'';if(/src\\/.*\\.(ts|js)$/.test(p)&&!/\\.test\\.|\\.spec\\./.test(p)){const testPath=p.replace(/\\.(ts|js)$/,'.test.$1');if(!fs.existsSync(testPath)){console.error('[Hook] No test file found for: '+p);console.error('[Hook] Expected: '+testPath);console.error('[Hook] Consider writing tests first (/tdd)')}}console.log(d)})\""
  }],
  "description": "Remind to create tests when adding new source files"
}
```

## 크로스 플랫폼 참고사항

Hook 로직은 Windows, macOS, Linux에서 크로스 플랫폼 동작을 위해 Node.js 스크립트로 구현됩니다. 소수의 셸 래퍼는 지속적 학습 옵저버 hook에 사용되며, 이러한 래퍼는 프로파일 게이트 방식이고 Windows 안전 폴백 동작이 있습니다.

## 관련 항목

- [rules/common/hooks.md](../rules/common/hooks.md) — Hook 아키텍처 가이드라인
- [skills/strategic-compact/](../skills/strategic-compact/) — 전략적 압축 skill
- [scripts/hooks/](../scripts/hooks/) — Hook 스크립트 구현

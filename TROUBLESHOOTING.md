# 문제 해결 가이드

Everything Claude Code (ECC) 플러그인의 일반적인 문제 및 해결 방법.

## 목차

- [메모리 및 컨텍스트 문제](#메모리--컨텍스트-문제)
- [Agent Harness 실패](#agent-harness-실패)
- [Hook 및 워크플로우 오류](#hook--워크플로우-오류)
- [설치 및 설정](#설치--설정)
- [성능 문제](#성능-문제)
- [일반적인 오류 메시지](#일반적인-오류-메시지)
- [도움 받기](#도움-받기)

---

## 메모리 및 컨텍스트 문제

### 컨텍스트 윈도우 초과

**증상:** "Context too long" 오류 또는 불완전한 응답

**원인:**
- 토큰 제한을 초과하는 대용량 파일 업로드
- 누적된 대화 이력
- 단일 세션에서 여러 개의 대용량 도구 출력

**해결 방법:**
```bash
# 1. Clear conversation history and start fresh
# Use Claude Code: "New Chat" or Cmd/Ctrl+Shift+N

# 2. Reduce file size before analysis
head -n 100 large-file.log > sample.log

# 3. Use streaming for large outputs
head -n 50 large-file.txt

# 4. Split tasks into smaller chunks
# Instead of: "Analyze all 50 files"
# Use: "Analyze files in src/components/ directory"
```

### 메모리 지속성 실패

**증상:** Agent가 이전 컨텍스트나 관찰을 기억하지 못함

**원인:**
- 지속적 학습 hook 비활성화
- 손상된 관찰 파일
- 프로젝트 감지 실패

**해결 방법:**
```bash
# Check if observations are being recorded
ls ~/.claude/homunculus/projects/*/observations.jsonl

# Find the current project's hash id
python3 - <<'PY'
import json, os
registry_path = os.path.expanduser("~/.claude/homunculus/projects.json")
with open(registry_path) as f:
    registry = json.load(f)
for project_id, meta in registry.items():
    if meta.get("root") == os.getcwd():
        print(project_id)
        break
else:
    raise SystemExit("Project hash not found in ~/.claude/homunculus/projects.json")
PY

# View recent observations for that project
tail -20 ~/.claude/homunculus/projects/<project-hash>/observations.jsonl

# Back up a corrupted observations file before recreating it
mv ~/.claude/homunculus/projects/<project-hash>/observations.jsonl \
  ~/.claude/homunculus/projects/<project-hash>/observations.jsonl.bak.$(date +%Y%m%d-%H%M%S)

# Verify hooks are enabled
grep -r "observe" ~/.claude/settings.json
```

---

## Agent Harness 실패

### Agent를 찾을 수 없음

**증상:** "Agent not loaded" 또는 "Unknown agent" 오류

**원인:**
- 플러그인이 올바르게 설치되지 않음
- Agent 경로 잘못된 설정
- 마켓플레이스 대 수동 설치 불일치

**해결 방법:**
```bash
# Check plugin installation
ls ~/.claude/plugins/cache/

# Verify agent exists (marketplace install)
ls ~/.claude/plugins/cache/*/agents/

# For manual install, agents should be in:
ls ~/.claude/agents/  # Custom agents only

# Reload plugin
# Claude Code → Settings → Extensions → Reload
```

### 워크플로우 실행 중단

**증상:** Agent가 시작되지만 완료되지 않음

**원인:**
- Agent 로직의 무한 루프
- 사용자 입력 대기 차단
- API 대기 중 네트워크 타임아웃

**해결 방법:**
```bash
# 1. Check for stuck processes
ps aux | grep claude

# 2. Enable debug mode
export CLAUDE_DEBUG=1

# 3. Set shorter timeouts
export CLAUDE_TIMEOUT=30

# 4. Check network connectivity
curl -I https://api.anthropic.com
```

### 도구 사용 오류

**증상:** "Tool execution failed" 또는 권한 거부

**원인:**
- 누락된 의존성 (npm, python 등)
- 불충분한 파일 권한
- 경로를 찾을 수 없음

**해결 방법:**
```bash
# Verify required tools are installed
which node python3 npm git

# Fix permissions on hook scripts
chmod +x ~/.claude/plugins/cache/*/hooks/*.sh
chmod +x ~/.claude/plugins/cache/*/skills/*/hooks/*.sh

# Check PATH includes necessary binaries
echo $PATH
```

---

## Hook 및 워크플로우 오류

### Hook이 실행되지 않음

**증상:** 사전/사후 hook이 실행되지 않음

**원인:**
- Hook이 settings.json에 등록되지 않음
- 잘못된 hook 구문
- Hook 스크립트가 실행 불가능

**해결 방법:**
```bash
# Check hooks are registered
grep -A 10 '"hooks"' ~/.claude/settings.json

# Verify hook files exist and are executable
ls -la ~/.claude/plugins/cache/*/hooks/

# Test hook manually
bash ~/.claude/plugins/cache/*/hooks/pre-bash.sh <<< '{"command":"echo test"}'

# Re-register hooks (if using plugin)
# Disable and re-enable plugin in Claude Code settings
```

### Python/Node 버전 불일치

**증상:** "python3 not found" 또는 "node: command not found"

**원인:**
- Python/Node 설치 누락
- PATH 설정 안됨
- 잘못된 Python 버전 (Windows)

**해결 방법:**
```bash
# Install Python 3 (if missing)
# macOS: brew install python3
# Ubuntu: sudo apt install python3
# Windows: Download from python.org

# Install Node.js (if missing)
# macOS: brew install node
# Ubuntu: sudo apt install nodejs npm
# Windows: Download from nodejs.org

# Verify installations
python3 --version
node --version
npm --version

# Windows: Ensure python (not python3) works
python --version
```

### 개발 서버 차단기 오탐

**증상:** Hook이 "dev"를 언급하는 합법적인 명령을 차단

**원인:**
- Heredoc 내용이 패턴 매치를 트리거
- 인수에 "dev"가 포함된 비개발 명령

**해결 방법:**
```bash
# This is fixed in v1.8.0+ (PR #371)
# Upgrade plugin to latest version

# Workaround: Wrap dev servers in tmux
tmux new-session -d -s dev "npm run dev"
tmux attach -t dev

# Disable hook temporarily if needed
# Edit ~/.claude/settings.json and remove pre-bash hook
```

---

## 설치 및 설정

### 플러그인이 로드되지 않음

**증상:** 설치 후 플러그인 기능을 사용할 수 없음

**원인:**
- 마켓플레이스 캐시가 업데이트되지 않음
- Claude Code 버전 비호환성
- 손상된 플러그인 파일

**해결 방법:**
```bash
# Inspect the plugin cache before changing it
ls -la ~/.claude/plugins/cache/

# Back up the plugin cache instead of deleting it in place
mv ~/.claude/plugins/cache ~/.claude/plugins/cache.backup.$(date +%Y%m%d-%H%M%S)
mkdir -p ~/.claude/plugins/cache

# Reinstall from marketplace
# Claude Code → Extensions → Everything Claude Code → Uninstall
# Then reinstall from marketplace

# Check Claude Code version
claude --version
# Requires Claude Code 2.0+

# Manual install (if marketplace fails)
git clone https://github.com/affaan-m/everything-claude-code.git
cp -r everything-claude-code ~/.claude/plugins/ecc
```

### 패키지 매니저 감지 실패

**증상:** 잘못된 패키지 매니저 사용 (pnpm 대신 npm)

**원인:**
- 잠금 파일 없음
- CLAUDE_PACKAGE_MANAGER 미설정
- 여러 잠금 파일로 감지 혼란

**해결 방법:**
```bash
# Set preferred package manager globally
export CLAUDE_PACKAGE_MANAGER=pnpm
# Add to ~/.bashrc or ~/.zshrc

# Or set per-project
echo '{"packageManager": "pnpm"}' > .claude/package-manager.json

# Or use package.json field
npm pkg set packageManager="pnpm@8.15.0"

# Warning: removing lock files can change installed dependency versions.
# Commit or back up the lock file first, then run a fresh install and re-run CI.
# Only do this when intentionally switching package managers.
rm package-lock.json  # If using pnpm/yarn/bun
```

---

## 성능 문제

### 느린 응답 시간

**증상:** Agent가 응답하는 데 30초 이상 소요

**원인:**
- 대용량 관찰 파일
- 너무 많은 활성 hook
- API로의 네트워크 지연

**해결 방법:**
```bash
# Archive large observations instead of deleting them
archive_dir="$HOME/.claude/homunculus/archive/$(date +%Y%m%d)"
mkdir -p "$archive_dir"
find ~/.claude/homunculus/projects -name "observations.jsonl" -size +10M -exec sh -c '
  for file do
    base=$(basename "$(dirname "$file")")
    gzip -c "$file" > "'"$archive_dir"'/${base}-observations.jsonl.gz"
    : > "$file"
  done
' sh {} +

# Disable unused hooks temporarily
# Edit ~/.claude/settings.json

# Keep active observation files small
# Large archives should live under ~/.claude/homunculus/archive/
```

### 높은 CPU 사용량

**증상:** Claude Code가 CPU 100% 사용

**원인:**
- 무한 관찰 루프
- 대용량 디렉토리에서 파일 감시
- hook에서 메모리 누수

**해결 방법:**
```bash
# Check for runaway processes
top -o cpu | grep claude

# Disable continuous learning temporarily
touch ~/.claude/homunculus/disabled

# Restart Claude Code
# Cmd/Ctrl+Q then reopen

# Check observation file size
du -sh ~/.claude/homunculus/*/
```

---

## 일반적인 오류 메시지

### "EACCES: permission denied"

```bash
# Fix hook permissions
find ~/.claude/plugins -name "*.sh" -exec chmod +x {} \;

# Fix observation directory permissions
chmod -R u+rwX,go+rX ~/.claude/homunculus
```

### "MODULE_NOT_FOUND"

```bash
# Install plugin dependencies
cd ~/.claude/plugins/cache/everything-claude-code
npm install

# Or for manual install
cd ~/.claude/plugins/ecc
npm install
```

### "spawn UNKNOWN"

```bash
# Windows-specific: Ensure scripts use correct line endings
# Convert CRLF to LF
find ~/.claude/plugins -name "*.sh" -exec dos2unix {} \;

# Or install dos2unix
# macOS: brew install dos2unix
# Ubuntu: sudo apt install dos2unix
```

---

## 도움 받기

문제가 지속되는 경우:

1. **GitHub 이슈 확인**: [github.com/affaan-m/everything-claude-code/issues](https://github.com/affaan-m/everything-claude-code/issues)
2. **디버그 로깅 활성화**:
   ```bash
   export CLAUDE_DEBUG=1
   export CLAUDE_LOG_LEVEL=debug
   ```
3. **진단 정보 수집**:
   ```bash
   claude --version
   node --version
   python3 --version
   echo $CLAUDE_PACKAGE_MANAGER
   ls -la ~/.claude/plugins/cache/
   ```
4. **이슈 열기**: 디버그 로그, 오류 메시지, 진단 정보 포함

---

## 관련 문서

- [README.md](./README.md) - 설치 및 기능
- [CONTRIBUTING.md](./CONTRIBUTING.md) - 개발 지침
- [docs/](./docs/) - 상세 문서
- [examples/](./examples/) - 사용 예시

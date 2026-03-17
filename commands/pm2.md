# PM2 Init

프로젝트를 자동 분석하고 PM2 서비스 커맨드를 생성합니다.

**커맨드**: `$ARGUMENTS`

---

## 워크플로우

1. PM2 확인 (없으면 `npm install -g pm2`로 설치)
2. 프로젝트를 스캔하여 서비스 식별 (frontend/backend/database)
3. 설정 파일 및 개별 커맨드 파일 생성

---

## 서비스 감지

| 유형 | 감지 | 기본 포트 |
|------|------|----------|
| Vite | vite.config.* | 5173 |
| Next.js | next.config.* | 3000 |
| Nuxt | nuxt.config.* | 3000 |
| CRA | package.json의 react-scripts | 3000 |
| Express/Node | server/backend/api 디렉토리 + package.json | 3000 |
| FastAPI/Flask | requirements.txt / pyproject.toml | 8000 |
| Go | go.mod / main.go | 8080 |

**포트 감지 우선순위**: 사용자 지정 > .env > 설정 파일 > scripts 인수 > 기본 포트

---

## 생성 파일

```
project/
├── ecosystem.config.cjs              # PM2 config
├── {backend}/start.cjs               # Python wrapper (if applicable)
└── .claude/
    ├── commands/
    │   ├── pm2-all.md                # Start all + monit
    │   ├── pm2-all-stop.md           # Stop all
    │   ├── pm2-all-restart.md        # Restart all
    │   ├── pm2-{port}.md             # Start single + logs
    │   ├── pm2-{port}-stop.md        # Stop single
    │   ├── pm2-{port}-restart.md     # Restart single
    │   ├── pm2-logs.md               # View all logs
    │   └── pm2-status.md             # View status
    └── scripts/
        ├── pm2-logs-{port}.ps1       # Single service logs
        └── pm2-monit.ps1             # PM2 monitor
```

---

## Windows 설정 (중요)

### ecosystem.config.cjs

**반드시 `.cjs` 확장자 사용**

```javascript
module.exports = {
  apps: [
    // Node.js (Vite/Next/Nuxt)
    {
      name: 'project-3000',
      cwd: './packages/web',
      script: 'node_modules/vite/bin/vite.js',
      args: '--port 3000',
      interpreter: 'C:/Program Files/nodejs/node.exe',
      env: { NODE_ENV: 'development' }
    },
    // Python
    {
      name: 'project-8000',
      cwd: './backend',
      script: 'start.cjs',
      interpreter: 'C:/Program Files/nodejs/node.exe',
      env: { PYTHONUNBUFFERED: '1' }
    }
  ]
}
```

**프레임워크 script 경로:**

| Framework | script | args |
|-----------|--------|------|
| Vite | `node_modules/vite/bin/vite.js` | `--port {port}` |
| Next.js | `node_modules/next/dist/bin/next` | `dev -p {port}` |
| Nuxt | `node_modules/nuxt/bin/nuxt.mjs` | `dev --port {port}` |
| Express | `src/index.js` 또는 `server.js` | - |

### Python 래퍼 스크립트 (start.cjs)

```javascript
const { spawn } = require('child_process');
const proc = spawn('python', ['-m', 'uvicorn', 'app.main:app', '--host', '0.0.0.0', '--port', '8000', '--reload'], {
  cwd: __dirname, stdio: 'inherit', windowsHide: true
});
proc.on('close', (code) => process.exit(code));
```

---

## 커맨드 파일 템플릿 (최소 내용)

### pm2-all.md (전체 시작 + monit)
````markdown
Start all services and open PM2 monitor.
```bash
cd "{PROJECT_ROOT}" && pm2 start ecosystem.config.cjs && start wt.exe -d "{PROJECT_ROOT}" pwsh -NoExit -c "pm2 monit"
```
````

### pm2-all-stop.md
````markdown
Stop all services.
```bash
cd "{PROJECT_ROOT}" && pm2 stop all
```
````

### pm2-all-restart.md
````markdown
Restart all services.
```bash
cd "{PROJECT_ROOT}" && pm2 restart all
```
````

### pm2-{port}.md (단일 시작 + 로그)
````markdown
Start {name} ({port}) and open logs.
```bash
cd "{PROJECT_ROOT}" && pm2 start ecosystem.config.cjs --only {name} && start wt.exe -d "{PROJECT_ROOT}" pwsh -NoExit -c "pm2 logs {name}"
```
````

### pm2-{port}-stop.md
````markdown
Stop {name} ({port}).
```bash
cd "{PROJECT_ROOT}" && pm2 stop {name}
```
````

### pm2-{port}-restart.md
````markdown
Restart {name} ({port}).
```bash
cd "{PROJECT_ROOT}" && pm2 restart {name}
```
````

### pm2-logs.md
````markdown
View all PM2 logs.
```bash
cd "{PROJECT_ROOT}" && pm2 logs
```
````

### pm2-status.md
````markdown
View PM2 status.
```bash
cd "{PROJECT_ROOT}" && pm2 status
```
````

### PowerShell 스크립트 (pm2-logs-{port}.ps1)
```powershell
Set-Location "{PROJECT_ROOT}"
pm2 logs {name}
```

### PowerShell 스크립트 (pm2-monit.ps1)
```powershell
Set-Location "{PROJECT_ROOT}"
pm2 monit
```

---

## 핵심 규칙

1. **설정 파일**: `ecosystem.config.cjs` (.js 아님)
2. **Node.js**: bin 경로 직접 지정 + interpreter
3. **Python**: Node.js 래퍼 스크립트 + `windowsHide: true`
4. **새 창 열기**: `start wt.exe -d "{path}" pwsh -NoExit -c "command"`
5. **최소 내용**: 각 커맨드 파일은 1-2줄 설명 + bash 블록만 포함
6. **직접 실행**: AI 파싱 불필요, bash 커맨드를 바로 실행

---

## 실행

`$ARGUMENTS`를 기반으로 init 실행:

1. 서비스를 위한 프로젝트 스캔
2. `ecosystem.config.cjs` 생성
3. Python 서비스를 위한 `{backend}/start.cjs` 생성 (해당하는 경우)
4. `.claude/commands/`에 커맨드 파일 생성
5. `.claude/scripts/`에 스크립트 파일 생성
6. **프로젝트 CLAUDE.md에 PM2 정보 업데이트** (아래 참고)
7. **터미널 커맨드와 함께 완료 요약 표시**

---

## Init 후: CLAUDE.md 업데이트

파일 생성 후, 프로젝트 `CLAUDE.md`에 PM2 섹션 추가 (없으면 생성):

````markdown
## PM2 Services

| Port | Name | Type |
|------|------|------|
| {port} | {name} | {type} |

**Terminal Commands:**
```bash
pm2 start ecosystem.config.cjs   # 처음 실행
pm2 start all                    # 이후 실행
pm2 stop all / pm2 restart all
pm2 start {name} / pm2 stop {name}
pm2 logs / pm2 status / pm2 monit
pm2 save                         # 프로세스 목록 저장
pm2 resurrect                    # 저장된 목록 복원
```
````

**CLAUDE.md 업데이트 규칙:**
- PM2 섹션이 존재하면 교체
- 없으면 끝에 추가
- 내용은 최소하고 핵심적으로 유지

---

## Init 후: 요약 표시

모든 파일 생성 후 출력:

```
## PM2 Init Complete

**Services:**

| Port | Name | Type |
|------|------|------|
| {port} | {name} | {type} |

**Claude Commands:** /pm2-all, /pm2-all-stop, /pm2-{port}, /pm2-{port}-stop, /pm2-logs, /pm2-status

**Terminal Commands:**
## First time (with config file)
pm2 start ecosystem.config.cjs && pm2 save

## After first time (simplified)
pm2 start all          # Start all
pm2 stop all           # Stop all
pm2 restart all        # Restart all
pm2 start {name}       # Start single
pm2 stop {name}        # Stop single
pm2 logs               # View logs
pm2 monit              # Monitor panel
pm2 resurrect          # Restore saved processes

**Tip:** Run `pm2 save` after first start to enable simplified commands.
```

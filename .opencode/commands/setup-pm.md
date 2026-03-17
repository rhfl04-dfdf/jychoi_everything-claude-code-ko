---
description: 패키지 매니저 설정 구성
agent: build
---

# Setup Package Manager 명령어

선호하는 패키지 매니저를 설정합니다: $ARGUMENTS

## 작업 내용

프로젝트 또는 전역 패키지 매니저 설정을 구성합니다.

## 감지 순서

1. **환경 변수**: `CLAUDE_PACKAGE_MANAGER`
2. **프로젝트 설정**: `.claude/package-manager.json`
3. **package.json**: `packageManager` 필드
4. **Lock 파일**: lock 파일에서 자동 감지
5. **전역 설정**: `~/.claude/package-manager.json`
6. **폴백**: 사용 가능한 첫 번째 매니저

## 설정 옵션

### 옵션 1: 환경 변수
```bash
export CLAUDE_PACKAGE_MANAGER=pnpm
```

### 옵션 2: 프로젝트 설정
```bash
# Create .claude/package-manager.json
echo '{"packageManager": "pnpm"}' > .claude/package-manager.json
```

### 옵션 3: package.json
```json
{
  "packageManager": "pnpm@8.0.0"
}
```

### 옵션 4: 전역 설정
```bash
# Create ~/.claude/package-manager.json
echo '{"packageManager": "yarn"}' > ~/.claude/package-manager.json
```

## 지원되는 패키지 매니저

| 매니저 | Lock 파일 | 명령어 |
|--------|-----------|--------|
| npm | package-lock.json | `npm install`, `npm run` |
| pnpm | pnpm-lock.yaml | `pnpm install`, `pnpm run` |
| yarn | yarn.lock | `yarn install`, `yarn run` |
| bun | bun.lockb | `bun install`, `bun run` |

## 확인

현재 설정 확인:
```bash
node scripts/setup-package-manager.js --detect
```

---

**팁**: 팀 전체 일관성을 위해 package.json에 `packageManager` 필드를 추가하세요.

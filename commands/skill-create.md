---
name: skill-create
description: 로컬 git 히스토리를 분석하여 코딩 패턴을 추출하고 SKILL.md 파일을 생성합니다. Skill Creator GitHub App의 로컬 버전입니다.
allowed_tools: ["Bash", "Read", "Write", "Grep", "Glob"]
---

# /skill-create - 로컬 Skill 생성

리포지토리의 git 히스토리를 분석하여 코딩 패턴을 추출하고, 팀의 관행을 Claude에게 가르치는 SKILL.md 파일을 생성합니다.

## 사용법

```bash
/skill-create                    # Analyze current repo
/skill-create --commits 100      # Analyze last 100 commits
/skill-create --output ./skills  # Custom output directory
/skill-create --instincts        # Also generate instincts for continuous-learning-v2
```

## 주요 기능

1. **Git 히스토리 파싱** - 커밋, 파일 변경 사항 및 패턴 분석
2. **패턴 감지** - 반복되는 workflow 및 규칙 식별
3. **SKILL.md 생성** - 유효한 Claude Code skill 파일 생성
4. **선택적 Instinct 생성** - continuous-learning-v2 시스템용

## 분석 단계

### 1단계: Git 데이터 수집

```bash
# Get recent commits with file changes
git log --oneline -n ${COMMITS:-200} --name-only --pretty=format:"%H|%s|%ad" --date=short

# Get commit frequency by file
git log --oneline -n 200 --name-only | grep -v "^$" | grep -v "^[a-f0-9]" | sort | uniq -c | sort -rn | head -20

# Get commit message patterns
git log --oneline -n 200 | cut -d' ' -f2- | head -50
```

### 2단계: 패턴 감지

다음과 같은 패턴 유형을 찾습니다:

| 패턴 | 감지 방법 |
|---------|-----------------|
| **Commit 규칙** | 커밋 메시지에 대한 정규식 (feat:, fix:, chore:) |
| **파일 동시 변경** | 항상 함께 변경되는 파일들 |
| **Workflow 시퀀스** | 반복되는 파일 변경 패턴 |
| **아키텍처** | 폴더 구조 및 명명 규칙 |
| **테스트 패턴** | 테스트 파일 위치, 명명, 커버리지 |

### 3단계: SKILL.md 생성

출력 형식:

```markdown
---
name: {repo-name}-patterns
description: Coding patterns extracted from {repo-name}
version: 1.0.0
source: local-git-analysis
analyzed_commits: {count}
---

# {Repo Name} Patterns

## Commit Conventions
{detected commit message patterns}

## Code Architecture
{detected folder structure and organization}

## Workflows
{detected repeating file change patterns}

## Testing Patterns
{detected test conventions}
```

### 4단계: Instinct 생성 (--instincts 옵션 사용 시)

continuous-learning-v2 연동용:

```yaml
---
id: {repo}-commit-convention
trigger: "when writing a commit message"
confidence: 0.8
domain: git
source: local-repo-analysis
---

# Use Conventional Commits

## Action
Prefix commits with: feat:, fix:, chore:, docs:, test:, refactor:

## Evidence
- Analyzed {n} commits
- {percentage}% follow conventional commit format
```

## 출력 예시

TypeScript 프로젝트에서 `/skill-create`를 실행하면 다음과 같은 결과가 생성될 수 있습니다:

```markdown
---
name: my-app-patterns
description: Coding patterns from my-app repository
version: 1.0.0
source: local-git-analysis
analyzed_commits: 150
---

# My App Patterns

## Commit Conventions

This project uses **conventional commits**:
- `feat:` - New features
- `fix:` - Bug fixes
- `chore:` - Maintenance tasks
- `docs:` - Documentation updates

## Code Architecture

```
src/
├── components/     # React components (PascalCase.tsx)
├── hooks/          # Custom hooks (use*.ts)
├── utils/          # Utility functions
├── types/          # TypeScript type definitions
└── services/       # API and external services
```

## Workflows

### Adding a New Component
1. Create `src/components/ComponentName.tsx`
2. Add tests in `src/components/__tests__/ComponentName.test.tsx`
3. Export from `src/components/index.ts`

### Database Migration
1. Modify `src/db/schema.ts`
2. Run `pnpm db:generate`
3. Run `pnpm db:migrate`

## Testing Patterns

- Test files: `__tests__/` directories or `.test.ts` suffix
- Coverage target: 80%+
- Framework: Vitest
```

## GitHub App 연동

고급 기능(10,000개 이상의 커밋, 팀 공유, 자동 PR 등)을 사용하려면 [Skill Creator GitHub App](https://github.com/apps/skill-creator)을 이용하세요:

- 설치: [github.com/apps/skill-creator](https://github.com/apps/skill-creator)
- issue에 `/skill-creator analyze` 코멘트 작성
- 생성된 skill이 포함된 PR 수신

## 관련 명령

- `/instinct-import` - 생성된 instinct 임포트
- `/instinct-status` - 학습된 instinct 확인
- `/evolve` - instinct를 skill/agent로 클러스터링

---

*[Everything Claude Code](https://github.com/affaan-m/everything-claude-code)의 일부*

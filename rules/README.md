# Rules
## 구조

Rule은 **common** 레이어와 **언어별** 디렉토리로 구성됩니다:

```
rules/
├── common/          # Language-agnostic principles (always install)
│   ├── coding-style.md
│   ├── git-workflow.md
│   ├── testing.md
│   ├── performance.md
│   ├── patterns.md
│   ├── hooks.md
│   ├── agents.md
│   └── security.md
├── typescript/      # TypeScript/JavaScript specific
├── python/          # Python specific
├── golang/          # Go specific
├── swift/           # Swift specific
└── php/             # PHP specific
```

- **common/**에는 범용 원칙이 포함됩니다 — 언어별 코드 예시 없음.
- **언어 디렉토리**는 프레임워크별 패턴, 도구, 코드 예시로 common rule을 확장합니다. 각 파일은 해당 common 파일을 참조합니다.

## 설치

### 옵션 1: 설치 스크립트 (권장)

```bash
# Install common + one or more language-specific rule sets
./install.sh typescript
./install.sh python
./install.sh golang
./install.sh swift
./install.sh php

# Install multiple languages at once
./install.sh typescript python
```

### 옵션 2: 수동 설치

> **중요:** 전체 디렉토리를 복사하세요 — `/*`로 평탄화하지 마세요.
> common과 언어별 디렉토리에는 동일한 이름의 파일이 포함됩니다.
> 하나의 디렉토리로 평탄화하면 언어별 파일이 common rule을 덮어쓰고,
> 언어별 파일에서 사용하는 상대 경로 `../common/` 참조가 깨집니다.

```bash
# Install common rules (required for all projects)
cp -r rules/common ~/.claude/rules/common

# Install language-specific rules based on your project's tech stack
cp -r rules/typescript ~/.claude/rules/typescript
cp -r rules/python ~/.claude/rules/python
cp -r rules/golang ~/.claude/rules/golang
cp -r rules/swift ~/.claude/rules/swift
cp -r rules/php ~/.claude/rules/php

# Attention ! ! ! Configure according to your actual project requirements; the configuration here is for reference only.
```

## Rule vs Skill

- **Rule**은 광범위하게 적용되는 표준, 컨벤션, 체크리스트를 정의합니다 (예: "80% 테스트 커버리지", "하드코딩된 시크릿 금지").
- **Skill** (`skills/` 디렉토리)은 특정 작업에 대한 심층적이고 실행 가능한 참고 자료를 제공합니다 (예: `python-patterns`, `golang-testing`).

언어별 rule 파일은 적절한 경우 관련 skill을 참조합니다. Rule은 *무엇을* 해야 하는지 알려주고, skill은 *어떻게* 하는지 알려줍니다.

## 새 언어 추가

새 언어 지원을 추가하려면 (예: `rust/`):

1. `rules/rust/` 디렉토리 생성
2. common rule을 확장하는 파일 추가:
   - `coding-style.md` — 포맷팅 도구, 관용구, 에러 처리 패턴
   - `testing.md` — 테스트 프레임워크, 커버리지 도구, 테스트 구성
   - `patterns.md` — 언어별 디자인 패턴
   - `hooks.md` — 포맷터, 린터, 타입 검사기를 위한 PostToolUse hook
   - `security.md` — 시크릿 관리, 보안 스캔 도구
3. 각 파일은 다음으로 시작해야 합니다:
   ```
   > This file extends [common/xxx.md](../common/xxx.md) with <Language> specific content.
   ```
4. 사용 가능한 skill을 참조하거나, `skills/` 아래에 새로운 것을 만드세요.

## Rule 우선순위

언어별 rule과 common rule이 충돌하는 경우, **언어별 rule이 우선합니다** (구체적인 것이 일반적인 것을 오버라이드). 이는 표준 레이어드 설정 패턴을 따릅니다 (CSS 특이성 또는 `.gitignore` 우선순위와 유사).

- `rules/common/`은 모든 프로젝트에 적용 가능한 범용 기본값을 정의합니다.
- `rules/golang/`, `rules/python/`, `rules/swift/`, `rules/php/`, `rules/typescript/` 등은 언어 관용구가 다른 경우 해당 기본값을 오버라이드합니다.

### 예시

`common/coding-style.md`는 기본 원칙으로 불변성을 권장합니다. 언어별 `golang/coding-style.md`는 이를 오버라이드할 수 있습니다:

> 관용적 Go는 구조체 변경에 포인터 수신자를 사용합니다 — 일반 원칙은 [common/coding-style.md](../common/coding-style.md)를 참조하세요. 단, Go 관용적 변경이 여기서는 선호됩니다.

### 오버라이드 참고가 있는 common rule

`rules/common/`의 rule 중 해당 패턴이 관용적이지 않은 언어의 언어별 파일에 의해 오버라이드될 수 있는 것들은 다음과 같이 표시됩니다:

> **언어 참고**: 이 rule은 해당 패턴이 관용적이지 않은 언어의 언어별 rule에 의해 오버라이드될 수 있습니다.

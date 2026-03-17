<div align="center">

# 🇰🇷 Everything Claude Code 한국어

**GitHub에서 가장 많이 Fork된 Claude Code 설정의 한국어 완역본**

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Original Repo Stars](https://img.shields.io/github/stars/affaan-m/everything-claude-code?style=flat&label=원본%20Stars&color=yellow)](https://github.com/affaan-m/everything-claude-code)
[![Korean](https://img.shields.io/badge/lang-한국어-red)](README.md)

<br/>

**16개 에이전트** · **65+ 스킬** · **40개 커맨드** · **자동화 훅** · **보안 스캐닝**

전부 한국어로. 코드 블록은 원본 그대로.

<br/>

[빠른 시작](#-빠른-시작) · [포함된 것들](#-뭐가-들어있나) · [가이드](#-가이드) · [설치](#-설치)

</div>

---

## 이게 뭔가요?

[everything-claude-code](https://github.com/affaan-m/everything-claude-code) (50K+ Stars)의 **한국어 완역본**입니다.

Claude Code, Cursor, Codex 등 AI 코딩 도구를 쓸 때 필요한 **에이전트, 스킬, 커맨드, 규칙, 훅, MCP 설정**이 전부 들어있고, 모든 문서가 한국어로 번역되어 있습니다.

> 원본 저작자: [Affaan Mustafa](https://github.com/affaan-m) · 원본 라이선스: MIT · Anthropic 해커톤 우승작

---

## 🚀 빠른 시작

**2분이면 됩니다.**

```bash
# 1. 클론
git clone https://github.com/rhfl04-dfdf/jychoi_everything-claude-code-ko.git
cd jychoi_everything-claude-code-ko

# 2. 설치 스크립트 실행 (원하는 언어 선택)
./install.sh typescript    # 또는 python, golang, kotlin, swift

# 3. Claude Code에서 바로 사용
/tdd                       # 테스트 주도 개발
/plan "기능 설명"           # 구현 계획
/code-review               # 코드 리뷰
/security-scan             # 보안 스캔
```

또는 원하는 것만 골라서 설치:

```bash
# 에이전트만
cp agents/*.md ~/.claude/agents/

# 스킬만
cp -r skills/* ~/.claude/skills/

# 커맨드만
cp commands/*.md ~/.claude/commands/

# 규칙만
cp -r rules/common/* ~/.claude/rules/
```

---

## 📦 뭐가 들어있나

| 구성 요소 | 개수 | 설명 |
|-----------|------|------|
| **에이전트** | 16개 | planner, code-reviewer, tdd-guide, security-reviewer, e2e-runner 등 |
| **스킬** | 65+ | 코딩 표준, API 설계, TDD, 보안, Spring Boot, Django, SwiftUI 등 |
| **커맨드** | 40개 | `/tdd`, `/plan`, `/e2e`, `/code-review`, `/build-fix` 등 |
| **규칙** | 언어별 | TypeScript, Python, Go, Kotlin, Swift, PHP, Perl |
| **훅** | 자동화 | 포매팅, 타입 체크, console.log 경고, tmux 리마인더 |
| **MCP 설정** | 14개 | GitHub, Supabase, Vercel, Railway, Cloudflare 등 |

### 에이전트 사용 예시

| 하고 싶은 것 | 커맨드 | 에이전트 |
|-------------|--------|---------|
| 새 기능 계획 | `/plan "인증 추가"` | planner |
| 테스트 먼저 작성 | `/tdd` | tdd-guide |
| 코드 리뷰 | `/code-review` | code-reviewer |
| 빌드 에러 수정 | `/build-fix` | build-error-resolver |
| E2E 테스트 | `/e2e` | e2e-runner |
| 보안 취약점 스캔 | `/security-scan` | security-reviewer |
| 죽은 코드 정리 | `/refactor-clean` | refactor-cleaner |

---

## 📖 가이드

모든 가이드가 한국어로 번역되어 있습니다.

| 가이드 | 내용 |
|--------|------|
| [요약 가이드](the-shortform-guide.md) | 설정, 기초, 스킬/훅/에이전트 개요 |
| [상세 가이드](the-longform-guide.md) | 토큰 최적화, 메모리 영속성, 병렬 처리, 서브에이전트 |
| [보안 가이드](the-security-guide.md) | 에이전트 보안, prompt injection 방어, 샌드박싱 |
| [OpenClaw 가이드](the-openclaw-guide.md) | 멀티채널 에이전트의 보안 위험 분석 |

---

## 💡 토큰 절약 팁

```json
{
  "model": "sonnet",
  "env": {
    "MAX_THINKING_TOKENS": "10000",
    "CLAUDE_AUTOCOMPACT_PCT_OVERRIDE": "50"
  }
}
```

| 설정 | 효과 |
|------|------|
| `model: sonnet` | ~60% 비용 절감 (80% 이상의 작업 처리 가능) |
| `MAX_THINKING_TOKENS: 10000` | 사고 비용 ~70% 절감 |
| `AUTOCOMPACT: 50` | 더 일찍 압축 → 긴 세션 품질 향상 |

복잡한 아키텍처/디버깅이 필요할 때만 `/model opus`로 전환하세요.

---

## 🌐 크로스 플랫폼

| 플랫폼 | 지원 |
|--------|------|
| **Claude Code** | 네이티브 (주 타겟) |
| **Cursor** | `.cursor/` 설정 제공 |
| **OpenCode** | `.opencode/` 플러그인 지원 |
| **Codex** | macOS 앱 + CLI 지원 |

---

## 📥 설치

### 옵션 1: 설치 스크립트 (권장)

```bash
git clone https://github.com/rhfl04-dfdf/jychoi_everything-claude-code-ko.git
cd jychoi_everything-claude-code-ko
./install.sh typescript    # python, golang, kotlin, swift 등
```

### 옵션 2: 수동 설치

```bash
git clone https://github.com/rhfl04-dfdf/jychoi_everything-claude-code-ko.git
cd jychoi_everything-claude-code-ko

# 원하는 것만 복사
cp agents/*.md ~/.claude/agents/
cp commands/*.md ~/.claude/commands/
cp -r skills/* ~/.claude/skills/
cp -r rules/common/* ~/.claude/rules/
cp -r rules/typescript/* ~/.claude/rules/   # 사용하는 언어 선택
```

---

## ❓ FAQ

<details>
<summary><b>영어 원본과 뭐가 다른가요?</b></summary>

모든 문서, 가이드, 에이전트 설명, 스킬 설명이 한국어로 번역되어 있습니다. 코드 블록(실제 코드, JSON, YAML, bash)은 원본 영어 그대로 유지되어 실행에 문제 없습니다.
</details>

<details>
<summary><b>원본이 업데이트되면 이것도 업데이트되나요?</b></summary>

원본 업데이트에 맞춰 주기적으로 동기화할 예정입니다. Issue나 PR로 요청해주셔도 됩니다.
</details>

<details>
<summary><b>기여하고 싶어요</b></summary>

번역 개선, 오타 수정, 새 스킬 추가 등 PR 환영합니다!
</details>

---

## 📄 라이선스

MIT License

원본: [affaan-m/everything-claude-code](https://github.com/affaan-m/everything-claude-code) by [Affaan Mustafa](https://github.com/affaan-m)

---

<div align="center">

**도움이 되었다면 ⭐ Star를 눌러주세요!**

한국 개발자들이 AI 코딩 도구를 더 잘 활용할 수 있도록.

</div>

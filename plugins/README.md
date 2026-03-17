# 플러그인과 마켓플레이스

플러그인은 Claude Code를 새로운 도구와 기능으로 확장합니다. 이 가이드는 설치만 다루며 — 사용 시기와 이유는 [전체 문서](https://x.com/affaanmustafa/status/2012378465664745795)를 참조하세요.

---

## 마켓플레이스

마켓플레이스는 설치 가능한 플러그인 저장소입니다.

### 마켓플레이스 추가

```bash
# Add official Anthropic marketplace
claude plugin marketplace add https://github.com/anthropics/claude-plugins-official

# Add community marketplaces (mgrep by @mixedbread-ai)
claude plugin marketplace add https://github.com/mixedbread-ai/mgrep
```

### 권장 마켓플레이스

| 마켓플레이스 | 소스 |
|-------------|--------|
| claude-plugins-official | `anthropics/claude-plugins-official` |
| claude-code-plugins | `anthropics/claude-code` |
| Mixedbread-Grep (@mixedbread-ai) | `mixedbread-ai/mgrep` |

---

## 플러그인 설치

```bash
# Open plugins browser
/plugins

# Or install directly
claude plugin install typescript-lsp@claude-plugins-official
```

### 권장 플러그인

**개발:**
- `typescript-lsp` - TypeScript 인텔리전스
- `pyright-lsp` - Python 타입 검사
- `hookify` - 대화형으로 hook 생성
- `code-simplifier` - 코드 리팩토링

**코드 품질:**
- `code-review` - 코드 리뷰
- `pr-review-toolkit` - PR 자동화
- `security-guidance` - 보안 검사

**검색:**
- `mgrep` - 향상된 검색 (ripgrep보다 우수)
- `context7` - 실시간 문서 조회

**워크플로우:**
- `commit-commands` - Git 워크플로우
- `frontend-design` - UI 패턴
- `feature-dev` - 기능 개발

---

## 빠른 설정

```bash
# Add marketplaces
claude plugin marketplace add https://github.com/anthropics/claude-plugins-official
claude plugin marketplace add https://github.com/mixedbread-ai/mgrep

# Open /plugins and install what you need
```

---

## 플러그인 파일 위치

```
~/.claude/plugins/
|-- cache/                    # Downloaded plugins
|-- installed_plugins.json    # Installed list
|-- known_marketplaces.json   # Added marketplaces
|-- marketplaces/             # Marketplace data
```

---
description: git 이력 분석을 통한 skill 생성
agent: build
---

# Skill Create 커맨드

git 이력을 분석하여 Claude Code skill 생성: $ARGUMENTS

## 작업 내용

1. **커밋 분석** - 이력에서 패턴 인식
2. **패턴 추출** - 일반적인 관행과 규칙
3. **SKILL.md 생성** - 구조화된 skill 문서
4. **instinct 생성** - continuous-learning-v2용

## 분석 프로세스

### 스텝 1: 커밋 데이터 수집
```bash
# Recent commits
git log --oneline -100

# Commits by file type
git log --name-only --pretty=format: | sort | uniq -c | sort -rn

# Most changed files
git log --pretty=format: --name-only | sort | uniq -c | sort -rn | head -20
```

### 스텝 2: 패턴 식별

**커밋 메시지 패턴**:
- 일반적인 접두사 (feat, fix, refactor)
- 명명 규칙
- 공동 저자 패턴

**코드 패턴**:
- 파일 구조 규칙
- import 구성
- 에러 처리 접근 방식

**리뷰 패턴**:
- 일반적인 리뷰 피드백
- 반복되는 수정 유형
- 품질 게이트

### 스텝 3: SKILL.md 생성

```markdown
# [Skill Name]

## Overview
[What this skill teaches]

## Patterns

### Pattern 1: [Name]
- When to use
- Implementation
- Example

### Pattern 2: [Name]
- When to use
- Implementation
- Example

## Best Practices

1. [Practice 1]
2. [Practice 2]
3. [Practice 3]

## Common Mistakes

1. [Mistake 1] - How to avoid
2. [Mistake 2] - How to avoid

## Examples

### Good Example
```[language]
// Code example
```

### Anti-pattern
```[language]
// What not to do
```
```

### 스텝 4: instinct 생성

continuous-learning-v2용:

```json
{
  "instincts": [
    {
      "trigger": "[situation]",
      "action": "[response]",
      "confidence": 0.8,
      "source": "git-history-analysis"
    }
  ]
}
```

## 출력

생성 파일:
- `skills/[name]/SKILL.md` - Skill 문서
- `skills/[name]/instincts.json` - instinct 컬렉션

---

**팁**: 지속적 학습을 위한 instinct도 함께 생성하려면 `/skill-create --instincts`를 실행하세요.

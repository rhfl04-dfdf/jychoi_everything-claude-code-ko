---
name: continuous-learning
description: Claude Code 세션에서 재사용 가능한 패턴을 자동으로 추출하고 향후 사용을 위한 학습된 skill로 저장합니다.
origin: ECC
---

# 지속적 학습 Skill

세션 종료 시 Claude Code 세션을 자동으로 평가하여 학습된 skill로 저장할 수 있는 재사용 가능한 패턴을 추출합니다.

## 활성화 시점

- Claude Code 세션에서 자동 패턴 추출을 설정할 때
- 세션 평가를 위한 Stop hook을 설정할 때
- `~/.claude/skills/learned/`의 학습된 skill을 검토하거나 관리할 때
- 추출 임계값이나 패턴 카테고리를 조정할 때
- v1(현재 방식)과 v2(instinct 기반) 접근 방식을 비교할 때

## 작동 방식

이 skill은 각 세션 종료 시 **Stop hook**으로 실행됩니다:

1. **세션 평가**: 세션에 충분한 메시지가 있는지 확인 (기본값: 10개 이상)
2. **패턴 감지**: 세션에서 추출 가능한 패턴을 식별
3. **Skill 추출**: 유용한 패턴을 `~/.claude/skills/learned/`에 저장

## 설정

`config.json`을 편집하여 사용자 정의할 수 있습니다:

```json
{
  "min_session_length": 10,
  "extraction_threshold": "medium",
  "auto_approve": false,
  "learned_skills_path": "~/.claude/skills/learned/",
  "patterns_to_detect": [
    "error_resolution",
    "user_corrections",
    "workarounds",
    "debugging_techniques",
    "project_specific"
  ],
  "ignore_patterns": [
    "simple_typos",
    "one_time_fixes",
    "external_api_issues"
  ]
}
```

## 패턴 유형

| 패턴 | 설명 |
|---------|-------------|
| `error_resolution` | 특정 오류가 해결된 방법 |
| `user_corrections` | 사용자 수정으로부터의 패턴 |
| `workarounds` | 프레임워크/라이브러리 특이사항에 대한 해결책 |
| `debugging_techniques` | 효과적인 디버깅 접근 방식 |
| `project_specific` | 프로젝트별 규칙 |

## Hook 설정

`~/.claude/settings.json`에 추가하세요:

```json
{
  "hooks": {
    "Stop": [{
      "matcher": "*",
      "hooks": [{
        "type": "command",
        "command": "~/.claude/skills/continuous-learning/evaluate-session.sh"
      }]
    }]
  }
}
```

## Stop Hook을 사용하는 이유

- **경량**: 세션 종료 시 한 번만 실행
- **비차단**: 모든 메시지에 지연을 추가하지 않음
- **완전한 컨텍스트**: 전체 세션 기록에 접근 가능

## 관련 자료

- [The Longform Guide](https://x.com/affaanmustafa/status/2014040193557471352) - 지속적 학습 섹션
- `/learn` command - 세션 중 수동 패턴 추출

---

## 비교 참고 (연구: 2025년 1월)

### vs Homunculus

Homunculus v2는 더 정교한 접근 방식을 취합니다:

| 기능 | 현재 접근 방식 | Homunculus v2 |
|---------|--------------|---------------|
| 관찰 | Stop hook (세션 종료 시) | PreToolUse/PostToolUse hook (100% 신뢰도) |
| 분석 | 메인 컨텍스트 | 백그라운드 agent (Haiku) |
| 세분화 수준 | 전체 skill | 원자적 "instinct" |
| 신뢰도 | 없음 | 0.3-0.9 가중치 |
| 발전 경로 | 직접 skill로 | instinct -> 클러스터 -> skill/command/agent |
| 공유 | 없음 | instinct 내보내기/가져오기 |

**Homunculus에서 얻은 핵심 통찰:**
> "v1은 관찰을 위해 skill에 의존했습니다. Skill은 확률적이며 약 50-80%의 확률로 작동합니다. v2는 관찰에 hook(100% 신뢰도)을 사용하고 instinct를 학습된 행동의 원자적 단위로 사용합니다."

### 잠재적 v2 개선사항

1. **Instinct 기반 학습** - 신뢰도 점수가 있는 더 작은 원자적 행동
2. **백그라운드 관찰자** - 병렬로 분석하는 Haiku agent
3. **신뢰도 감소** - 모순되면 instinct의 신뢰도가 하락
4. **도메인 태깅** - code-style, testing, git, debugging 등
5. **발전 경로** - 관련 instinct를 skill/command로 클러스터링

자세한 사양은 `docs/continuous-learning-v2-spec.md`를 참조하세요.

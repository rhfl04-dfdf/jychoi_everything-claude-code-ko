---
description: "Claude skill 및 command의 품질을 감사할 때 사용합니다. 변경된 skill만 확인하는 Quick Scan과 순차적 subagent 배치 평가를 포함하는 Full Stocktake 모드를 지원합니다."
origin: ECC
---

# skill-stocktake

품질 체크리스트와 AI의 종합적인 판단을 사용하여 모든 Claude skill 및 command를 감사하는 Slash command (`/skill-stocktake`)입니다. 최근 변경된 skill을 위한 Quick Scan과 전체 리뷰를 위한 Full Stocktake의 두 가지 모드를 지원합니다.

## Scope

이 command는 **실행된 디렉토리를 기준으로** 다음 경로를 대상으로 합니다.

| Path | Description |
|------|-------------|
| `~/.claude/skills/` | Global skills (모든 프로젝트) |
| `{cwd}/.claude/skills/` | 프로젝트 레벨 skills (디렉토리가 존재하는 경우) |

**Phase 1 시작 시, command는 어떤 경로가 발견되고 스캔되었는지 명시적으로 나열합니다.**

### 특정 프로젝트 타겟팅

프로젝트 레벨 skills를 포함하려면 해당 프로젝트의 루트 디렉토리에서 실행하세요.

```bash
cd ~/path/to/my-project
/skill-stocktake
```

프로젝트에 `.claude/skills/` 디렉토리가 없으면 global skills와 command만 평가됩니다.

## Modes

| Mode | Trigger | Duration |
|------|---------|---------|
| Quick Scan | `results.json` 존재 시 (기본값) | 5~10분 |
| Full Stocktake | `results.json`이 없거나 `/skill-stocktake full` 실행 시 | 20~30분 |

**결과 캐시:** `~/.claude/skills/skill-stocktake/results.json`

## Quick Scan Flow

마지막 실행 이후 변경된 skill만 재평가합니다 (5~10분 소요).

1. `~/.claude/skills/skill-stocktake/results.json` 읽기
2. 실행: `bash ~/.claude/skills/skill-stocktake/scripts/quick-diff.sh \
         ~/.claude/skills/skill-stocktake/results.json`
   (프로젝트 디렉토리는 `$PWD/.claude/skills`에서 자동 감지됩니다. 필요한 경우에만 명시적으로 전달하세요.)
3. 출력이 `[]`인 경우: "마지막 실행 이후 변경 사항 없음"을 보고하고 중단합니다.
4. 동일한 Phase 2 기준을 사용하여 변경된 파일만 재평가합니다.
5. 이전 결과에서 변경되지 않은 skill들을 그대로 가져옵니다.
6. diff만 출력합니다.
7. 실행: `bash ~/.claude/skills/skill-stocktake/scripts/save-results.sh \
         ~/.claude/skills/skill-stocktake/results.json <<< "$EVAL_RESULTS"`

## Full Stocktake Flow

### Phase 1 — Inventory

실행: `bash ~/.claude/skills/skill-stocktake/scripts/scan.sh`

스크립트는 skill 파일을 열거하고, frontmatter를 추출하며, UTC mtime을 수집합니다.
프로젝트 디렉토리는 `$PWD/.claude/skills`에서 자동 감지됩니다. 필요한 경우에만 명시적으로 전달하세요.
스크립트 출력에서 스캔 요약 및 inventory 테이블을 제시합니다.

```
Scanning:
  ✓ ~/.claude/skills/         (17 files)
  ✗ {cwd}/.claude/skills/    (not found — global skills only)
```

| Skill | 7d use | 30d use | Description |
|-------|--------|---------|-------------|

### Phase 2 — Quality Evaluation

전체 inventory와 체크리스트를 사용하여 Agent 도구 subagent (**general-purpose agent**)를 실행합니다.

```text
Agent(
  subagent_type="general-purpose",
  prompt="
Evaluate the following skill inventory against the checklist.

[INVENTORY]

[CHECKLIST]

Return JSON for each skill:
{ \"verdict\": \"Keep\"|\"Improve\"|\"Update\"|\"Retire\"|\"Merge into [X]\", \"reason\": \"...\" }
"
)
```

subagent는 각 skill을 읽고 체크리스트를 적용한 후, skill별 JSON을 반환합니다.

`{ "verdict": "Keep"|"Improve"|"Update"|"Retire"|"Merge into [X]", "reason": "..." }`

**Chunk 가이드:** context를 관리 가능한 수준으로 유지하기 위해 subagent 호출당 약 20개의 skill을 처리합니다. 각 chunk 이후 중간 결과를 `results.json` (`status: "in_progress"`)에 저장합니다.

모든 skill 평가가 완료되면 `status: "completed"`로 설정하고 Phase 3로 진행합니다.

**재개 감지:** 시작 시 `status: "in_progress"`가 발견되면 평가되지 않은 첫 번째 skill부터 재개합니다.

각 skill은 다음 체크리스트에 따라 평가됩니다.

```
- [ ] Content overlap with other skills checked
- [ ] Overlap with MEMORY.md / CLAUDE.md checked
- [ ] Freshness of technical references verified (use WebSearch if tool names / CLI flags / APIs are present)
- [ ] Usage frequency considered
```

평가 결과(Verdict) 기준:

| Verdict | Meaning |
|---------|---------|
| Keep | 유용하고 최신 상태임 |
| Improve | 유지할 가치가 있으나, 구체적인 개선이 필요함 |
| Update | 참조된 기술이 구형임 (WebSearch로 확인 필요) |
| Retire | 품질이 낮거나, 오래되었거나, 비용 대비 효율이 낮음 |
| Merge into [X] | 다른 skill과 내용이 상당히 겹침; 병합 대상의 이름을 지정함 |

평가는 수치적 루브릭이 아닌 **AI의 종합적인 판단**입니다. 가이드가 되는 관점은 다음과 같습니다.
- **실행 가능성(Actionability)**: 즉시 실행할 수 있는 코드 예제, command 또는 단계 포함 여부
- **범위 적합성(Scope fit)**: 이름, trigger, 내용의 일치 여부; 너무 넓거나 좁지 않음
- **독창성(Uniqueness)**: MEMORY.md / CLAUDE.md / 다른 skill로 대체할 수 없는 가치
- **최신성(Currency)**: 기술 참조가 현재 환경에서 작동함

**Reason 품질 요구 사항** — `reason` 필드는 그 자체로 완결성을 가져야 하며 의사 결정을 지원할 수 있어야 합니다.
- "변경 없음"이라고만 적지 마세요 — 항상 핵심 증거를 다시 기술하세요.
- **Retire**의 경우: (1) 어떤 구체적인 결함이 발견되었는지, (2) 대신 무엇이 동일한 요구를 충족하는지 기술하세요.
  - 나쁨: `"Superseded"`
  - 좋음: `"disable-model-invocation: true already set; superseded by continuous-learning-v2 which covers all the same patterns plus confidence scoring. No unique content remains."`
- **Merge**의 경우: 대상 이름을 지정하고 통합할 내용을 설명하세요.
  - 나쁨: `"Overlaps with X"`
  - 좋음: `"42-line thin content; Step 4 of chatlog-to-article already covers the same workflow. Integrate the 'article angle' tip as a note in that skill."`
- **Improve**의 경우: 필요한 구체적인 변경 사항을 설명하세요 (어느 섹션, 어떤 작업, 해당되는 경우 대상 크기).
  - 나쁨: `"Too long"`
  - 좋음: `"276 lines; Section 'Framework Comparison' (L80–140) duplicates ai-era-architecture-principles; delete it to reach ~150 lines."`
- **Keep**의 경우 (Quick Scan에서 mtime만 변경된 경우): "변경 없음"이라고 적지 말고 원래의 평가 근거를 다시 기술하세요.
  - 나쁨: `"Unchanged"`
  - 좋음: `"mtime updated but content unchanged. Unique Python reference explicitly imported by rules/python/; no overlap found."`

### Phase 3 — Summary Table

| Skill | 7d use | Verdict | Reason |
|-------|--------|---------|--------|

### Phase 4 — Consolidation

1. **Retire / Merge**: 사용자에게 확인하기 전에 파일별로 상세한 정당성을 제시합니다.
   - 어떤 구체적인 문제(중복, 노후화, 끊어진 참조 등)가 발견되었는지
   - 어떤 대안이 동일한 기능을 제공하는지 (Retire의 경우: 어떤 기존 skill/rule인지; Merge의 경우: 대상 파일과 통합할 내용)
   - 제거 시 영향 (의존하는 skill, MEMORY.md 참조 또는 영향을 받는 workflow)
2. **Improve**: 근거와 함께 구체적인 개선 제안을 제시합니다.
   - 무엇을 왜 변경해야 하는지 (예: "X/Y 섹션이 python-patterns와 중복되므로 430→200라인으로 축소")
   - 사용자가 실행 여부를 결정합니다.
3. **Update**: 소스가 확인된 업데이트된 내용을 제시합니다.
4. MEMORY.md 라인 수를 확인하고, 100라인을 초과할 경우 압축을 제안합니다.

## Results File Schema

`~/.claude/skills/skill-stocktake/results.json`:

**`evaluated_at`**: 평가 완료 시점의 실제 UTC 시간으로 설정되어야 합니다.
Bash를 통해 획득: `date -u +%Y-%m-%dT%H:%M:%SZ`. `T00:00:00Z`와 같은 날짜 전용 근사치를 사용하지 마세요.

```json
{
  "evaluated_at": "2026-02-21T10:00:00Z",
  "mode": "full",
  "batch_progress": {
    "total": 80,
    "evaluated": 80,
    "status": "completed"
  },
  "skills": {
    "skill-name": {
      "path": "~/.claude/skills/skill-name/SKILL.md",
      "verdict": "Keep",
      "reason": "Concrete, actionable, unique value for X workflow",
      "mtime": "2026-01-15T08:30:00Z"
    }
  }
}
```

## Notes

- 평가는 블라인드로 진행됩니다: 출처(ECC, 직접 작성, 자동 추출)에 관계없이 모든 skill에 동일한 체크리스트가 적용됩니다.
- 보관(Archive) / 삭제 작업은 항상 사용자의 명시적인 확인이 필요합니다.
- skill 출처에 따른 평가 결과 차등은 없습니다.

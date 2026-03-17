### 플러그인 매니페스트 주의사항

`.claude-plugin/plugin.json`을 편집할 계획이라면, Claude 플러그인 검증기가 모호한 오류(예: `agents: Invalid input`)로 설치를 실패하게 만들 수 있는 **문서화되지 않았지만 엄격한 제약사항**을 강제한다는 점을 알아야 합니다. 특히, 컴포넌트 필드는 배열이어야 하고, `agents`는 디렉토리가 아닌 명시적인 파일 경로를 사용해야 하며, 신뢰할 수 있는 검증과 설치를 위해 `version` 필드가 필요합니다.

이러한 제약사항은 공개 예시에서 명확하지 않으며 과거에 반복적인 설치 실패를 유발했습니다. 자세한 내용은 플러그인 매니페스트를 변경하기 전에 반드시 검토해야 하는 `.claude-plugin/PLUGIN_SCHEMA_NOTES.md`에 문서화되어 있습니다.

### 커스텀 엔드포인트 및 게이트웨이

ECC는 Claude Code 전송 설정을 재정의하지 않습니다. Claude Code가 공식 LLM 게이트웨이 또는 호환 가능한 커스텀 엔드포인트를 통해 실행되도록 설정된 경우, CLI가 성공적으로 시작된 후 hook, command, skill이 로컬에서 실행되므로 플러그인은 계속 작동합니다.

전송 선택을 위해 Claude Code 자체 환경/설정을 사용하세요. 예를 들어:

```bash
export ANTHROPIC_BASE_URL=https://your-gateway.example.com
export ANTHROPIC_AUTH_TOKEN=your-token
claude
```

---
paths:
  - "**/*.php"
  - "**/composer.json"
  - "**/phpstan.neon"
  - "**/phpstan.neon.dist"
  - "**/psalm.xml"
---
# PHP Hooks

> 이 파일은 [common/hooks.md](../common/hooks.md)를 확장하여 PHP 특화 내용을 다룹니다.

## PostToolUse Hooks

`~/.claude/settings.json`에서 설정:

- **Pint / PHP-CS-Fixer**: 편집된 `.php` 파일 자동 포맷팅
- **PHPStan / Psalm**: 타입이 지정된 코드베이스에서 PHP 편집 후 정적 분석 실행
- **PHPUnit / Pest**: 편집이 동작에 영향을 미칠 때 수정된 파일 또는 모듈에 대한 대상 테스트 실행

## 경고

- 편집된 파일에 남아있는 `var_dump`, `dd`, `dump`, `die()` 경고
- 편집된 PHP 파일에 raw SQL이 추가되거나 CSRF/세션 보호가 비활성화될 때 경고

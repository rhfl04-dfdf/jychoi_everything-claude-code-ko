---
paths:
  - "**/*.php"
  - "**/phpunit.xml"
  - "**/phpunit.xml.dist"
  - "**/composer.json"
---
# PHP 테스팅

> 이 파일은 [common/testing.md](../common/testing.md)를 확장하여 PHP 특화 내용을 다룹니다.

## 프레임워크

기본 테스트 프레임워크로 **PHPUnit** 사용. 프로젝트에서 이미 사용 중인 경우 **Pest**도 허용됩니다.

## 커버리지

```bash
vendor/bin/phpunit --coverage-text
# 또는
vendor/bin/pest --coverage
```

CI에서는 **pcov** 또는 **Xdebug**를 선호하고, 커버리지 임계값을 암묵적 지식이 아닌 CI에서 관리합니다.

## 테스트 구성

- 빠른 단위 테스트와 프레임워크/데이터베이스 통합 테스트를 분리
- 대형 수동 배열 대신 픽스처에 factory/builder 사용
- HTTP/컨트롤러 테스트는 전송 및 검증에 집중; 비즈니스 규칙은 서비스 레벨 테스트로 이동

## 참고

저장소 전체 RED -> GREEN -> REFACTOR 루프는 skill: `tdd-workflow`를 참조하세요.

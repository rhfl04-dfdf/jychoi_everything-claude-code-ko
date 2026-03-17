---
paths:
  - "**/*.pl"
  - "**/*.pm"
  - "**/*.t"
  - "**/*.psgi"
  - "**/*.cgi"
---
# Perl 테스팅

> 이 파일은 [common/testing.md](../common/testing.md)를 확장하여 Perl 특화 내용을 다룹니다.

## 프레임워크

새 프로젝트에는 **Test2::V0** 사용 (Test::More 대신):

```perl
use Test2::V0;

is($result, 42, 'answer is correct');

done_testing;
```

## 실행기

```bash
prove -l t/              # adds lib/ to @INC
prove -lr -j8 t/         # recursive, 8 parallel jobs
```

`lib/`이 `@INC`에 있도록 항상 `-l` 플래그 사용.

## 커버리지

**Devel::Cover** 사용 — 80% 이상 목표:

```bash
cover -test
```

## Mocking

- **Test::MockModule** — 기존 모듈의 메서드 mock
- **Test::MockObject** — 처음부터 테스트 더블 생성

## 주의사항

- 항상 테스트 파일을 `done_testing`으로 끝내기
- `prove`에서 `-l` 플래그 누락 금지

## 참고

Test2::V0, prove, Devel::Cover를 사용한 상세한 Perl TDD 패턴은 skill: `perl-testing`을 참조하세요.

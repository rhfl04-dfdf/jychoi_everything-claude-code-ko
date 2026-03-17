---
paths:
  - "**/*.pl"
  - "**/*.pm"
  - "**/*.t"
  - "**/*.psgi"
  - "**/*.cgi"
---
# Perl 코딩 스타일

> 이 파일은 [common/coding-style.md](../common/coding-style.md)를 확장하여 Perl 특화 내용을 다룹니다.

## 표준

- 항상 `use v5.36` 사용 (`strict`, `warnings`, `say`, 서브루틴 시그니처 활성화)
- 서브루틴 시그니처 사용 — `@_`를 수동으로 unpack하지 않기
- 명시적 개행이 있는 `print`보다 `say` 선호

## 불변성

- 모든 속성에 `is => 'ro'`와 `Types::Standard`를 사용하는 **Moo** 사용
- blessed hashref를 직접 사용하지 않기 — 항상 Moo/Moose 접근자 사용
- **OO 재정의 참고**: `builder` 또는 `default`가 있는 Moo `has` 속성은 읽기 전용 계산 값에 허용

## 포맷팅

다음 설정으로 **perltidy** 사용:

```
-i=4    # 4칸 들여쓰기
-l=100  # 100자 줄 길이
-ce     # cuddled else
-bar    # 여는 중괄호 항상 오른쪽
```

## 린팅

`core`, `pbp`, `security` 테마와 함께 severity 3으로 **perlcritic** 사용.

```bash
perlcritic --severity 3 --theme 'core || pbp || security' lib/
```

## 참고

포괄적인 현대 Perl 관용구 및 모범 사례는 skill: `perl-patterns`를 참조하세요.

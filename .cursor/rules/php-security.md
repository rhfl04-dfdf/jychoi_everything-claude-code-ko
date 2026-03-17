---
paths:
  - "**/*.php"
  - "**/composer.lock"
  - "**/composer.json"
---
# PHP 보안

> 이 파일은 [common/security.md](../common/security.md)를 확장하여 PHP 특화 내용을 다룹니다.

## 입력 및 출력

- 프레임워크 경계에서 요청 입력 검증 (`FormRequest`, Symfony Validator, 또는 명시적 DTO 검증)
- 템플릿에서 기본적으로 출력 이스케이프; raw HTML 렌더링은 정당성이 입증되어야 하는 예외로 처리
- 검증 없이 쿼리 파라미터, 쿠키, 헤더, 업로드된 파일 메타데이터를 절대 신뢰하지 않기

## 데이터베이스 안전성

- 모든 동적 쿼리에 prepared statement (`PDO`, Doctrine, Eloquent query builder) 사용
- 컨트롤러/뷰에서 SQL 문자열 조합 금지
- ORM 대량 할당을 신중하게 범위 지정하고 쓰기 가능한 필드 허용 목록 관리

## 시크릿 및 의존성

- 커밋된 설정 파일이 아닌 환경 변수 또는 시크릿 매니저에서 시크릿 로드
- CI에서 `composer audit` 실행하고 의존성 추가 전에 새 패키지 관리자 신뢰도 검토
- 주요 버전을 신중하게 고정하고 폐기된 패키지 신속히 제거

## 인증 및 세션 안전성

- 비밀번호 저장에 `password_hash()` / `password_verify()` 사용
- 인증 및 권한 변경 후 세션 식별자 재생성
- 상태를 변경하는 웹 요청에 CSRF 보호 적용

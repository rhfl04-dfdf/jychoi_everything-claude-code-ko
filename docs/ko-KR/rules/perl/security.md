---
paths:
  - "**/*.pl"
  - "**/*.pm"
  - "**/*.t"
  - "**/*.psgi"
  - "**/*.cgi"
---
# Perl 보안

> 이 파일은 [common/security.md](../common/security.md)를 확장하여 Perl 특화 내용을 다룹니다.

## Taint 모드

- 모든 CGI/웹 대면 스크립트에 `-T` 플래그 사용
- 외부 명령 실행 전에 `%ENV` (`$ENV{PATH}`, `$ENV{CDPATH}` 등) 정제

## 입력 검증

- 허용 목록(allowlist) 정규식으로 오염 제거 — `/(.*)/s`는 절대 사용 금지
- 명시적 패턴으로 모든 사용자 입력 검증:

```perl
if ($input =~ /\A([a-zA-Z0-9_-]+)\z/) {
    my $clean = $1;
}
```

## 파일 I/O

- **3인수 open만 사용** — 2인수 open 금지
- `Cwd::realpath`로 경로 탐색 방지:

```perl
use Cwd 'realpath';
my $safe_path = realpath($user_path);
die "Path traversal" unless $safe_path =~ m{\A/allowed/directory/};
```

## 프로세스 실행

- **리스트 형식의 `system()` 사용** — 단일 문자열 형식 금지
- 출력 캡처에는 **IPC::Run3** 사용
- 변수 보간이 있는 백틱 절대 사용 금지

```perl
system('grep', '-r', $pattern, $directory);  # 안전
```

## SQL 인젝션 방지

항상 DBI 플레이스홀더 사용 — SQL에 변수 직접 보간 금지:

```perl
my $sth = $dbh->prepare('SELECT * FROM users WHERE email = ?');
$sth->execute($email);
```

## 보안 스캐닝

security 테마로 severity 4 이상에서 **perlcritic** 실행:

```bash
perlcritic --severity 4 --theme security lib/
```

## 참고

포괄적인 Perl 보안 패턴, taint 모드, 안전한 I/O는 skill: `perl-security`를 참조하세요.

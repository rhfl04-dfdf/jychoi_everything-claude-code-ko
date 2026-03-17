---
paths:
  - "**/*.pl"
  - "**/*.pm"
  - "**/*.t"
  - "**/*.psgi"
  - "**/*.cgi"
---
# Perl 패턴

> 이 파일은 [common/patterns.md](../common/patterns.md)를 확장하여 Perl 특화 내용을 다룹니다.

## Repository 패턴

인터페이스 뒤에 **DBI** 또는 **DBIx::Class** 사용:

```perl
package MyApp::Repo::User;
use Moo;

has dbh => (is => 'ro', required => 1);

sub find_by_id ($self, $id) {
    my $sth = $self->dbh->prepare('SELECT * FROM users WHERE id = ?');
    $sth->execute($id);
    return $sth->fetchrow_hashref;
}
```

## DTO / 값 객체

**Types::Standard**와 함께 **Moo** 클래스 사용 (Python dataclass와 동등):

```perl
package MyApp::DTO::User;
use Moo;
use Types::Standard qw(Str Int);

has name  => (is => 'ro', isa => Str, required => 1);
has email => (is => 'ro', isa => Str, required => 1);
has age   => (is => 'ro', isa => Int);
```

## 리소스 관리

- 항상 `autodie`와 함께 **3인수 open** 사용
- 파일 작업에는 **Path::Tiny** 사용

```perl
use autodie;
use Path::Tiny;

my $content = path('config.json')->slurp_utf8;
```

## 모듈 인터페이스

`@EXPORT_OK`와 함께 `Exporter 'import'` 사용 — `@EXPORT`는 절대 사용 금지:

```perl
use Exporter 'import';
our @EXPORT_OK = qw(parse_config validate_input);
```

## 의존성 관리

재현 가능한 설치를 위해 **cpanfile** + **carton** 사용:

```bash
carton install
carton exec prove -lr t/
```

## 참고

포괄적인 현대 Perl 패턴 및 관용구는 skill: `perl-patterns`를 참조하세요.

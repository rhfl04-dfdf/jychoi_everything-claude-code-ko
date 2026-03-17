---
name: perl-patterns
description: 견고하고 유지보수가 용이한 Perl 애플리케이션 구축을 위한 모던 Perl 5.36+ 이디엄, best practice 및 컨벤션입니다.
origin: ECC
---

# 모던 Perl 개발 패턴 (Modern Perl Development Patterns)

견고하고 유지보수가 용이한 애플리케이션 구축을 위한 관용적인(Idiomatic) Perl 5.36+ 패턴과 best practice입니다.

## 활성화 시점

- 새로운 Perl 코드 또는 모듈을 작성할 때
- Perl 코드가 이디엄을 준수하는지 리뷰할 때
- 레거시 Perl을 모던 표준으로 refactor할 때
- Perl 모듈 아키텍처를 설계할 때
- 5.36 이전 코드를 모던 Perl로 마이그레이션할 때

## 작동 방식

시그니처(signatures), 명시적 모듈, 집중된 에러 핸들링, 테스트 가능한 경계와 같은 모던 Perl 5.36+ 기본값을 우선적으로 적용합니다. 아래 예시들은 시작점으로 복사하여 실제 앱, 종속성 스택 및 배포 모델에 맞게 조정하여 사용하도록 설계되었습니다.

## 핵심 원칙

### 1. `v5.36` Pragma 사용

단일 `use v5.36` 선언은 오래된 상용구(boilerplate)를 대체하며 strict, warnings 및 서브루틴 시그니처를 활성화합니다.

```perl
# Good: 모던 서문
use v5.36;

sub greet($name) {
    say "Hello, $name!";
}

# Bad: 레거시 상용구
use strict;
use warnings;
use feature 'say', 'signatures';
no warnings 'experimental::signatures';

sub greet {
    my ($name) = @_;
    say "Hello, $name!";
}
```

### 2. 서브루틴 시그니처 (Subroutine Signatures)

명확성과 자동 인수 개수(arity) 체크를 위해 시그니처를 사용합니다.

```perl
use v5.36;

# Good: 기본값이 있는 시그니처
sub connect_db($host, $port = 5432, $timeout = 30) {
    # $host는 필수이며, 나머지는 기본값이 있습니다.
    return DBI->connect("dbi:Pg:host=$host;port=$port", undef, undef, {
        RaiseError => 1,
        PrintError => 0,
    });
}

# Good: 가변 인수를 위한 Slurpy 파라미터
sub log_message($level, @details) {
    say "[$level] " . join(' ', @details);
}

# Bad: 수동 인수 언팩킹
sub connect_db {
    my ($host, $port, $timeout) = @_;
    $port    //= 5432;
    $timeout //= 30;
    # ...
}
```

### 3. 컨텍스트 민감도 (Context Sensitivity)

Perl의 핵심 개념인 스칼라(scalar) vs 리스트(list) 컨텍스트를 이해해야 합니다.

```perl
use v5.36;

my @items = (1, 2, 3, 4, 5);

my @everything  = @items;            # 리스트 컨텍스트: 모든 요소
my $count = @items;            # 스칼라 컨텍스트: 개수 (5)
say "Items: " . scalar @items; # 스칼라 컨텍스트 강제
```

### 4. Postfix Dereferencing

중첩된 구조에서 가독성을 높이기 위해 postfix dereference 구문을 사용합니다.

```perl
use v5.36;

my $data = {
    users => [
        { name => 'Alice', roles => ['admin', 'user'] },
        { name => 'Bob',   roles => ['user'] },
    ],
};

# Good: Postfix dereferencing
my @users = $data->{users}->@*;
my @roles = $data->{users}[0]{roles}->@*;
my %first = $data->{users}[0]->%*;

# Bad: Circumfix dereferencing (체이닝 시 읽기 어려움)
my @users = @{ $data->{users} };
my @roles = @{ $data->{users}[0]{roles} };
```

### 5. `isa` 연산자 (5.32+)

Infix 타입 체크 — `blessed($o) && $o->isa('X')`를 대체합니다.

```perl
use v5.36;
if ($obj isa 'My::Class') { $obj->do_something }
```

## 에러 핸들링

### eval/die 패턴

```perl
use v5.36;

sub parse_config($path) {
    my $content = eval { path($path)->slurp_utf8 };
    die "Config error: $@" if $@;
    return decode_json($content);
}
```

### Try::Tiny (신뢰할 수 있는 예외 처리)

```perl
use v5.36;
use Try::Tiny;

sub fetch_user($id) {
    my $user = try {
        $db->resultset('User')->find($id)
            // die "User $id not found\n";
    }
    catch {
        warn "Failed to fetch user $id: $_";
        undef;
    };
    return $user;
}
```

### 네이티브 try/catch (5.40+)

```perl
use v5.40;

sub divide($x, $y) {
    try {
        die "Division by zero" if $y == 0;
        return $x / $y;
    }
    catch ($e) {
        warn "Error: $e";
        return;
    }
}
```

## Moo를 이용한 모던 OO

가볍고 모던한 OO를 위해 Moo를 선호합니다. 메타프로토콜이 필요한 경우에만 Moose를 사용하세요.

```perl
# Good: Moo 클래스
package User;
use Moo;
use Types::Standard qw(Str Int ArrayRef);
use namespace::autoclean;

has name  => (is => 'ro', isa => Str, required => 1);
has email => (is => 'ro', isa => Str, required => 1);
has age   => (is => 'ro', isa => Int, default  => sub { 0 });
has roles => (is => 'ro', isa => ArrayRef[Str], default => sub { [] });

sub is_admin($self) {
    return grep { $_ eq 'admin' } $self->roles->@*;
}

sub greet($self) {
    return "Hello, I'm " . $self->name;
}

1;

# 사용법
my $user = User->new(
    name  => 'Alice',
    email => 'alice@example.com',
    roles => ['admin', 'user'],
);

# Bad: Blessed hashref (유효성 검사 및 접근자 없음)
package User;
sub new {
    my ($class, %args) = @_;
    return bless \%args, $class;
}
sub name { return $_[0]->{name} }
1;
```

### Moo Roles

```perl
package Role::Serializable;
use Moo::Role;
use JSON::MaybeXS qw(encode_json);
requires 'TO_HASH';
sub to_json($self) { encode_json($self->TO_HASH) }
1;

package User;
use Moo;
with 'Role::Serializable';
has name  => (is => 'ro', required => 1);
has email => (is => 'ro', required => 1);
sub TO_HASH($self) { { name => $self->name, email => $self->email } }
1;
```

### 네이티브 `class` 키워드 (5.38+, Corinna)

```perl
use v5.38;
use feature 'class';
no warnings 'experimental::class';

class Point {
    field $x :param;
    field $y :param;
    method magnitude() { sqrt($x**2 + $y**2) }
}

my $p = Point->new(x => 3, y => 4);
say $p->magnitude;  # 5
```

## 정규 표현식 (Regular Expressions)

### Named Captures 및 `/x` 플래그

```perl
use v5.36;

# Good: 가독성을 위해 /x와 함께 Named captures 사용
my $log_re = qr{
    ^ (?<timestamp> \d{4}-\d{2}-\d{2} \s \d{2}:\d{2}:\d{2} )
    \s+ \[ (?<level> \w+ ) \]
    \s+ (?<message> .+ ) $
}x;

if ($line =~ $log_re) {
    say "Time: $+{timestamp}, Level: $+{level}";
    say "Message: $+{message}";
}

# Bad: 위치 기반 캡처 (유지보수가 어려움)
if ($line =~ /^(\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2})\s+\[(\w+)\]\s+(.+)$/) {
    say "Time: $1, Level: $2";
}
```

### 미리 컴파일된 패턴

```perl
use v5.36;

# Good: 한 번 컴파일하고 여러 번 사용
my $email_re = qr/^[A-Za-z0-9._%+-]+\@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$/;

sub validate_emails(@emails) {
    return grep { $_ =~ $email_re } @emails;
}
```

## 데이터 구조

### 참조(References) 및 안전한 딥 액세스

```perl
use v5.36;

# 해시 및 배열 참조
my $config = {
    database => {
        host => 'localhost',
        port => 5432,
        options => ['utf8', 'sslmode=require'],
    },
};

# 안전한 딥 액세스 (어느 단계라도 누락되면 undef 반환)
my $port = $config->{database}{port};           # 5432
my $missing = $config->{cache}{host};           # undef, 에러 없음

# 해시 슬라이스
my %subset;
@subset{qw(host port)} = @{$config->{database}}{qw(host port)};

# 배열 슬라이스
my @first_two = $config->{database}{options}->@[0, 1];

# 다중 변수 for 루프 (5.36에서 실험적, 5.40에서 안정화)
use feature 'for_list';
no warnings 'experimental::for_list';
for my ($key, $val) (%$config) {
    say "$key => $val";
}
```

## 파일 I/O

### 3-인자 Open

```perl
use v5.36;

# Good: autodie와 함께 3-인자 open 사용 (코어 모듈, 'or die' 제거 가능)
use autodie;

sub read_file($path) {
    open my $fh, '<:encoding(UTF-8)', $path;
    local $/;
    my $content = <$fh>;
    close $fh;
    return $content;
}

# Bad: 2-인자 open (쉘 인젝션 위험, perl-security 참조)
open FH, $path;            # 절대 하지 마세요
open FH, "< $path";        # 여전히 좋지 않음 — 모드 문자열에 사용자 데이터 포함
```

### 파일 작업을 위한 Path::Tiny

```perl
use v5.36;
use Path::Tiny;

my $file = path('config', 'app.json');
my $content = $file->slurp_utf8;
$file->spew_utf8($new_content);

# 디렉토리 반복
for my $child (path('src')->children(qr/\.pl$/)) {
    say $child->basename;
}
```

## 모듈 구조화

### 표준 프로젝트 레이아웃

```text
MyApp/
├── lib/
│   └── MyApp/
│       ├── App.pm           # 메인 모듈
│       ├── Config.pm        # 설정
│       ├── DB.pm            # 데이터베이스 레이어
│       └── Util.pm          # 유틸리티
├── bin/
│   └── myapp                # 엔트리 포인트 스크립트
├── t/
│   ├── 00-load.t            # 컴파일 테스트
│   ├── unit/                # 유닛 테스트
│   └── integration/         # 통합 테스트
├── cpanfile                 # 종속성
├── Makefile.PL              # 빌드 시스템
└── .perlcriticrc            # 린팅 설정
```

### Exporter 패턴

```perl
package MyApp::Util;
use v5.36;
use Exporter 'import';

our @EXPORT_OK   = qw(trim);
our %EXPORT_TAGS = (all => \@EXPORT_OK);

sub trim($str) { $str =~ s/^\s+|\s+$//gr }

1;
```

## 도구 (Tooling)

### perltidy 설정 (.perltidyrc)

```text
-i=4        # 4-스페이스 들여쓰기
-l=100      # 100자 라인 길이
-ci=4       # 연속된 라인 들여쓰기
-ce         # else를 닫는 중괄호와 같은 줄에 (cuddled else)
-bar        # 여는 중괄호를 같은 줄에
-nolq       # 긴 따옴표 문자열을 내어쓰지 않음
```

### perlcritic 설정 (.perlcriticrc)

```ini
severity = 3
theme = core + pbp + security

[InputOutput::RequireCheckedSyscalls]
functions = :builtins
exclude_functions = say print

[Subroutines::ProhibitExplicitReturnUndef]
severity = 4

[ValuesAndExpressions::ProhibitMagicNumbers]
allowed_values = 0 1 2 -1
```

### 종속성 관리 (cpanfile + carton)

```bash
cpanm App::cpanminus Carton   # 도구 설치
carton install                 # cpanfile에서 종속성 설치
carton exec -- perl bin/myapp  # 로컬 종속성과 함께 실행
```

```perl
# cpanfile
requires 'Moo', '>= 2.005';
requires 'Path::Tiny';
requires 'JSON::MaybeXS';
requires 'Try::Tiny';

on test => sub {
    requires 'Test2::V0';
    requires 'Test::MockModule';
};
```

## 빠른 참조: 모던 Perl 이디엄

| 레거시 패턴 | 모던 대체제 |
|---|---|
| `use strict; use warnings;` | `use v5.36;` |
| `my ($x, $y) = @_;` | `sub foo($x, $y) { ... }` |
| `@{ $ref }` | `$ref->@*` |
| `%{ $ref }` | `$ref->%*` |
| `open FH, "< $file"` | `open my $fh, '<:encoding(UTF-8)', $file` |
| `blessed hashref` | 타입을 가진 `Moo` 클래스 |
| `$1, $2, $3` | `$+{name}` (Named captures) |
| `eval { }; if ($@)` | `Try::Tiny` 또는 네이티브 `try/catch` (5.40+) |
| `BEGIN { require Exporter; }` | `use Exporter 'import';` |
| 수동 파일 작업 | `Path::Tiny` |
| `blessed($o) && $o->isa('X')` | `$o isa 'X'` (5.32+) |
| `builtin::true / false` | `use builtin 'true', 'false';` (5.36+, 실험적) |

## 안티 패턴 (Anti-Patterns)

```perl
# 1. 2-인자 open (보안 위험)
open FH, $filename;                     # 절대 사용 금지

# 2. 간접 객체 구문 (파싱 모호성)
my $obj = new Foo(bar => 1);            # Bad
my $obj = Foo->new(bar => 1);           # Good

# 3. $_에 대한 과도한 의존
map { process($_) } grep { validate($_) } @items;  # 따라가기 어려움
my @filtered = grep { validate($_) } @items;           # Better: 단계를 나눔
my @processed = map { process($_) } @filtered;

# 4. strict refs 비활성화
no strict 'refs';                        # 거의 항상 잘못된 접근
${"My::Package::$var"} = $value;         # 대신 해시를 사용하세요

# 5. 설정을 위한 전역 변수
our $TIMEOUT = 30;                       # Bad: 변경 가능한 전역 변수
use constant TIMEOUT => 30;              # Better: 상수
# Best: 기본값이 있는 Moo 속성

# 6. 모듈 로딩을 위한 문자열 eval
eval "require $module";                  # Bad: 코드 인젝션 위험
eval "use $module";                      # Bad
use Module::Runtime 'require_module';    # Good: 안전한 모듈 로딩
require_module($module);
```

**기억하세요**: 모던 Perl은 깔끔하고 가독성이 좋으며 안전합니다. `use v5.36`이 상용구를 처리하게 하고, 객체에는 Moo를 사용하며, 직접 만든 해결책보다는 CPAN의 검증된 모듈을 선호하십시오.

---
name: perl-security
description: taint mode, input validation, safe process execution, DBI parameterized queries, web security (XSS/SQLi/CSRF) 및 perlcritic security policies를 포함하는 포괄적인 Perl 보안 가이드.
origin: ECC
---

# Perl 보안 패턴

input validation, injection 방지 및 보안 코딩 관행을 다루는 Perl 애플리케이션을 위한 포괄적인 보안 가이드라인.

## 활성화 시점

- Perl 애플리케이션에서 사용자 입력을 처리할 때
- Perl web 애플리케이션(CGI, Mojolicious, Dancer2, Catalyst)을 빌드할 때
- 보안 취약점에 대해 Perl 코드를 review할 때
- 사용자가 제공한 경로로 파일 작업을 수행할 때
- Perl에서 시스템 명령을 실행할 때
- DBI 데이터베이스 쿼리를 작성할 때

## 작동 방식

taint-aware input 경계에서 시작하여 외부로 확장하세요. 입력을 validate 및 untaint하고, 파일 시스템 및 프로세스 실행을 제한하며, 모든 곳에서 parameterized DBI 쿼리를 사용하세요. 아래 예시들은 사용자 입력, 쉘 또는 네트워크와 상호작용하는 Perl 코드를 배포하기 전에 이 skill이 적용할 것으로 예상하는 안전한 기본값들을 보여줍니다.

## Taint Mode

Perl의 taint mode (`-T`)는 외부 소스의 데이터를 추적하고 명시적인 validation 없이 안전하지 않은 작업에 사용되는 것을 방지합니다.

### Taint Mode 활성화

```perl
#!/usr/bin/perl -T
use v5.36;

# Tainted: anything from outside the program
my $input    = $ARGV[0];        # Tainted
my $env_path = $ENV{PATH};      # Tainted
my $form     = <STDIN>;         # Tainted
my $query    = $ENV{QUERY_STRING}; # Tainted

# Sanitize PATH early (required in taint mode)
$ENV{PATH} = '/usr/local/bin:/usr/bin:/bin';
delete @ENV{qw(IFS CDPATH ENV BASH_ENV)};
```

### Untainting 패턴

```perl
use v5.36;

# Good: Validate and untaint with a specific regex
sub untaint_username($input) {
    if ($input =~ /^([a-zA-Z0-9_]{3,30})$/) {
        return $1;  # $1 is untainted
    }
    die "Invalid username: must be 3-30 alphanumeric characters\n";
}

# Good: Validate and untaint a file path
sub untaint_filename($input) {
    if ($input =~ m{^([a-zA-Z0-9._-]+)$}) {
        return $1;
    }
    die "Invalid filename: contains unsafe characters\n";
}

# Bad: Overly permissive untainting (defeats the purpose)
sub bad_untaint($input) {
    $input =~ /^(.*)$/s;
    return $1;  # Accepts ANYTHING — pointless
}
```

## Input Validation

### Blocklist보다 Allowlist 우선

```perl
use v5.36;

# Good: Allowlist — define exactly what's permitted
sub validate_sort_field($field) {
    my %allowed = map { $_ => 1 } qw(name email created_at updated_at);
    die "Invalid sort field: $field\n" unless $allowed{$field};
    return $field;
}

# Good: Validate with specific patterns
sub validate_email($email) {
    if ($email =~ /^([a-zA-Z0-9._%+-]+\@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,})$/) {
        return $1;
    }
    die "Invalid email address\n";
}

sub validate_integer($input) {
    if ($input =~ /^(-?\d{1,10})$/) {
        return $1 + 0;  # Coerce to number
    }
    die "Invalid integer\n";
}

# Bad: Blocklist — always incomplete
sub bad_validate($input) {
    die "Invalid" if $input =~ /[<>"';&|]/;  # Misses encoded attacks
    return $input;
}
```

### 길이 제한

```perl
use v5.36;

sub validate_comment($text) {
    die "Comment is required\n"        unless length($text) > 0;
    die "Comment exceeds 10000 chars\n" if length($text) > 10_000;
    return $text;
}
```

## 안전한 정규식

### ReDoS 방지

겹치는 패턴에 중첩된 수량자가 있을 때 치명적인 backtracking이 발생합니다.

```perl
use v5.36;

# Bad: Vulnerable to ReDoS (exponential backtracking)
my $bad_re = qr/^(a+)+$/;           # Nested quantifiers
my $bad_re2 = qr/^([a-zA-Z]+)*$/;   # Nested quantifiers on class
my $bad_re3 = qr/^(.*?,){10,}$/;    # Repeated greedy/lazy combo

# Good: Rewrite without nesting
my $good_re = qr/^a+$/;             # Single quantifier
my $good_re2 = qr/^[a-zA-Z]+$/;     # Single quantifier on class

# Good: Use possessive quantifiers or atomic groups to prevent backtracking
my $safe_re = qr/^[a-zA-Z]++$/;             # Possessive (5.10+)
my $safe_re2 = qr/^(?>a+)$/;                # Atomic group

# Good: Enforce timeout on untrusted patterns
use POSIX qw(alarm);
sub safe_match($string, $pattern, $timeout = 2) {
    my $matched;
    eval {
        local $SIG{ALRM} = sub { die "Regex timeout\n" };
        alarm($timeout);
        $matched = $string =~ $pattern;
        alarm(0);
    };
    alarm(0);
    die $@ if $@;
    return $matched;
}
```

## 안전한 파일 작업

### Three-Argument Open

```perl
use v5.36;

# Good: Three-arg open, lexical filehandle, check return
sub read_file($path) {
    open my $fh, '<:encoding(UTF-8)', $path
        or die "Cannot open '$path': $!\n";
    local $/;
    my $content = <$fh>;
    close $fh;
    return $content;
}

# Bad: Two-arg open with user data (command injection)
sub bad_read($path) {
    open my $fh, $path;        # If $path = "|rm -rf /", runs command!
    open my $fh, "< $path";   # Shell metacharacter injection
}
```

### TOCTOU 방지 및 Path Traversal

```perl
use v5.36;
use Fcntl qw(:DEFAULT :flock);
use File::Spec;
use Cwd qw(realpath);

# Atomic file creation
sub create_file_safe($path) {
    sysopen(my $fh, $path, O_WRONLY | O_CREAT | O_EXCL, 0600)
        or die "Cannot create '$path': $!\n";
    return $fh;
}

# Validate path stays within allowed directory
sub safe_path($base_dir, $user_path) {
    my $real = realpath(File::Spec->catfile($base_dir, $user_path))
        // die "Path does not exist\n";
    my $base_real = realpath($base_dir)
        // die "Base dir does not exist\n";
    die "Path traversal blocked\n" unless $real =~ /^\Q$base_real\E(?:\/|\z)/;
    return $real;
}
```

임시 파일에는 `File::Temp` (`tempfile(UNLINK => 1)`)를 사용하고, race condition 방지를 위해 `flock(LOCK_EX)`을 사용하세요.

## Safe Process Execution

### List-Form system 및 exec

```perl
use v5.36;

# Good: List form — no shell interpolation
sub run_command(@cmd) {
    system(@cmd) == 0
        or die "Command failed: @cmd\n";
}

run_command('grep', '-r', $user_pattern, '/var/log/app/');

# Good: Capture output safely with IPC::Run3
use IPC::Run3;
sub capture_output(@cmd) {
    my ($stdout, $stderr);
    run3(\@cmd, \undef, \$stdout, \$stderr);
    if ($?) {
        die "Command failed (exit $?): $stderr\n";
    }
    return $stdout;
}

# Bad: String form — shell injection!
sub bad_search($pattern) {
    system("grep -r '$pattern' /var/log/app/");  # If $pattern = "'; rm -rf / #"
}

# Bad: Backticks with interpolation
my $output = `ls $user_dir`;   # Shell injection risk
```

또한 외부 명령의 stdout/stderr를 안전하게 캡처하려면 `Capture::Tiny`를 사용하세요.

## SQL Injection 방지

### DBI Placeholders

```perl
use v5.36;
use DBI;

my $dbh = DBI->connect($dsn, $user, $pass, {
    RaiseError => 1,
    PrintError => 0,
    AutoCommit => 1,
});

# Good: Parameterized queries — always use placeholders
sub find_user($dbh, $email) {
    my $sth = $dbh->prepare('SELECT * FROM users WHERE email = ?');
    $sth->execute($email);
    return $sth->fetchrow_hashref;
}

sub search_users($dbh, $name, $status) {
    my $sth = $dbh->prepare(
        'SELECT * FROM users WHERE name LIKE ? AND status = ? ORDER BY name'
    );
    $sth->execute("%$name%", $status);
    return $sth->fetchall_arrayref({});
}

# Bad: String interpolation in SQL (SQLi vulnerability!)
sub bad_find($dbh, $email) {
    my $sth = $dbh->prepare("SELECT * FROM users WHERE email = '$email'");
    # If $email = "' OR 1=1 --", returns all users
    $sth->execute;
    return $sth->fetchrow_hashref;
}
```

### 동적 컬럼 Allowlist

```perl
use v5.36;

# Good: Validate column names against an allowlist
sub order_by($dbh, $column, $direction) {
    my %allowed_cols = map { $_ => 1 } qw(name email created_at);
    my %allowed_dirs = map { $_ => 1 } qw(ASC DESC);

    die "Invalid column: $column\n"    unless $allowed_cols{$column};
    die "Invalid direction: $direction\n" unless $allowed_dirs{uc $direction};

    my $sth = $dbh->prepare("SELECT * FROM users ORDER BY $column $direction");
    $sth->execute;
    return $sth->fetchall_arrayref({});
}

# Bad: Directly interpolating user-chosen column
sub bad_order($dbh, $column) {
    $dbh->prepare("SELECT * FROM users ORDER BY $column");  # SQLi!
}
```

### DBIx::Class (ORM 안전성)

```perl
use v5.36;

# DBIx::Class은 안전한 parameterized queries를 생성합니다.
my @users = $schema->resultset('User')->search({
    status => 'active',
    email  => { -like => '%@example.com' },
}, {
    order_by => { -asc => 'name' },
    rows     => 50,
});
```

## Web Security

### XSS 방지

```perl
use v5.36;
use HTML::Entities qw(encode_entities);
use URI::Escape qw(uri_escape_utf8);

# Good: Encode output for HTML context
sub safe_html($user_input) {
    return encode_entities($user_input);
}

# Good: Encode for URL context
sub safe_url_param($value) {
    return uri_escape_utf8($value);
}

# Good: Encode for JSON context
use JSON::MaybeXS qw(encode_json);
sub safe_json($data) {
    return encode_json($data);  # Handles escaping
}

# Template auto-escaping (Mojolicious)
# <%= $user_input %>   — auto-escaped (safe)
# <%== $raw_html %>    — raw output (dangerous, use only for trusted content)

# Template auto-escaping (Template Toolkit)
# [% user_input | html %]  — explicit HTML encoding

# Bad: Raw output in HTML
sub bad_html($input) {
    print "<div>$input</div>";  # XSS if $input contains <script>
}
```

### CSRF 보호

```perl
use v5.36;
use Crypt::URandom qw(urandom);
use MIME::Base64 qw(encode_base64url);

sub generate_csrf_token() {
    return encode_base64url(urandom(32));
}
```

토큰을 검증할 때 constant-time 비교를 사용하세요. 대부분의 웹 프레임워크(Mojolicious, Dancer2, Catalyst)는 내장된 CSRF 보호 기능을 제공하므로 직접 만든 솔루션보다 이를 선호하세요.

### 세션 및 헤더 보안

```perl
use v5.36;

# Mojolicious session + headers
$app->secrets(['long-random-secret-rotated-regularly']);
$app->sessions->secure(1);          # HTTPS only
$app->sessions->samesite('Lax');

$app->hook(after_dispatch => sub ($c) {
    $c->res->headers->header('X-Content-Type-Options' => 'nosniff');
    $c->res->headers->header('X-Frame-Options'        => 'DENY');
    $c->res->headers->header('Content-Security-Policy' => "default-src 'self'");
    $c->res->headers->header('Strict-Transport-Security' => 'max-age=31536000; includeSubDomains');
});
```

## Output Encoding

항상 컨텍스트에 맞게 출력을 encode하세요: HTML의 경우 `HTML::Entities::encode_entities()`, URL의 경우 `URI::Escape::uri_escape_utf8()`, JSON의 경우 `JSON::MaybeXS::encode_json()`.

## CPAN 모듈 보안

- **버전 고정**: cpanfile에서 버전을 고정하세요: `requires 'DBI', '== 1.643';`
- **유지 관리되는 모듈 선호**: 최근 릴리스에 대해 MetaCPAN 확인
- **종속성 최소화**: 각 종속성은 공격 표면(attack surface)입니다.

## 보안 툴링

### perlcritic Security Policies

```ini
# .perlcriticrc — security-focused configuration
severity = 3
theme = security + core

# Require three-arg open
[InputOutput::RequireThreeArgOpen]
severity = 5

# Require checked system calls
[InputOutput::RequireCheckedSyscalls]
functions = :builtins
severity = 4

# Prohibit string eval
[BuiltinFunctions::ProhibitStringyEval]
severity = 5

# Prohibit backtick operators
[InputOutput::ProhibitBacktickOperators]
severity = 4

# Require taint checking in CGI
[Modules::RequireTaintChecking]
severity = 5

# Prohibit two-arg open
[InputOutput::ProhibitTwoArgOpen]
severity = 5

# Prohibit bare-word filehandles
[InputOutput::ProhibitBarewordFileHandles]
severity = 5
```

### perlcritic 실행

```bash
# 파일 확인
perlcritic --severity 3 --theme security lib/MyApp/Handler.pm

# 프로젝트 전체 확인
perlcritic --severity 3 --theme security lib/

# CI 통합
perlcritic --severity 4 --theme security --quiet lib/ || exit 1
```

## 빠른 보안 체크리스트

| 확인 항목 | 검증 내용 |
|---|---|
| Taint mode | CGI/웹 스크립트의 `-T` 플래그 |
| Input validation | Allowlist 패턴, 길이 제한 |
| 파일 작업 | Three-arg open, path traversal 확인 |
| 프로세스 실행 | List-form system, 쉘 보간 없음 |
| SQL 쿼리 | DBI placeholders, 보간 금지 |
| HTML 출력 | `encode_entities()`, 템플릿 자동 이스케이프 |
| CSRF 토큰 | 생성됨, 상태 변경 요청 시 검증됨 |
| 세션 설정 | Secure, HttpOnly, SameSite 쿠키 |
| HTTP 헤더 | CSP, X-Frame-Options, HSTS |
| 종속성 | 고정된 버전, 감사된 모듈 |
| 정규식 안전성 | 중첩된 수량자 없음, anchored 패턴 |
| 에러 메시지 | 사용자에게 스택 추적이나 경로 유출 금지 |

## 안티 패턴 (Anti-Patterns)

```perl
# 1. 사용자 데이터가 포함된 Two-arg open (command injection)
open my $fh, $user_input;               # 치명적인 취약점

# 2. String-form system (shell injection)
system("convert $user_file output.png"); # 치명적인 취약점

# 3. SQL 문자열 보간
$dbh->do("DELETE FROM users WHERE id = $id");  # SQLi

# 4. 사용자 입력이 포함된 eval (code injection)
eval $user_code;                         # 원격 코드 실행

# 5. sanitizing 없이 $ENV 신뢰
my $path = $ENV{UPLOAD_DIR};             # 조작될 수 있음
system("ls $path");                      # 이중 취약점

# 6. validation 없이 taint 비활성화
($input) = $input =~ /(.*)/s;           # 게으른 untaint — 목적에 어긋남

# 7. HTML 내의 Raw 사용자 데이터
print "<div>Welcome, $username!</div>";  # XSS

# 8. 검증되지 않은 리다이렉트
print $cgi->redirect($user_url);         # Open redirect
```

**기억하세요**: Perl의 유연성은 강력하지만 규율이 필요합니다. 웹 지향 코드에는 taint mode를 사용하고, 모든 입력을 allowlist로 validate하며, 모든 쿼리에 DBI placeholders를 사용하고, 모든 출력을 컨텍스트에 맞게 encode하세요. 심층 방어(Defense in depth) — 단일 계층에만 의존하지 마세요.

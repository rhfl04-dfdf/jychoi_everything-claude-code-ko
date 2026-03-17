---
name: perl-testing
description: Test2::V0, Test::More, prove runner, mocking, Devel::Cover를 사용한 coverage, TDD 방법론을 포함한 Perl testing pattern들.
origin: ECC
---

# Perl Testing Patterns

Test2::V0, Test::More, prove, 그리고 TDD 방법론을 사용한 Perl 애플리케이션을 위한 종합적인 testing 전략.

## 활성화 시점

- 새로운 Perl 코드를 작성할 때 (TDD 준수: red, green, refactor)
- Perl 모듈이나 애플리케이션을 위한 test suite를 설계할 때
- Perl test coverage를 검토할 때
- Perl testing 인프라를 설정할 때
- test를 Test::More에서 Test2::V0로 마이그레이션할 때
- 실패하는 Perl test를 디버깅할 때

## TDD Workflow

항상 RED-GREEN-REFACTOR 사이클을 따릅니다.

```perl
# Step 1: RED — Write a failing test
# t/unit/calculator.t
use v5.36;
use Test2::V0;

use lib 'lib';
use Calculator;

subtest 'addition' => sub {
    my $calc = Calculator->new;
    is($calc->add(2, 3), 5, 'adds two numbers');
    is($calc->add(-1, 1), 0, 'handles negatives');
};

done_testing;

# Step 2: GREEN — Write minimal implementation
# lib/Calculator.pm
package Calculator;
use v5.36;
use Moo;

sub add($self, $a, $b) {
    return $a + $b;
}

1;

# Step 3: REFACTOR — Improve while tests stay green
# Run: prove -lv t/unit/calculator.t
```

## Test::More Fundamentals

표준 Perl testing 모듈 — 널리 사용되며 core에 포함되어 있습니다.

### Basic Assertions

```perl
use v5.36;
use Test::More;

# Plan upfront or use done_testing
# plan tests => 5;  # Fixed plan (optional)

# Equality
is($result, 42, 'returns correct value');
isnt($result, 0, 'not zero');

# Boolean
ok($user->is_active, 'user is active');
ok(!$user->is_banned, 'user is not banned');

# Deep comparison
is_deeply(
    $got,
    { name => 'Alice', roles => ['admin'] },
    'returns expected structure'
);

# Pattern matching
like($error, qr/not found/i, 'error mentions not found');
unlike($output, qr/password/, 'output hides password');

# Type check
isa_ok($obj, 'MyApp::User');
can_ok($obj, 'save', 'delete');

done_testing;
```

### SKIP and TODO

```perl
use v5.36;
use Test::More;

# Skip tests conditionally
SKIP: {
    skip 'No database configured', 2 unless $ENV{TEST_DB};

    my $db = connect_db();
    ok($db->ping, 'database is reachable');
    is($db->version, '15', 'correct PostgreSQL version');
}

# Mark expected failures
TODO: {
    local $TODO = 'Caching not yet implemented';
    is($cache->get('key'), 'value', 'cache returns value');
}

done_testing;
```

## Test2::V0 Modern Framework

Test2::V0는 Test::More의 현대적인 대체제입니다 — 더 풍부한 assertion, 더 나은 진단 기능, 그리고 확장성을 제공합니다.

### 왜 Test2인가?

- hash/array builder를 사용한 우수한 deep comparison
- 실패 시 더 나은 진단 출력
- 더 깔끔한 scoping을 가진 subtest
- Test2::Tools::* 플러그인을 통한 확장성
- Test::More test와 하위 호환성 유지

### Builder를 사용한 Deep Comparison

```perl
use v5.36;
use Test2::V0;

# Hash builder — check partial structure
is(
    $user->to_hash,
    hash {
        field name  => 'Alice';
        field email => match(qr/\@example\.com$/);
        field age   => validator(sub { $_ >= 18 });
        # Ignore other fields
        etc();
    },
    'user has expected fields'
);

# Array builder
is(
    $result,
    array {
        item 'first';
        item match(qr/^second/);
        item DNE();  # Does Not Exist — verify no extra items
    },
    'result matches expected list'
);

# Bag — order-independent comparison
is(
    $tags,
    bag {
        item 'perl';
        item 'testing';
        item 'tdd';
    },
    'has all required tags regardless of order'
);
```

### Subtests

```perl
use v5.36;
use Test2::V0;

subtest 'User creation' => sub {
    my $user = User->new(name => 'Alice', email => 'alice @example.com');
    ok($user, 'user object created');
    is($user->name, 'Alice', 'name is set');
    is($user->email, 'alice @example.com', 'email is set');
};

subtest 'User validation' => sub {
    my $warnings = warns {
        User->new(name => '', email => 'bad');
    };
    ok($warnings, 'warns on invalid data');
};

done_testing;
```

### Test2를 사용한 Exception Testing

```perl
use v5.36;
use Test2::V0;

# Test that code dies
like(
    dies { divide(10, 0) },
    qr/Division by zero/,
    'dies on division by zero'
);

# Test that code lives
ok(lives { divide(10, 2) }, 'division succeeds') or note($@);

# Combined pattern
subtest 'error handling' => sub {
    ok(lives { parse_config('valid.json') }, 'valid config parses');
    like(
        dies { parse_config('missing.json') },
        qr/Cannot open/,
        'missing file dies with message'
    );
};

done_testing;
```

## Test Organization과 prove

### 디렉토리 구조

```text
t/
├── 00-load.t              # Verify modules compile
├── 01-basic.t             # Core functionality
├── unit/
│   ├── config.t           # Unit tests by module
│   ├── user.t
│   └── util.t
├── integration/
│   ├── database.t
│   └── api.t
├── lib/
│   └── TestHelper.pm      # Shared test utilities
└── fixtures/
    ├── config.json        # Test data files
    └── users.csv
```

### prove 커맨드

```bash
# Run all tests
prove -l t/

# Verbose output
prove -lv t/

# Run specific test
prove -lv t/unit/user.t

# Recursive search
prove -lr t/

# Parallel execution (8 jobs)
prove -lr -j8 t/

# Run only failing tests from last run
prove -l --state=failed t/

# Colored output with timer
prove -l --color --timer t/

# TAP output for CI
prove -l --formatter TAP::Formatter::JUnit t/ > results.xml
```

### .proverc 설정

```text
-l
--color
--timer
-r
-j4
--state=save
```

## Fixtures 및 Setup/Teardown

### Subtest 격리

```perl
use v5.36;
use Test2::V0;
use File::Temp qw(tempdir);
use Path::Tiny;

subtest 'file processing' => sub {
    # Setup
    my $dir = tempdir(CLEANUP => 1);
    my $file = path($dir, 'input.txt');
    $file->spew_utf8("line1\nline2\nline3\n");

    # Test
    my $result = process_file("$file");
    is($result->{line_count}, 3, 'counts lines');

    # Teardown happens automatically (CLEANUP => 1)
};
```

### 공유 Test Helper

재사용 가능한 helper를 `t/lib/TestHelper.pm`에 배치하고 `use lib 't/lib'`로 로드하십시오. `create_test_db()`, `create_temp_dir()`, `fixture_path()`와 같은 팩토리 함수를 `Exporter`를 통해 내보내십시오.

## Mocking

### Test::MockModule

```perl
use v5.36;
use Test2::V0;
use Test::MockModule;

subtest 'mock external API' => sub {
    my $mock = Test::MockModule->new('MyApp::API');

    # Good: Mock returns controlled data
    $mock->mock(fetch_user => sub ($self, $id) {
        return { id => $id, name => 'Mock User', email => 'mock @test.com' };
    });

    my $api = MyApp::API->new;
    my $user = $api->fetch_user(42);
    is($user->{name}, 'Mock User', 'returns mocked user');

    # Verify call count
    my $call_count = 0;
    $mock->mock(fetch_user => sub { $call_count++; return {} });
    $api->fetch_user(1);
    $api->fetch_user(2);
    is($call_count, 2, 'fetch_user called twice');

    # Mock is automatically restored when $mock goes out of scope
};

# Bad: Monkey-patching without restoration
# *MyApp::API::fetch_user = sub { ... };  # NEVER — leaks across tests
```

가벼운 mock 객체의 경우, `Test::MockObject`를 사용하여 `->mock()`으로 주입 가능한 test double을 생성하고 `->called_ok()`로 호출을 검증하십시오.

## Devel::Cover를 사용한 Coverage

### Coverage 실행

```bash
# Basic coverage report
cover -test

# Or step by step
perl -MDevel::Cover -Ilib t/unit/user.t
cover

# HTML report
cover -report html
open cover_db/coverage.html

# Specific thresholds
cover -test -report text | grep 'Total'

# CI-friendly: fail under threshold
cover -test && cover -report text -select '^lib/' \
  | perl -ne 'if (/Total.*?(\d+\.\d+)/) { exit 1 if $1 < 80 }'
```

### Integration Testing

데이터베이스 test에는 in-memory SQLite를 사용하고, API test에는 `HTTP::Tiny`를 mocking 하십시오.

```perl
use v5.36;
use Test2::V0;
use DBI;

subtest 'database integration' => sub {
    my $dbh = DBI->connect('dbi:SQLite:dbname=:memory:', '', '', {
        RaiseError => 1,
    });
    $dbh->do('CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT)');

    $dbh->prepare('INSERT INTO users (name) VALUES (?)')->execute('Alice');
    my $row = $dbh->selectrow_hashref('SELECT * FROM users WHERE name = ?', undef, 'Alice');
    is($row->{name}, 'Alice', 'inserted and retrieved user');
};

done_testing;
```

## Best Practices

### 권장 사항 (DO)

- **TDD 준수**: 구현 전에 test 작성 (red-green-refactor)
- **Test2::V0 사용**: 현대적인 assertion, 더 나은 진단
- **subtest 사용**: 관련 assertion 그룹화, 상태 격리
- **외부 종속성 mocking**: 네트워크, 데이터베이스, 파일 시스템
- **`prove -l` 사용**: 항상 ` @INC`에 lib/ 포함
- **test 이름을 명확하게 지정**: `'user login with invalid password fails'`
- **edge case test**: 빈 문자열, undef, 0, 경계값
- **80% 이상의 coverage 목표**: 비즈니스 로직 경로에 집중
- **test 속도 유지**: I/O mocking, in-memory 데이터베이스 사용

### 피해야 할 사항 (DON'T)

- **구현을 test하지 말 것**: 내부 구조가 아닌 동작과 출력을 test
- **subtest 간에 상태를 공유하지 말 것**: 각 subtest는 독립적이어야 함
- **`done_testing`을 생략하지 말 것**: 계획된 모든 test가 실행되었는지 확인
- **과도하게 mocking하지 말 것**: 경계 부분만 mocking하고, test 대상 코드는 제외
- **새 프로젝트에 `Test::More`를 사용하지 말 것**: Test2::V0 권장
- **test 실패를 무시하지 말 것**: merge 전에 모든 test를 통과해야 함
- **CPAN 모듈을 test하지 말 것**: 라이브러리가 올바르게 작동한다고 신뢰
- **취약한(brittle) test를 작성하지 말 것**: 과도하게 구체적인 문자열 매칭 지양

## Quick Reference

| Task | Command / Pattern |
|---|---|
| 모든 test 실행 | `prove -lr t/` |
| 한 개의 test를 상세히 실행 | `prove -lv t/unit/user.t` |
| 병렬 test 실행 | `prove -lr -j8 t/` |
| Coverage 보고서 | `cover -test && cover -report html` |
| 동등성 test | `is($got, $expected, 'label')` |
| Deep comparison | `is($got, hash { field k => 'v'; etc() }, 'label')` |
| 예외 test | `like(dies { ... }, qr/msg/, 'label')` |
| 예외 없음 test | `ok(lives { ... }, 'label')` |
| 메서드 mocking | `Test::MockModule->new('Pkg')->mock(m => sub { ... })` |
| test 건너뛰기 | `SKIP: { skip 'reason', $count unless $cond; ... }` |
| TODO test | `TODO: { local $TODO = 'reason'; ... }` |

## 자주 발생하는 실수 (Common Pitfalls)

### `done_testing` 누락

```perl
# Bad: Test file runs but doesn't verify all tests executed
use Test2::V0;
is(1, 1, 'works');
# Missing done_testing — silent bugs if test code is skipped

# Good: Always end with done_testing
use Test2::V0;
is(1, 1, 'works');
done_testing;
```

### `-l` 플래그 누락

```bash
# Bad: Modules in lib/ not found
prove t/unit/user.t
# Can't locate MyApp/User.pm in @INC

# Good: Include lib/ in @INC
prove -l t/unit/user.t
```

### 과도한 Mocking

test 대상 코드가 아닌 *종속성*을 mocking 하십시오. 만약 test가 단순히 mock이 지시한 대로 반환하는지만 확인한다면, 그것은 아무것도 test하지 않는 것입니다.

### Test 오염 (Test Pollution)

test 간의 상태 누출을 방지하기 위해 subtest 내부에서는 `my` 변수를 사용하십시오 — 절대로 `our`를 사용하지 마십시오.

**기억하세요**: Test는 여러분의 안전망입니다. Test를 빠르고, 집중적이며, 독립적으로 유지하십시오. 새 프로젝트에는 Test2::V0를, 실행에는 prove를, 책임성 확인에는 Devel::Cover를 사용하십시오.

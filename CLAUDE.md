# Perlコーディング規約

## 基本パターン

関数の引数・返り値は以下のツールを組み合わせて書きます：

1. **Syntax::Keyword::Assert + Type::Tiny** - 開発時のみの型チェック
2. **Result::Simple** - エラーハンドリング（ok/err）+ 返り値型チェック（result_for）

### 基本例

```perl
use Types::Standard qw(Num);
use Result::Simple qw(ok err result_for);
use Syntax::Keyword::Assert;

sub divide {
    my ($a, $b) = @_;

    # 引数の型チェック（開発時のみ）
    assert Num->check($a);
    assert Num->check($b);
    assert $b != 0;

    # バリデーション（常に実行）
    return err('Undefined value') unless defined $a && defined $b;
    return err('Division by zero') if $b == 0;

    # 計算
    my $result = $a / $b;

    # 事後条件（開発時のみ）
    assert defined $result;

    return ok($result);
}

# 返り値の型チェック（開発時のみ）
result_for divide => 'Num', 'Str';

# 使用例
my ($result, $error) = divide(10, 2);
if ($error) {
    warn "Error: $error";
} else {
    say "Result: $result";
}
```

## 複雑な型チェック

```perl
use Types::Standard qw(Str Int Object);
use Result::Simple qw(ok err result_for);
use Syntax::Keyword::Assert;

sub create_user {
    my ($name, $email, $age) = @_;

    # 引数の型チェック（開発時のみ）
    assert Str->check($name);
    assert Str->check($email);
    assert Int->check($age);
    assert $age > 0;

    # バリデーション（常に実行）
    return err('Name is required') unless defined $name;
    return err('Invalid email') unless $email =~ /^[^@]+@[^@]+$/;
    return err('Age must be positive') if $age <= 0;

    # 処理
    my $user = eval { $db->create_user($name, $email, $age) };
    return err("DB error: $@") if $@;

    # 事後条件（開発時のみ）
    assert defined $user;
    assert Object->check($user);

    return ok($user);
}

result_for create_user => 'Object', 'Str';
```

## 名前付き引数パターン

```perl
use Types::Standard qw(Dict Str Int Optional);
use Result::Simple qw(ok err result_for);
use Syntax::Keyword::Assert;

sub create_user {
    my ($args) = @_;

    # 引数の型チェック（開発時のみ）
    assert Dict[
        name => Str,
        email => Str,
        age => Int,
        phone => Optional[Str],
    ]->check($args);

    # バリデーション（常に実行）
    return err('Name is required') unless $args->{name};
    return err('Invalid email') unless $args->{email} =~ /^[^@]+@[^@]+$/;
    return err('Age must be positive') if $args->{age} <= 0;

    # 処理
    my $user = eval { $db->create_user($args) };
    return err("DB error: $@") if $@;

    # 事後条件（開発時のみ）
    assert defined $user;

    return ok($user);
}

result_for create_user => 'Object', 'Str';

# 使用例
my ($user, $error) = create_user({
    name => 'John Doe',
    email => 'john@example.com',
    age => 30,
});
```

## カスタム型の定義（kura）

```perl
package MyTypes {
    use parent 'Exporter::Tiny';
    use Types::Common -types;

    use kura EmailAddress => Str & sub { $_ =~ /^[^@]+@[^@]+$/ };
    use kura UserName => StrLength[3, 50];
}

# 使用例
use MyTypes qw(EmailAddress UserName);
use Types::Common::Numeric qw(PositiveInt);
use Result::Simple qw(ok err);
use Syntax::Keyword::Assert;

sub register_user {
    my ($username, $email, $age) = @_;

    # カスタム型でチェック（開発時のみ）
    assert UserName->check($username);
    assert EmailAddress->check($email);
    assert PositiveInt->check($age);

    # バリデーション（常に実行）
    return err('Invalid username') unless length($username) >= 3;
    return err('Invalid email') unless $email =~ /^[^@]+@[^@]+$/;
    return err('Age must be positive') if $age <= 0;

    # 処理...
    return ok($user);
}
```

## エラーチェーンパターン

```perl
use Types::Standard qw(Str HashRef);
use Result::Simple qw(ok err);
use Syntax::Keyword::Assert;

sub process_config {
    my ($filename) = @_;

    # 引数チェック（開発時のみ）
    assert Str->check($filename);

    # ファイル読み込み
    my ($content, $read_error) = read_file($filename);
    return (undef, $read_error) if $read_error;

    # JSON解析
    my $data = eval { decode_json($content) };
    return err("JSON error: $@") if $@;

    # 型チェック（開発時のみ）
    assert HashRef->check($data);

    # バリデーション
    my ($validated, $validation_error) = validate_config($data);
    return (undef, $validation_error) if $validation_error;

    return ok($validated);
}

result_for process_config => 'HashRef', 'Str';
```

## 環境設定

```bash
# 開発環境
export PERL_ASSERT_ENABLED=1
export RESULT_SIMPLE_CHECK_ENABLED=1

# 本番環境
export PERL_ASSERT_ENABLED=0
export RESULT_SIMPLE_CHECK_ENABLED=0
```

## 役割の違い

| 目的 | ツール | 実行タイミング |
|------|--------|----------------|
| 引数の型チェック | assert + Type->check() | 開発時のみ |
| エラーハンドリング | Result::Simple (err) | 常に実行 |
| 返り値の検証 | assert | 開発時のみ |
| 返り値の型チェック | result_for | 開発時のみ |

## ポイント

- **assert Type->check()**: 開発時の型チェック（本番では削除される）
- **return err**: 実際のエラーハンドリング（常に実行）
- **result_for**: 返り値の型が正しいか確認（開発時のみ）

このパターンで、開発時は厳格にチェックしつつ、本番ではゼロコストを実現できます。

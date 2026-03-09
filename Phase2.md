# Phase 2: アサーション（Should マッチャー）

## 1. アサーションとは

**アサーション（assertion）**とは「この値はこうであるべき」という主張をコードで表現したものです。
Pesterでは `Should` コマンドにマッチャーを組み合わせてアサーションを書きます。

```powershell
実際の値 | Should -マッチャー名 [期待値]
```

Phase 1では `-Be` / `-BeNullOrEmpty` / `-Throw` を学びました。
Phase 2ではカテゴリ別に全マッチャーを体系的に学びます。

---

## 2. 否定：-Not

すべてのマッチャーは `-Not` を前に置くことで否定になります。

```powershell
$result | Should -Not -Be 0          # 0 ではないこと
$result | Should -Not -BeNullOrEmpty # null でも空でもないこと
{ ... }  | Should -Not -Throw        # 例外が発生しないこと
```

このルールはこれ以降のすべてのマッチャーに適用できます。

---

## 3. 等値・比較系マッチャー

### 3-1. -Be：完全一致

```powershell
# 数値
42 | Should -Be 42

# 文字列（大文字小文字を区別する）
"Hello" | Should -Be "Hello"
"Hello" | Should -Not -Be "hello"
```

### 3-2. -BeExactly：文字列の厳密一致

`-Be` と同じく大文字小文字を区別します。文字列専用であることを明示したい場合に使います。

```powershell
"PowerShell" | Should -BeExactly "PowerShell"
```

### 3-3. -BeLike：ワイルドカード一致

`*`（任意の文字列）や `?`（任意の1文字）を使ったパターンマッチです。

```powershell
"Hello, World" | Should -BeLike "Hello*"
"Hello, World" | Should -BeLike "*World"
"Hello, World" | Should -BeLike "*o, W*"

# 大文字小文字を区別しない
"hello, world" | Should -BeLike "Hello*"   # 成功（BeLike は区別しない）
```

> `-BeLike` は大文字小文字を**区別しません**。区別したい場合は `-BeLikeExactly` を使います。

### 3-4. -Match：正規表現マッチ

正規表現（regex）でパターンマッチします。

```powershell
"error: file not found" | Should -Match "error:"
"user@example.com"      | Should -Match "^\w+@\w+\.\w+$"
"abc123"                | Should -Match "\d+"   # 数字を含む

# 大文字小文字を区別しない（-Match はデフォルトで区別しない）
"Hello" | Should -Match "hello"   # 成功
```

> `-Match` は大文字小文字を**区別しません**。区別したい場合は `-MatchExactly` を使います。

### 3-5. 数値比較

```powershell
10 | Should -BeGreaterThan 5       # 10 > 5
10 | Should -BeGreaterOrEqual 10   # 10 >= 10
5  | Should -BeLessThan 10         # 5 < 10
5  | Should -BeLessOrEqual 5       # 5 <= 5

# 範囲チェックの組み合わせ
$value = 7
$value | Should -BeGreaterOrEqual 1
$value | Should -BeLessOrEqual 10
```

---

## 4. 型・存在確認系マッチャー

### 4-1. -BeOfType：型の確認

```powershell
42          | Should -BeOfType [int]
"hello"     | Should -BeOfType [string]
3.14        | Should -BeOfType [double]
@(1, 2, 3)  | Should -BeOfType [array]

# カスタムクラスや .NET 型
Get-Date    | Should -BeOfType [DateTime]
```

### 4-2. -BeNullOrEmpty：null または空

```powershell
$null | Should -BeNullOrEmpty
""    | Should -BeNullOrEmpty
@()   | Should -BeNullOrEmpty   # 空配列も対象

"hello" | Should -Not -BeNullOrEmpty
```

### 4-3. -BeTrue / -BeFalse：真偽値

```powershell
$true  | Should -BeTrue
$false | Should -BeFalse

(1 -eq 1) | Should -BeTrue
(1 -eq 2) | Should -BeFalse
```

---

## 5. コレクション系マッチャー

配列やリストを扱うときに使います。

### 5-1. -Contain：要素が含まれるか

```powershell
$fruits = @("apple", "banana", "cherry")

$fruits | Should -Contain "banana"
$fruits | Should -Not -Contain "grape"
```

> `-Contain` は大文字小文字を**区別しません**。

### 5-2. -HaveCount：要素数の確認

```powershell
$items = @("a", "b", "c")
$items | Should -HaveCount 3

@() | Should -HaveCount 0
```

### 5-3. まとめてコレクションを確認するパターン

```powershell
$result = Get-SomeList

# 要素数を確認
$result | Should -HaveCount 3

# 特定の要素が含まれることを確認
$result | Should -Contain "期待する値"

# 空でないことを確認
$result | Should -Not -BeNullOrEmpty
```

---

## 6. 文字列系マッチャー

### 6-1. -BeIn：値が配列の中にあるか

`-Contain` の逆方向です。「この値がコレクションの中にある」ことを確認します。

```powershell
$allowedRoles = @("admin", "user", "guest")

"admin" | Should -BeIn $allowedRoles
"hacker" | Should -Not -BeIn $allowedRoles
```

### 6-2. -HaveParameter（関数のパラメータ確認）

関数が特定のパラメータを持つことを確認します。

```powershell
function Get-User {
    param([string]$Name, [int]$Age)
}

Get-Command Get-User | Should -HaveParameter "Name"
Get-Command Get-User | Should -HaveParameter "Age" -Type [int]
```

---

## 7. ファイル・パス系マッチャー

### 7-1. -Exist：パスが存在するか

```powershell
"C:\Windows" | Should -Exist
"C:\存在しないフォルダ" | Should -Not -Exist

# ファイルの存在確認
"C:\Windows\System32\notepad.exe" | Should -Exist
```

### 7-2. -FileContentMatch：ファイルの中身を確認

```powershell
# ファイルに特定の文字列が含まれるか（正規表現）
"C:\Logs\app.log" | Should -FileContentMatch "ERROR"
"C:\Config\settings.ini" | Should -FileContentMatch "^\[Database\]"
```

---

## 8. 例外系マッチャー

### 8-1. -Throw：例外の発生を確認

Phase 1で学びましたが、詳細を補足します。

```powershell
# 例外が発生することだけを確認
{ throw "何らかのエラー" } | Should -Throw

# メッセージのワイルドカード一致
{ throw "ファイルが見つかりません" } | Should -Throw "*ファイル*"

# 特定の例外型を確認
{ [System.IO.FileNotFoundException]::new("test") |
  ForEach-Object { throw $_ }
} | Should -Throw -ExceptionType ([System.IO.FileNotFoundException])
```

### 8-2. エラーレコードの取得

例外の詳細を取り出してさらに検証することもできます。

```powershell
$err = { throw "詳細なエラー情報" } | Should -Throw -PassThru
$err.Exception.Message | Should -Be "詳細なエラー情報"
```

---

## 9. 動作するサンプル：マッチャーの総まとめ

### テスト対象

```powershell
# UserManager.ps1

function New-UserRecord {
    param(
        [string]$Name,
        [int]$Age,
        [string]$Role
    )

    $validRoles = @("admin", "user", "guest")

    if ([string]::IsNullOrEmpty($Name)) {
        throw "名前は必須です"
    }
    if ($Age -lt 0 -or $Age -gt 150) {
        throw "年齢が範囲外です: $Age"
    }
    if ($Role -notin $validRoles) {
        throw "無効なロール: $Role"
    }

    return [PSCustomObject]@{
        Name      = $Name
        Age       = $Age
        Role      = $Role
        CreatedAt = Get-Date
    }
}

function Get-AdminUsers {
    param([array]$Users)
    return $Users | Where-Object { $_.Role -eq "admin" }
}
```

### テストファイル

```powershell
# UserManager.Tests.ps1

BeforeAll {
    . $PSScriptRoot/UserManager.ps1
}

Describe "New-UserRecord" {

    Context "正常な入力のとき" {
        BeforeAll {
            $user = New-UserRecord -Name "田中太郎" -Age 30 -Role "admin"
        }

        It "Name プロパティが正しい" {
            $user.Name | Should -Be "田中太郎"
        }

        It "Age プロパティが正しい" {
            $user.Age | Should -Be 30
        }

        It "Role が有効なロールである" {
            $user.Role | Should -BeIn @("admin", "user", "guest")
        }

        It "CreatedAt が DateTime 型である" {
            $user.CreatedAt | Should -BeOfType [DateTime]
        }
    }

    Context "無効な入力のとき" {
        It "名前が空のとき例外を投げる" {
            { New-UserRecord -Name "" -Age 25 -Role "user" } |
                Should -Throw "*名前は必須です*"
        }

        It "年齢がマイナスのとき例外を投げる" {
            { New-UserRecord -Name "山田" -Age -1 -Role "user" } |
                Should -Throw "*年齢が範囲外です*"
        }

        It "無効なロールのとき例外を投げる" {
            { New-UserRecord -Name "鈴木" -Age 40 -Role "superuser" } |
                Should -Throw "*無効なロール*"
        }
    }
}

Describe "Get-AdminUsers" {

    BeforeAll {
        $users = @(
            [PSCustomObject]@{ Name = "Alice"; Role = "admin" }
            [PSCustomObject]@{ Name = "Bob";   Role = "user"  }
            [PSCustomObject]@{ Name = "Carol";  Role = "admin" }
            [PSCustomObject]@{ Name = "Dave";   Role = "guest" }
        )
    }

    It "admin ロールのユーザーだけを返す" {
        $admins = Get-AdminUsers -Users $users
        $admins | Should -HaveCount 2
    }

    It "返ってきたユーザーは全員 admin ロール" {
        $admins = Get-AdminUsers -Users $users
        $admins | ForEach-Object {
            $_.Role | Should -Be "admin"
        }
    }

    It "結果に Alice が含まれる" {
        $admins = Get-AdminUsers -Users $users
        $admins.Name | Should -Contain "Alice"
    }

    It "Bob は含まれない" {
        $admins = Get-AdminUsers -Users $users
        $admins.Name | Should -Not -Contain "Bob"
    }
}
```

---

## 10. マッチャー選択の指針

どのマッチャーを使えばよいか迷ったときの判断基準です。

```
確認したいこと                      使うマッチャー
─────────────────────────────────────────────────
値が完全に等しい                    -Be
文字列がパターンに一致する           -BeLike（ワイルドカード）
文字列が正規表現に一致する           -Match
値が配列の中にある                  -BeIn
配列が値を含む                      -Contain
配列の要素数                        -HaveCount
値が null または空                  -BeNullOrEmpty
値が true / false                   -BeTrue / -BeFalse
型が正しい                          -BeOfType
数値の大小                          -BeGreaterThan 等
ファイルが存在する                  -Exist
例外が発生する                      -Throw
上記すべての否定                    -Not -マッチャー
```

---

## 11. まとめ

| カテゴリ | マッチャー |
|---|---|
| 等値 | `-Be` `-BeExactly` `-BeLike` `-Match` |
| 数値比較 | `-BeGreaterThan` `-BeLessThan` `-BeGreaterOrEqual` `-BeLessOrEqual` |
| 型・存在 | `-BeOfType` `-BeNullOrEmpty` `-BeTrue` `-BeFalse` |
| コレクション | `-Contain` `-HaveCount` `-BeIn` |
| ファイル | `-Exist` `-FileContentMatch` |
| 例外 | `-Throw` |
| 否定 | 全マッチャーに `-Not` を付けられる |

---

## 練習問題

### 問題 1
次のコードで使うべき正しいマッチャーを選んでください。

```powershell
$result = "ERROR: Connection timeout at 14:32:05"

# (a) "ERROR" という文字列で始まることを確認したい
$result | Should -???

# (b) タイムスタンプ（HH:MM:SS形式）が含まれることを確認したい
$result | Should -???

# (c) "WARNING" が含まれないことを確認したい
$result | Should -???
```

### 問題 2
以下の関数に対してテストを書いてください。

```powershell
# MathUtils.ps1
function Get-Statistics {
    param([int[]]$Numbers)

    if ($Numbers.Count -eq 0) {
        throw "数値が空です"
    }

    return [PSCustomObject]@{
        Count   = $Numbers.Count
        Sum     = ($Numbers | Measure-Object -Sum).Sum
        Average = ($Numbers | Measure-Object -Average).Average
        Min     = ($Numbers | Measure-Object -Minimum).Minimum
        Max     = ($Numbers | Measure-Object -Maximum).Maximum
    }
}
```

テストで確認すること：
1. `@(1, 2, 3, 4, 5)` を渡したとき `Count` が `5` であること
2. `Sum` が `15` であること
3. `Average` が `3` であること
4. `Min` が `1` 以上であること
5. `Max` が `Count` 以下であること（違います、`Max` が `5` であること）
6. 空配列を渡すと例外が投げられること
7. 結果が `PSCustomObject` 型であること

### 問題 3
`-BeLike` と `-Match` の違いを説明し、それぞれが適しているケースを1つずつ挙げてください。

---

## 練習問題の解答

### 問題 1 の解答

```powershell
# (a) "ERROR" で始まることを確認 → ワイルドカード
$result | Should -BeLike "ERROR*"

# (b) HH:MM:SS 形式が含まれる → 正規表現
$result | Should -Match "\d{2}:\d{2}:\d{2}"

# (c) "WARNING" を含まない → 否定
$result | Should -Not -Match "WARNING"
# または
$result | Should -Not -BeLike "*WARNING*"
```

### 問題 2 の解答

```powershell
# MathUtils.Tests.ps1
BeforeAll {
    . $PSScriptRoot/MathUtils.ps1
}

Describe "Get-Statistics" {

    Context "有効な数値配列のとき" {
        BeforeAll {
            $stats = Get-Statistics -Numbers @(1, 2, 3, 4, 5)
        }

        It "Count が 5 である" {
            $stats.Count | Should -Be 5
        }

        It "Sum が 15 である" {
            $stats.Sum | Should -Be 15
        }

        It "Average が 3 である" {
            $stats.Average | Should -Be 3
        }

        It "Min が 1 以上である" {
            $stats.Min | Should -BeGreaterOrEqual 1
        }

        It "Max が 5 である" {
            $stats.Max | Should -Be 5
        }

        It "結果が PSCustomObject 型である" {
            $stats | Should -BeOfType [PSCustomObject]
        }
    }

    Context "空配列のとき" {
        It "例外を投げる" {
            { Get-Statistics -Numbers @() } | Should -Throw "*数値が空です*"
        }
    }
}
```

### 問題 3 の解答

| | `-BeLike` | `-Match` |
|---|---|---|
| 使うパターン | `*`（任意文字列）`?`（任意1文字）のワイルドカード | 正規表現 |
| 大文字小文字 | 区別しない | 区別しない（`-MatchExactly` で区別可） |

**`-BeLike` が適しているケース：** ファイル名のパターン確認
```powershell
$filename = "report_2026-03-06.csv"
$filename | Should -BeLike "report_*.csv"
```

**`-Match` が適しているケース：** 日付フォーマットの検証
```powershell
$dateStr = "2026-03-06"
$dateStr | Should -Match "^\d{4}-\d{2}-\d{2}$"
```

ワイルドカードで表現できないような複雑なパターン（桁数指定、文字クラス等）には `-Match` を使います。

---

**次のステップ:** [Phase 3](Phase3.md)では `BeforeAll` / `AfterAll` / `BeforeEach` / `AfterEach` を使ったセットアップとティアダウンを学びます。

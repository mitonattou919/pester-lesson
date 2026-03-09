# Phase 1: 基礎構文

## 1. テストファイルの骨格

Pesterのテストファイルは以下の3つのブロックで構成されます。

```powershell
BeforeAll {
    # テスト対象のスクリプトを読み込む
    . $PSScriptRoot/MyScript.ps1
}

Describe "テストグループの説明" {
    It "個別テストの説明" {
        # 実際の値
        $result = 何かの処理

        # 期待値と照合
        $result | Should -Be 期待値
    }
}
```

この3つ（`BeforeAll` / `Describe` / `It`）がPesterの骨格です。順番に見ていきましょう。

---

## 2. BeforeAll：テストの準備

### 2-1. ドットソーシング

テスト対象のスクリプトをテストファイルに読み込むには**ドットソーシング（dot sourcing）**を使います。

```powershell
BeforeAll {
    . $PSScriptRoot/Get-Greeting.ps1
}
```

| 書き方 | 意味 |
|---|---|
| `. ファイルパス` | ファイルを現在のスコープに読み込む（関数が使えるようになる） |
| `$PSScriptRoot` | テストファイル自身があるディレクトリの絶対パス |

> **なぜ `$PSScriptRoot` を使うのか？**
> `./Get-Greeting.ps1` と書くと、テストを実行するときのカレントディレクトリに依存します。
> `$PSScriptRoot` を使えばどこから実行しても正しいパスになります。

### 2-2. BeforeAll の注意点（Pester 5.x）

Pester 5.x では、`Describe` の外で変数を定義しても `It` ブロック内からは見えません。
テスト対象の読み込みは必ず `BeforeAll` の中で行ってください。

```powershell
# 悪い例（Pester 5.x では動かない）
. $PSScriptRoot/Get-Greeting.ps1   # Describe の外に書いてはいけない

Describe "Get-Greeting" {
    It "動くかな？" {
        Get-Greeting -Name "太郎"   # エラーになる
    }
}
```

```powershell
# 良い例
BeforeAll {
    . $PSScriptRoot/Get-Greeting.ps1   # BeforeAll の中に書く
}

Describe "Get-Greeting" {
    It "動く" {
        Get-Greeting -Name "太郎"   # 正しく動く
    }
}
```

---

## 3. Describe：テストのグループ化

`Describe`（ディスクライブ）はテストをグループ化するブロックです。

```powershell
Describe "グループの説明文" {
    # ここに It ブロックを書く
}
```

### 3-1. ネスト（入れ子）

`Describe` の中にさらに `Describe`（または `Context`）を入れ子にできます。

```powershell
Describe "Get-Greeting" {

    Describe "正常系" {
        It "名前を渡すと挨拶文を返す" { ... }
        It "長い名前でも動く" { ... }
    }

    Describe "異常系" {
        It "空文字を渡すとエラーになる" { ... }
    }
}
```

### 3-2. Context（コンテキスト）

`Context` は `Describe` と機能的に同じですが、**条件・状況を表す**のに使う慣習があります。

```powershell
Describe "Get-Greeting" {

    Context "名前が指定されているとき" {
        It "挨拶文を返す" { ... }
    }

    Context "名前が空のとき" {
        It "エラーを投げる" { ... }
    }
}
```

> `Describe` = 「何をテストするか」、`Context` = 「どういう状況でテストするか」という使い分けが一般的です。

---

## 4. It：個別テストケース

`It`（イット）は1つのテストケースを表します。

```powershell
It "テストの説明文" {
    # ここに検証コードを書く
}
```

### 4-1. 説明文の書き方

`It` の説明文は「〜であること」「〜を返すこと」という形で書くと読みやすくなります。

```powershell
# 良い例
It "名前を渡すと挨拶文を返すこと" { ... }
It "空文字を渡すと例外を投げること" { ... }

# 悪い例（何を確認しているか不明）
It "テスト1" { ... }
It "check" { ... }
```

### 4-2. テストの結果

`It` ブロックは3つの状態になります：

| 状態 | 記号 | 意味 |
|---|---|---|
| 成功 | `[+]` 緑 | `Should` の検証が全て通った |
| 失敗 | `[-]` 赤 | `Should` の検証が1つでも失敗した |
| スキップ | `[!]` 黄 | `-Skip` オプションで意図的にスキップ |

```powershell
# テストをスキップする
It "未実装のテスト" -Skip {
    $result = 未実装の関数
    $result | Should -Be "何か"
}
```

---

## 5. Should：値の検証

`Should`（シュッド）は実際の値と期待値を比較するコマンドです。パイプ（`|`）でつないで使います。

```powershell
実際の値 | Should -マッチャー 期待値
```

Phase 1では最もよく使う3つのマッチャーを覚えます。詳細はPhase 2で扱います。

### 5-1. -Be：完全一致

```powershell
$result = 1 + 1
$result | Should -Be 2
```

型も含めて完全一致を確認します。

```powershell
# 文字列
"hello" | Should -Be "hello"   # 成功
"hello" | Should -Be "Hello"   # 失敗（大文字小文字を区別する）

# 数値
42 | Should -Be 42             # 成功
42 | Should -Be "42"           # 失敗（型が違う）
```

### 5-2. -BeNullOrEmpty：null または空を確認

```powershell
$value = ""
$value | Should -BeNullOrEmpty   # 成功

$value = $null
$value | Should -BeNullOrEmpty   # 成功

$value = "hello"
$value | Should -BeNullOrEmpty   # 失敗
```

### 5-3. -Throw：例外の検証

例外が発生することを確認するには、コードブロック `{ }` をパイプに渡します。

```powershell
# 例外が投げられることを確認
{ throw "エラー" } | Should -Throw

# 特定のメッセージを含む例外を確認
{ throw "ファイルが見つかりません" } | Should -Throw "*ファイルが見つかりません*"
```

> **重要:** `-Throw` のときは `{ }` でコードを囲むのを忘れないでください。
> 囲まないとコードが先に実行されてしまい、テスト自体がエラーになります。

```powershell
# 悪い例（テストがクラッシュする）
throw "エラー" | Should -Throw

# 良い例
{ throw "エラー" } | Should -Throw
```

---

## 6. 動作するサンプル：全体を通して見る

### 6-1. テスト対象

```powershell
# Calculator.ps1

function Add-Numbers {
    param([int]$A, [int]$B)
    return $A + $B
}

function Divide-Numbers {
    param([int]$Dividend, [int]$Divisor)
    if ($Divisor -eq 0) {
        throw "ゼロ除算はできません"
    }
    return $Dividend / $Divisor
}
```

### 6-2. テストファイル

```powershell
# Calculator.Tests.ps1

BeforeAll {
    . $PSScriptRoot/Calculator.ps1
}

Describe "Add-Numbers" {

    Context "正常な入力のとき" {
        It "正の数を足すと正しい合計を返す" {
            $result = Add-Numbers -A 3 -B 4
            $result | Should -Be 7
        }

        It "負の数を含む場合も正しく計算する" {
            $result = Add-Numbers -A -5 -B 3
            $result | Should -Be -2
        }

        It "両方ゼロのとき 0 を返す" {
            $result = Add-Numbers -A 0 -B 0
            $result | Should -Be 0
        }
    }
}

Describe "Divide-Numbers" {

    Context "正常な入力のとき" {
        It "10 ÷ 2 は 5 を返す" {
            $result = Divide-Numbers -Dividend 10 -Divisor 2
            $result | Should -Be 5
        }
    }

    Context "ゼロ除算のとき" {
        It "例外を投げる" {
            { Divide-Numbers -Dividend 10 -Divisor 0 } | Should -Throw "*ゼロ除算はできません*"
        }
    }
}
```

### 6-3. テストの実行

```powershell
Invoke-Pester -Path "./Calculator.Tests.ps1" -Output Detailed
```

**期待される出力：**

```
Starting discovery in 1 files.
Discovery found 5 tests in 31ms.
Running tests.

Describing Add-Numbers
  Context 正常な入力のとき
    [+] 正の数を足すと正しい合計を返す 12ms (9ms|3ms)
    [+] 負の数を含む場合も正しく計算する 8ms (6ms|2ms)
    [+] 両方ゼロのとき 0 を返す 7ms (5ms|2ms)

Describing Divide-Numbers
  Context 正常な入力のとき
    [+] 10 ÷ 2 は 5 を返す 9ms (7ms|2ms)
  Context ゼロ除算のとき
    [+] 例外を投げる 11ms (8ms|3ms)

Tests completed in 72ms
Tests Passed: 5, Failed: 0, Skipped: 0
```

---

## 7. テストが失敗したときの読み方

失敗すると Pester は詳細なエラーメッセージを表示します。

```powershell
# わざと失敗するテスト
It "わざと失敗させる" {
    $result = Add-Numbers -A 1 -B 1
    $result | Should -Be 99   # 実際は 2 なので失敗する
}
```

**失敗時の出力：**

```
[-] わざと失敗させる 18ms (15ms|3ms)
 Expected: 99
 But was:  2
 at $result | Should -Be 99, ...
```

| 項目 | 意味 |
|---|---|
| `Expected` | テストで期待した値 |
| `But was` | 実際に返ってきた値 |
| `at ...` | 失敗した行 |

この情報をもとにスクリプト側かテスト側のどちらに問題があるかを判断します。

---

## 8. まとめ

```
BeforeAll { }    → テスト前の準備（ファイル読み込み等）
Describe { }     → テストのグループ化（何をテストするか）
Context { }      → 状況による分類（どういう状況でテストするか）
It { }           → 個別テストケース（1つの検証）
Should -Be       → 完全一致の検証
Should -BeNullOrEmpty → null/空の検証
Should -Throw    → 例外発生の検証
```

---

## 練習問題

### 問題 1
次のテストコードには問題があります。何が問題で、どう直せばよいですか？

```powershell
. $PSScriptRoot/MyScript.ps1

Describe "MyFunction" {
    It "正しい値を返す" {
        $result = MyFunction -Input "test"
        $result | Should -Be "TEST"
    }
}
```

### 問題 2
以下の関数に対するテストを書いてください。

```powershell
# StringUtils.ps1
function ConvertTo-UpperCase {
    param([string]$Text)
    if ([string]::IsNullOrEmpty($Text)) {
        throw "テキストが空です"
    }
    return $Text.ToUpper()
}
```

以下のケースをすべてカバーしてください：
1. `"hello"` を渡すと `"HELLO"` が返る
2. `"PowerShell"` を渡すと `"POWERSHELL"` が返る
3. 空文字を渡すと例外が投げられる
4. `$null` を渡すと例外が投げられる

### 問題 3
以下のテスト結果を読んで、何が起きているか説明してください。

```
[-] ユーザー名を取得する 22ms
 Expected: 'admin'
 But was:  'Admin'
 at $result | Should -Be 'admin', ...
```

---

## 練習問題の解答

### 問題 1 の解答

**問題点:** `Describe` の外でドットソーシングしている（Pester 5.x では動作しない）

```powershell
# 修正後
BeforeAll {
    . $PSScriptRoot/MyScript.ps1   # BeforeAll の中に移動する
}

Describe "MyFunction" {
    It "正しい値を返す" {
        $result = MyFunction -Input "test"
        $result | Should -Be "TEST"
    }
}
```

### 問題 2 の解答

```powershell
# StringUtils.Tests.ps1
BeforeAll {
    . $PSScriptRoot/StringUtils.ps1
}

Describe "ConvertTo-UpperCase" {

    Context "有効なテキストのとき" {
        It "'hello' を渡すと 'HELLO' を返す" {
            $result = ConvertTo-UpperCase -Text "hello"
            $result | Should -Be "HELLO"
        }

        It "'PowerShell' を渡すと 'POWERSHELL' を返す" {
            $result = ConvertTo-UpperCase -Text "PowerShell"
            $result | Should -Be "POWERSHELL"
        }
    }

    Context "無効なテキストのとき" {
        It "空文字を渡すと例外を投げる" {
            { ConvertTo-UpperCase -Text "" } | Should -Throw "*テキストが空です*"
        }

        It "null を渡すと例外を投げる" {
            { ConvertTo-UpperCase -Text $null } | Should -Throw "*テキストが空です*"
        }
    }
}
```

### 問題 3 の解答

関数が `'Admin'`（Aが大文字）を返しているが、テストは `'admin'`（全て小文字）を期待しています。

原因として考えられるもの：
- 関数の実装が大文字で始まる文字列を返している
- テストの期待値が間違っている（正しい仕様が `'Admin'` なら `Should -Be 'Admin'` に修正する）

`Should -Be` は大文字小文字を**区別する**ため、このような違いで失敗します。

---

**次のステップ:** [Phase 2](Phase2.md)では `Should` のさまざまなマッチャー（`-Contain` / `-Match` / `-BeLessThan` など）を学びます。

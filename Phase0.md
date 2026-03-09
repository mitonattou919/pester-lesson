# Phase 0: 概念・Why Pester？

## 1. テストとは何か

### 1-1. 手動テストの限界

スクリプトを書いたとき、あなたはどうやって「正しく動く」と確認しますか？

多くの場合、こんな流れになります：

1. スクリプトを実行する
2. 出力を目で確認する
3. 問題なければ完了

これは「手動テスト」です。最初は十分に見えますが、すぐに問題が出てきます。

**よくある問題のシナリオ：**

```
# バックアップスクリプト（backup.ps1）
function Backup-Files {
    param($Source, $Destination)
    Copy-Item -Path $Source -Destination $Destination -Recurse
    Write-Host "バックアップ完了: $Source -> $Destination"
}
```

このスクリプトに後から機能を追加したとします。**既存の動作が壊れていないか**、毎回手動で全パターンを確認しますか？

- コピー元が存在しない場合は？
- コピー先のディスクが満杯の場合は？
- ファイル名に特殊文字が含まれる場合は？

パターンが増えるほど手動確認は現実的でなくなります。

### 1-2. 自動テストの考え方

自動テストとは、「期待する動作」をコードとして記述しておき、機械に検証させることです。

```
期待: Backup-Files を呼んだとき、ファイルがコピーされること
期待: コピー元が存在しなければ、エラーを投げること
```

このような「期待」をコードで書いておけば、スクリプトを変更するたびに自動で検証できます。

### 1-3. テストがもたらすもの

| 手動テスト | 自動テスト |
|---|---|
| 毎回人間が確認 | 一度書けば何度でも再実行 |
| 見落としが起きる | 記述した条件は必ず検証 |
| 時間がかかる | 秒単位で完了 |
| 変更のたびに怖い | 安心してリファクタリングできる |

---

## 2. Pesterとは何か

### 2-1. 位置づけ

**Pester**はPowerShell用のテストフレームワークです。

- PowerShell向けに設計されており、PowerShellスクリプトのテストに最適化されている
- Windows PowerShell 5.1 および PowerShell 7.x の両方で動作する
- Windows 10/11にはデフォルトで同梱されている（バージョンは古い場合がある）
- 現在の安定版は **Pester 5.x系**（このコースでは5.x系を対象とする）

### 2-2. Pesterのインストール・バージョン確認

```powershell
# インストール済みバージョンを確認
Get-Module -Name Pester -ListAvailable

# 最新版をインストール（または更新）
Install-Module -Name Pester -Force -SkipPublisherCheck

# バージョン確認
Import-Module Pester -PassThru | Select-Object -ExpandProperty Version
```

> **注意:** Windows同梱のPesterは3.x系であることが多いです。このコースでは5.x系が必要なので、上記コマンドでインストールしてください。

### 2-3. Pesterの基本的な見た目

Pesterで書いたテストはこのような形をしています：

```powershell
Describe "Get-Greeting 関数のテスト" {
    It "名前を渡すと挨拶文を返す" {
        $result = Get-Greeting -Name "太郎"
        $result | Should -Be "こんにちは、太郎さん"
    }

    It "名前が空のときエラーを投げる" {
        { Get-Greeting -Name "" } | Should -Throw
    }
}
```

日本語で読むとこうなります：

```
【説明】Get-Greeting 関数のテスト
  ・名前を渡すと挨拶文を返す  → 検証: 結果が "こんにちは、太郎さん" であること
  ・名前が空のときエラーを投げる  → 検証: 例外が発生すること
```

英語の構文がそのまま「仕様書」として読めるのがPesterの特徴です。

---

## 3. Pesterを使うべき理由（実務での価値）

### 3-1. 変更に強くなる

スクリプトに手を加えるとき、テストがあれば「壊していない」という確信を持てます。これを**リグレッション（退行）防止**と呼びます。

### 3-2. 仕様をコードで表現できる

「このスクリプトは何をするのか」をテストが示します。コメントと違い、テストは実際に実行されるので古くなりません。

### 3-3. 設計が自然と良くなる

テストを書きにくいコードは、たいていの場合「設計が複雑すぎる」コードです。Pesterでテストを書こうとすると、自然と関数を小さく分けるようになります。

### 3-4. チームでの共有が容易

CI/CDパイプライン（GitHub Actions等）にPesterを組み込むことで、プルリクエストのたびに自動でテストを実行できます。

---

## 4. テストファイルの命名規則

Pesterにはファイル名の慣習があります：

```
スクリプト本体:  Get-Greeting.ps1
テストファイル:  Get-Greeting.Tests.ps1
```

`.Tests.ps1` というサフィックスがPesterのテストファイルの目印です。Pesterはこのパターンのファイルを自動的に検出します。

```
プロジェクト構成の例:
MyProject/
├── src/
│   ├── Get-Greeting.ps1
│   └── Backup-Files.ps1
└── tests/
    ├── Get-Greeting.Tests.ps1
    └── Backup-Files.Tests.ps1
```

---

## 5. テストを実行する（先取り体験）

Phase 0の最後に、実際にPesterを動かしてみましょう。

### 5-0. ディレクトリ構造を用意する

この演習では、テスト対象のスクリプトとテストファイルを**同じフォルダ**に置きます。

```
PesterDemo/          ← 新しく作るフォルダ
├── Get-Greeting.ps1
└── Get-Greeting.Tests.ps1
```

> **なぜ同じフォルダ？**
> テストファイル内の `$PSScriptRoot` は「テストファイル自身が置かれているフォルダ」を指します。
> 両ファイルが同じ場所にあれば、`$PSScriptRoot/Get-Greeting.ps1` で確実に本体を読み込めます。
> （`src/` と `tests/` を分ける構成はPhase 1以降で扱います）

まず作業用フォルダを作り、そこに移動してください：

```powershell
New-Item -ItemType Directory -Name PesterDemo
Set-Location PesterDemo
```

### 5-1. テスト対象のスクリプトを作成

```powershell
# Get-Greeting.ps1
function Get-Greeting {
    param(
        [string]$Name
    )
    return "こんにちは、${Name}さん"
}
```

### 5-2. テストファイルを作成

```powershell
# Get-Greeting.Tests.ps1
BeforeAll {
    . $PSScriptRoot/Get-Greeting.ps1  # テスト対象を読み込む（ドットソーシング）
}

Describe "Get-Greeting" {
    It "名前を渡すと挨拶文を返す" {
        $result = Get-Greeting -Name "太郎"
        $result | Should -Be "こんにちは、太郎さん"
    }
}
```

### 5-3. テストを実行

```powershell
# テストファイルを指定して実行
Invoke-Pester -Path "./Get-Greeting.Tests.ps1" -Output Detailed
```

**出力例：**

```
Starting discovery in 1 files.
Discovery found 1 tests in 22ms.
Running tests.

Describing Get-Greeting
  [+] 名前を渡すと挨拶文を返す 15ms (11ms|4ms)

Tests completed in 53ms
Tests Passed: 1, Failed: 0, Skipped: 0
```

`[+]` が緑色で表示されれば成功です。

---

## 6. まとめ

| キーワード | 意味 |
|---|---|
| テスト | 期待する動作をコードで記述し、自動的に検証する仕組み |
| Pester | PowerShell向けのテストフレームワーク |
| `.Tests.ps1` | Pesterテストファイルの命名規則 |
| `Invoke-Pester` | テストを実行するコマンド |
| `Describe` | テストのグループ（後続フェーズで詳解） |
| `It` | 個別のテストケース（後続フェーズで詳解） |
| `Should` | 検証を行うコマンド（後続フェーズで詳解） |

---

## 練習問題

### 問題 1
次のうち、Pesterのテストファイルとして正しい命名はどれですか？

```
a) MyScript.test.ps1
b) MyScript.Tests.ps1
c) Test-MyScript.ps1
d) MyScript_test.ps1
```

### 問題 2
以下の関数に対するテストファイルを作成してください。

```powershell
# Add-Numbers.ps1
function Add-Numbers {
    param([int]$A, [int]$B)
    return $A + $B
}
```

テストファイル `Add-Numbers.Tests.ps1` を作成し、以下を検証してください：
- `Add-Numbers -A 3 -B 4` の結果が `7` であること
- `Add-Numbers -A 0 -B 0` の結果が `0` であること

### 問題 3（考察）
あなたが今まで書いたスクリプト（または業務で使っているスクリプト）を思い浮かべてください。そのスクリプトに自動テストがあれば、どんな場面で役立ちますか？

---

## 練習問題の解答

### 問題 1 の解答
**b) MyScript.Tests.ps1** が正解です。

Pesterは `.Tests.ps1` というサフィックスを持つファイルをテストファイルとして認識します。

### 問題 2 の解答

```powershell
# Add-Numbers.Tests.ps1
BeforeAll {
    . $PSScriptRoot/Add-Numbers.ps1
}

Describe "Add-Numbers" {
    It "3 + 4 は 7 を返す" {
        $result = Add-Numbers -A 3 -B 4
        $result | Should -Be 7
    }

    It "0 + 0 は 0 を返す" {
        $result = Add-Numbers -A 0 -B 0
        $result | Should -Be 0
    }
}
```

---

**次のステップ:** Phase 1では `Describe` / `It` / `Should` の基礎構文を詳しく学びます。

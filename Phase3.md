# Phase 3: セットアップ／ティアダウン

## 1. なぜセットアップ／ティアダウンが必要か

複数のテストで同じ準備処理が必要になることはよくあります。

```powershell
# 準備コードが各 It に重複している（悪い例）
Describe "ファイル操作のテスト" {
    It "ファイルを読める" {
        New-Item -Path "C:\Temp\test.txt" -Value "hello"   # 準備
        # テスト...
        Remove-Item -Path "C:\Temp\test.txt"               # 後片付け
    }

    It "ファイルを書き換えられる" {
        New-Item -Path "C:\Temp\test.txt" -Value "hello"   # 同じ準備が重複
        # テスト...
        Remove-Item -Path "C:\Temp\test.txt"               # 同じ後片付けが重複
    }
}
```

これには2つの問題があります：
1. **重複**：同じコードが何度も登場する
2. **不安定**：テストが途中で失敗すると後片付けが実行されず、次のテストに影響する

セットアップ／ティアダウンブロックを使うと、これをきれいに解決できます。

---

## 2. 4つのブロック

Pesterには4つのセットアップ／ティアダウンブロックがあります。

| ブロック | 実行タイミング |
|---|---|
| `BeforeAll` | `Describe`（または`Context`）内の**全テストの前に1回** |
| `AfterAll` | `Describe`（または`Context`）内の**全テストの後に1回** |
| `BeforeEach` | `Describe`（または`Context`）内の**各 `It` の前に毎回** |
| `AfterEach` | `Describe`（または`Context`）内の**各 `It` の後に毎回** |

```
BeforeAll     ← 1回だけ
  BeforeEach  ← It のたびに
  It "テスト1"
  AfterEach   ← It のたびに

  BeforeEach  ← It のたびに
  It "テスト2"
  AfterEach   ← It のたびに

  BeforeEach  ← It のたびに
  It "テスト3"
  AfterEach   ← It のたびに
AfterAll      ← 1回だけ
```

---

## 3. BeforeAll / AfterAll

### 3-1. 基本的な使い方

```powershell
Describe "データベース接続テスト" {
    BeforeAll {
        # テスト全体の前に1回だけ実行される
        $script:connection = Open-DatabaseConnection -Server "localhost"
    }

    AfterAll {
        # テスト全体の後に1回だけ実行される
        Close-DatabaseConnection -Connection $script:connection
    }

    It "クエリを実行できる" {
        $result = Invoke-Query -Connection $script:connection -Query "SELECT 1"
        $result | Should -Not -BeNullOrEmpty
    }

    It "テーブルが存在する" {
        $tables = Get-Tables -Connection $script:connection
        $tables | Should -Contain "Users"
    }
}
```

### 3-2. $script: スコープについて

Pester 5.x では `BeforeAll` 内で定義した変数を `It` から参照するには **`$script:` スコープ**を使います。

```powershell
Describe "スコープのサンプル" {
    BeforeAll {
        $script:value = "共有する値"   # $script: を付ける
        $localValue   = "ローカルな値" # これは It から見えない
    }

    It "script スコープの変数は見える" {
        $script:value | Should -Be "共有する値"   # 成功
    }

    It "ローカル変数は見えない" {
        $localValue | Should -BeNullOrEmpty        # $localValue は $null になる
    }
}
```

> **重要:** `BeforeAll` 内の変数は `$script:変数名` で定義し、`It` 内でも `$script:変数名` でアクセスします。これはPester 5.xの仕様です。

---

## 4. BeforeEach / AfterEach

### 4-1. 基本的な使い方

```powershell
Describe "ファイル操作テスト" {
    BeforeEach {
        # 各テストの前にクリーンなファイルを用意する
        $script:testFile = Join-Path $TestDrive "test.txt"
        Set-Content -Path $script:testFile -Value "初期内容"
    }

    AfterEach {
        # 各テストの後にファイルを削除する
        if (Test-Path $script:testFile) {
            Remove-Item $script:testFile
        }
    }

    It "ファイルの内容を読める" {
        $content = Get-Content -Path $script:testFile
        $content | Should -Be "初期内容"
    }

    It "ファイルの内容を書き換えられる" {
        Set-Content -Path $script:testFile -Value "新しい内容"
        $content = Get-Content -Path $script:testFile
        $content | Should -Be "新しい内容"
    }

    It "書き換えても次のテストには影響しない" {
        # BeforeEach が毎回リセットするので、前のテストの影響を受けない
        $content = Get-Content -Path $script:testFile
        $content | Should -Be "初期内容"   # 必ず初期状態になっている
    }
}
```

### 4-2. $TestDrive：Pesterの一時ディレクトリ

`$TestDrive` はPesterが自動で用意する一時ディレクトリです。

```powershell
BeforeEach {
    # $TestDrive はテストごとに自動でクリーンアップされる
    $script:file = Join-Path $TestDrive "sample.txt"
    Set-Content -Path $script:file -Value "テスト用データ"
}
```

| | 通常の一時ファイル | `$TestDrive` |
|---|---|---|
| 場所 | 自分で指定 | Pesterが自動作成 |
| クリーンアップ | 自分でAfterEachに書く | テスト終了後に自動削除 |
| 安全性 | テスト間で汚染される可能性 | テストごとに分離 |

---

## 5. ネストしたブロックの実行順序

`Describe` の中に `Context` がある場合、ブロックはどの順で実行されるか確認します。

```powershell
Describe "外側" {
    BeforeAll  { Write-Host "外側 BeforeAll" }
    AfterAll   { Write-Host "外側 AfterAll" }
    BeforeEach { Write-Host "外側 BeforeEach" }
    AfterEach  { Write-Host "外側 AfterEach" }

    Context "内側" {
        BeforeAll  { Write-Host "内側 BeforeAll" }
        AfterAll   { Write-Host "内側 AfterAll" }
        BeforeEach { Write-Host "内側 BeforeEach" }
        AfterEach  { Write-Host "内側 AfterEach" }

        It "テスト A" { Write-Host "テスト A" }
        It "テスト B" { Write-Host "テスト B" }
    }
}
```

**実行順序：**

```
外側 BeforeAll
  内側 BeforeAll
    外側 BeforeEach → 内側 BeforeEach → テスト A → 内側 AfterEach → 外側 AfterEach
    外側 BeforeEach → 内側 BeforeEach → テスト B → 内側 AfterEach → 外側 AfterEach
  内側 AfterAll
外側 AfterAll
```

外側から内側へ「包み込む」順序になります。

---

## 6. 使い分けの判断基準

```
準備コストが高い（DB接続、ファイル生成など）
かつ テスト間で状態を共有してよい
    → BeforeAll / AfterAll

各テストが独立した状態で動く必要がある
かつ テストが互いに影響してはいけない
    → BeforeEach / AfterEach
```

**実務でよく見るパターン：**

```powershell
Describe "統合テスト" {
    BeforeAll {
        # コストの高い準備は1回だけ
        $script:tempDir = New-Item -ItemType Directory -Path (Join-Path $TestDrive "work")
        $script:config  = Import-PowerShellDataFile "./config.psd1"
    }

    AfterAll {
        # 後片付けも1回だけ
        Remove-Item $script:tempDir -Recurse -ErrorAction SilentlyContinue
    }

    BeforeEach {
        # 各テストの前に状態をリセット
        Get-ChildItem $script:tempDir | Remove-Item
    }

    It "テスト1" { ... }
    It "テスト2" { ... }
    It "テスト3" { ... }
}
```

---

## 7. 動作するサンプル：ファイル操作テスト

### テスト対象

```powershell
# FileProcessor.ps1

function Write-LogEntry {
    param(
        [string]$LogFile,
        [string]$Message,
        [ValidateSet("INFO", "WARN", "ERROR")]
        [string]$Level = "INFO"
    )

    if (-not (Test-Path (Split-Path $LogFile -Parent))) {
        throw "ログディレクトリが存在しません"
    }

    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $entry = "[$timestamp] [$Level] $Message"
    Add-Content -Path $LogFile -Value $entry
}

function Get-LogEntries {
    param(
        [string]$LogFile,
        [string]$Level
    )

    if (-not (Test-Path $LogFile)) {
        return @()
    }

    $lines = Get-Content -Path $LogFile
    if ($Level) {
        return $lines | Where-Object { $_ -match "\[$Level\]" }
    }
    return $lines
}
```

### テストファイル

```powershell
# FileProcessor.Tests.ps1

BeforeAll {
    . $PSScriptRoot/FileProcessor.ps1
}

Describe "Write-LogEntry" {

    BeforeAll {
        # ログディレクトリを1回だけ作成
        $script:logDir  = Join-Path $TestDrive "logs"
        $script:logFile = Join-Path $script:logDir "app.log"
        New-Item -ItemType Directory -Path $script:logDir
    }

    BeforeEach {
        # 各テストの前にログファイルをリセット
        if (Test-Path $script:logFile) {
            Remove-Item $script:logFile
        }
    }

    It "ログエントリがファイルに書き込まれる" {
        Write-LogEntry -LogFile $script:logFile -Message "起動しました"
        $script:logFile | Should -Exist
    }

    It "書き込んだメッセージが含まれる" {
        Write-LogEntry -LogFile $script:logFile -Message "テストメッセージ"
        $script:logFile | Should -FileContentMatch "テストメッセージ"
    }

    It "デフォルトのレベルは INFO である" {
        Write-LogEntry -LogFile $script:logFile -Message "情報ログ"
        $script:logFile | Should -FileContentMatch "\[INFO\]"
    }

    It "WARN レベルを指定できる" {
        Write-LogEntry -LogFile $script:logFile -Message "警告" -Level "WARN"
        $script:logFile | Should -FileContentMatch "\[WARN\]"
    }

    It "ディレクトリが存在しない場合は例外を投げる" {
        $badPath = Join-Path $TestDrive "nonexistent\app.log"
        { Write-LogEntry -LogFile $badPath -Message "test" } |
            Should -Throw "*ログディレクトリが存在しません*"
    }

    It "複数書き込んでも全エントリが保持される" {
        Write-LogEntry -LogFile $script:logFile -Message "1行目"
        Write-LogEntry -LogFile $script:logFile -Message "2行目"
        Write-LogEntry -LogFile $script:logFile -Message "3行目"

        $entries = Get-Content $script:logFile
        $entries | Should -HaveCount 3
    }
}

Describe "Get-LogEntries" {

    BeforeAll {
        $script:logDir  = Join-Path $TestDrive "logs2"
        $script:logFile = Join-Path $script:logDir "app.log"
        New-Item -ItemType Directory -Path $script:logDir

        # テスト用のログを事前に書き込む（全テストで共有）
        Write-LogEntry -LogFile $script:logFile -Message "情報1" -Level "INFO"
        Write-LogEntry -LogFile $script:logFile -Message "警告1" -Level "WARN"
        Write-LogEntry -LogFile $script:logFile -Message "情報2" -Level "INFO"
        Write-LogEntry -LogFile $script:logFile -Message "エラー1" -Level "ERROR"
    }

    It "レベル未指定のとき全エントリを返す" {
        $entries = Get-LogEntries -LogFile $script:logFile
        $entries | Should -HaveCount 4
    }

    It "INFO フィルターで INFO エントリだけ返す" {
        $entries = Get-LogEntries -LogFile $script:logFile -Level "INFO"
        $entries | Should -HaveCount 2
    }

    It "ERROR フィルターで ERROR エントリだけ返す" {
        $entries = Get-LogEntries -LogFile $script:logFile -Level "ERROR"
        $entries | Should -HaveCount 1
        $entries[0] | Should -Match "エラー1"
    }

    It "存在しないファイルを指定すると空配列を返す" {
        $entries = Get-LogEntries -LogFile (Join-Path $TestDrive "ghost.log")
        $entries | Should -HaveCount 0
    }
}
```

---

## 8. よくある間違いと対策

### 間違い 1：BeforeEach で共有すべきものを定義してしまう

```powershell
# 悪い例：毎回 DB 接続を作り直すのはコスト大
BeforeEach {
    $script:conn = Open-HeavyDatabaseConnection   # 毎回実行される
}
AfterEach {
    Close-DatabaseConnection $script:conn         # 毎回実行される
}
```

```powershell
# 良い例：接続は1回だけ、状態のリセットだけ毎回行う
BeforeAll {
    $script:conn = Open-HeavyDatabaseConnection   # 1回だけ
}
AfterAll {
    Close-DatabaseConnection $script:conn         # 1回だけ
}
BeforeEach {
    Clear-TestData -Connection $script:conn       # 状態だけリセット
}
```

### 間違い 2：AfterAll を書き忘れてテスト環境を汚染する

```powershell
# 悪い例：後片付けなし
BeforeAll {
    New-Item -Path "C:\Temp\TestOutput" -ItemType Directory
}
# AfterAll がない → テスト後もフォルダが残り続ける
```

```powershell
# 良い例：必ず後片付けをする
BeforeAll {
    $script:tempDir = New-Item -Path (Join-Path $TestDrive "output") -ItemType Directory
}
AfterAll {
    Remove-Item $script:tempDir -Recurse -ErrorAction SilentlyContinue
}
```

> `$TestDrive` を使えばPesterが自動でクリーンアップするため、`AfterAll` での削除を書かずに済みます。

### 間違い 3：$script: を忘れる

```powershell
# 悪い例
BeforeAll {
    $config = Load-Config   # $script: がない
}
It "設定が読める" {
    $config | Should -Not -BeNullOrEmpty   # $config は $null → 失敗
}
```

```powershell
# 良い例
BeforeAll {
    $script:config = Load-Config           # $script: を付ける
}
It "設定が読める" {
    $script:config | Should -Not -BeNullOrEmpty   # 正しく参照できる
}
```

---

## 9. まとめ

```
BeforeAll  → Describe/Context 内で1回だけ実行（重い初期化に使う）
AfterAll   → Describe/Context 内で1回だけ実行（後片付けに使う）
BeforeEach → 各 It の前に毎回実行（状態のリセットに使う）
AfterEach  → 各 It の後に毎回実行（テスト後のクリーンアップに使う）

$script:変数名  → BeforeAll/BeforeEach から It に値を渡すスコープ
$TestDrive      → Pester が用意する自動クリーンアップ付き一時ディレクトリ

実行順序（ネスト時）:
  外側BeforeAll → 内側BeforeAll →
    [外側BeforeEach → 内側BeforeEach → It → 内側AfterEach → 外側AfterEach] × テスト数
  → 内側AfterAll → 外側AfterAll
```

---

## 練習問題

### 問題 1
以下のテストコードには2つの問題があります。それぞれ何が問題で、どう直せばよいですか？

```powershell
Describe "設定ファイルのテスト" {
    BeforeAll {
        $configPath = Join-Path $TestDrive "config.json"
        '{"key": "value"}' | Set-Content -Path $configPath
    }

    It "設定ファイルが存在する" {
        $configPath | Should -Exist
    }

    It "key が value である" {
        $config = Get-Content $configPath | ConvertFrom-Json
        $config.key | Should -Be "value"
    }
}
```

### 問題 2
以下の仕様を持つ関数のテストを `BeforeAll` / `BeforeEach` / `AfterEach` を適切に使って書いてください。

```powershell
# CsvProcessor.ps1
function Import-CsvData {
    param([string]$Path)
    if (-not (Test-Path $Path)) {
        throw "ファイルが見つかりません: $Path"
    }
    return Import-Csv -Path $Path
}

function Export-CsvData {
    param([object[]]$Data, [string]$Path)
    $Data | Export-Csv -Path $Path -NoTypeInformation -Encoding UTF8
}
```

テストのシナリオ：
1. 正常なCSVファイルをインポートできる
2. 存在しないファイルを指定すると例外が発生する
3. データをエクスポートするとファイルが作成される
4. エクスポートしたファイルを再インポートすると元のデータと一致する

### 問題 3（考察）
`BeforeAll` と `BeforeEach` を使い分ける基準を自分の言葉で説明してください。

---

## 練習問題の解答

### 問題 1 の解答

**問題点：**
1. `BeforeAll` 内の `$configPath` に `$script:` スコープがないため、`It` から参照できない
2. `AfterAll` がなく、後片付けが定義されていない（`$TestDrive` を使っているので致命的ではないが、明示的に書くのがよい慣習）

```powershell
# 修正後
Describe "設定ファイルのテスト" {
    BeforeAll {
        $script:configPath = Join-Path $TestDrive "config.json"   # $script: を追加
        '{"key": "value"}' | Set-Content -Path $script:configPath
    }

    # $TestDrive は自動クリーンアップされるが、明示しても良い
    # AfterAll {
    #     Remove-Item $script:configPath -ErrorAction SilentlyContinue
    # }

    It "設定ファイルが存在する" {
        $script:configPath | Should -Exist
    }

    It "key が value である" {
        $config = Get-Content $script:configPath | ConvertFrom-Json
        $config.key | Should -Be "value"
    }
}
```

### 問題 2 の解答

```powershell
# CsvProcessor.Tests.ps1
BeforeAll {
    . $PSScriptRoot/CsvProcessor.ps1

    # テスト全体で使うサンプルデータ（変更しないので BeforeAll で定義）
    $script:sampleData = @(
        [PSCustomObject]@{ Name = "Alice"; Age = "30" }
        [PSCustomObject]@{ Name = "Bob";   Age = "25" }
    )
}

Describe "Import-CsvData" {

    BeforeEach {
        # 各テスト用にクリーンなCSVファイルを用意
        $script:csvFile = Join-Path $TestDrive "data.csv"
        $script:sampleData | Export-Csv -Path $script:csvFile -NoTypeInformation -Encoding UTF8
    }

    AfterEach {
        if (Test-Path $script:csvFile) {
            Remove-Item $script:csvFile
        }
    }

    It "正常なCSVファイルをインポートできる" {
        $result = Import-CsvData -Path $script:csvFile
        $result | Should -HaveCount 2
    }

    It "存在しないファイルを指定すると例外が発生する" {
        { Import-CsvData -Path (Join-Path $TestDrive "ghost.csv") } |
            Should -Throw "*ファイルが見つかりません*"
    }
}

Describe "Export-CsvData" {

    BeforeEach {
        $script:outputFile = Join-Path $TestDrive "output.csv"
    }

    AfterEach {
        if (Test-Path $script:outputFile) {
            Remove-Item $script:outputFile
        }
    }

    It "エクスポートするとファイルが作成される" {
        Export-CsvData -Data $script:sampleData -Path $script:outputFile
        $script:outputFile | Should -Exist
    }

    It "エクスポート後に再インポートすると元のデータと一致する" {
        Export-CsvData -Data $script:sampleData -Path $script:outputFile
        $reimported = Import-CsvData -Path $script:outputFile
        $reimported | Should -HaveCount 2
        $reimported[0].Name | Should -Be "Alice"
        $reimported[1].Name | Should -Be "Bob"
    }
}
```

### 問題 3 の解答

**`BeforeAll`：** テスト全体で共有できる状態を1回だけ用意するときに使う。DB接続の確立、設定ファイルの読み込み、変更されないテストデータの準備など、コストが高くて毎回やり直す必要がないものが対象。

**`BeforeEach`：** 各テストが独立した状態から始まる必要があるときに使う。テストの実行順序に関わらず結果が変わらないよう、テストごとに状態をリセットするのが目的。ファイルの初期化、変数の再設定などが対象。

**判断の問いかけ：** 「このテストで状態が変わったとき、次のテストに影響してほしくないか？」→ Yes なら `BeforeEach`。「準備に時間がかかるが全テストで同じものを使えるか？」→ Yes なら `BeforeAll`。

---

**次のステップ:** Phase 4では `Mock` を使った外部依存の切り離し方を学びます。

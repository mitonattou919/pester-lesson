# Phase 4: モック

## 1. なぜモックが必要か

### 1-1. 外部依存の問題

実務のスクリプトには必ず「外部への依存」があります。

```powershell
function Send-AlertMail {
    param([string]$Message)

    $body = "アラート: $Message"
    Send-MailMessage -To "admin@example.com" -Subject "Alert" -Body $body -SmtpServer "smtp.example.com"
    Write-Log -Message "メール送信: $Message"
}
```

このスクリプトをテストしようとすると：

- **メールサーバー**が必要（テスト環境にない）
- テストのたびに**実際のメールが送信**される
- ネットワーク障害で**テストが不安定**になる
- ログファイルへの**副作用**が残る

これらの問題を解決するのが**モック（Mock）**です。

### 1-2. モックとは

モックとは「本物の代わりに使う偽物の関数」です。

```
テストなし:  Send-AlertMail → Send-MailMessage（実際に送信）
モックあり:  Send-AlertMail → Mock Send-MailMessage（何もしない偽物）
```

モックを使うと：
- 外部システムへの実際のアクセスなしにテストできる
- 「この関数が呼ばれたか」「どんな引数で呼ばれたか」を検証できる
- 外部システムのエラーをシミュレートできる

---

## 2. Mock の基本

### 2-1. 基本構文

```powershell
Mock コマンド名 { 代わりに実行するコード }
```

```powershell
Describe "Send-AlertMail のテスト" {
    BeforeAll {
        . $PSScriptRoot/AlertMailer.ps1
    }

    It "Send-MailMessage が呼ばれる" {
        # Send-MailMessage を何もしない偽物に差し替える
        Mock Send-MailMessage {}

        Send-AlertMail -Message "ディスク容量不足"

        # 偽物が呼ばれたことを確認
        Should -Invoke Send-MailMessage -Times 1
    }
}
```

### 2-2. Mock の有効範囲

`Mock` は定義された `Describe` / `Context` ブロックの中でのみ有効です。
ブロックを抜けると自動的に元の関数に戻ります。

```powershell
Describe "外側" {
    Mock Get-Date { return [DateTime]"2026-01-01" }   # ここで有効

    It "日付が固定される" {
        (Get-Date).Year | Should -Be 2026   # 成功
    }

    Context "内側" {
        It "内側でも有効" {
            (Get-Date).Year | Should -Be 2026   # 成功（外側のモックが引き継がれる）
        }
    }
}

# ここでは Get-Date は本物に戻っている
```

---

## 3. Should -Invoke：呼び出しの検証

### 3-1. 基本的な検証

```powershell
# 1回だけ呼ばれたことを確認
Should -Invoke コマンド名 -Times 1

# 2回呼ばれたことを確認
Should -Invoke コマンド名 -Times 2

# 1回以上呼ばれたことを確認
Should -Invoke コマンド名 -Times 1 -AtLeast

# まったく呼ばれていないことを確認
Should -Invoke コマンド名 -Times 0

# 呼ばれたかどうかだけ確認（回数は問わない）
Should -Invoke コマンド名
```

### 3-2. 引数の検証：-ParameterFilter

特定の引数で呼ばれたかどうかを `{ }` フィルターで確認できます。

```powershell
Describe "メール送信のテスト" {
    BeforeAll {
        . $PSScriptRoot/AlertMailer.ps1
    }

    It "正しい宛先で送信される" {
        Mock Send-MailMessage {}

        Send-AlertMail -Message "テストアラート"

        Should -Invoke Send-MailMessage -Times 1 -ParameterFilter {
            $To -eq "admin@example.com"
        }
    }

    It "メッセージ本文にアラート内容が含まれる" {
        Mock Send-MailMessage {}

        Send-AlertMail -Message "CPU使用率が高い"

        Should -Invoke Send-MailMessage -Times 1 -ParameterFilter {
            $Body -like "*CPU使用率が高い*"
        }
    }
}
```

---

## 4. 戻り値を持つモック

### 4-1. 固定値を返す

```powershell
# Get-Content を常に固定の文字列を返す偽物に差し替える
Mock Get-Content { return "mock file content" }

# Get-Date を固定の日付を返す偽物に差し替える
Mock Get-Date { return [DateTime]"2026-01-01 12:00:00" }
```

### 4-2. オブジェクトを返す

```powershell
Mock Get-Process {
    return [PSCustomObject]@{
        Name        = "notepad"
        Id          = 1234
        CPU         = 0.5
        WorkingSet  = 10485760
    }
}
```

### 4-3. 引数に応じて異なる値を返す

`-ParameterFilter` を複数定義することで、引数によって振る舞いを変えられます。

```powershell
# "prod" サーバーと "dev" サーバーで異なる設定を返す
Mock Get-ServerConfig {
    return [PSCustomObject]@{ MaxConnections = 100; Timeout = 30 }
} -ParameterFilter { $ServerName -eq "prod" }

Mock Get-ServerConfig {
    return [PSCustomObject]@{ MaxConnections = 10; Timeout = 5 }
} -ParameterFilter { $ServerName -eq "dev" }
```

### 4-4. 例外を投げるモック

外部サービスのエラーをシミュレートするときに使います。

```powershell
Mock Invoke-RestMethod { throw "接続タイムアウト" }

It "API接続失敗時にエラーを処理する" {
    Mock Invoke-RestMethod { throw "接続タイムアウト" }

    { Get-DataFromApi -Url "https://api.example.com" } |
        Should -Throw "*接続タイムアウト*"
}
```

---

## 5. モックの対象になるもの

### 5-1. PowerShell 組み込みコマンド

```powershell
Mock Get-Date       { return [DateTime]"2026-01-01" }
Mock Get-Content    { return "偽のファイル内容" }
Mock Test-Path      { return $true }
Mock New-Item       {}
Mock Remove-Item    {}
Mock Write-Host     {}
```

### 5-2. 自作関数（同じスコープに読み込まれているもの）

```powershell
BeforeAll {
    . $PSScriptRoot/MyModule.ps1
}

Mock Write-Log {}   # MyModule.ps1 内の Write-Log をモック化
```

### 5-3. モジュールのコマンド：-ModuleName

別モジュールの関数をモックするときは `-ModuleName` を指定します。

```powershell
Mock Get-ADUser { return [PSCustomObject]@{ Name = "TestUser" } } `
    -ModuleName ActiveDirectory
```

---

## 6. 動作するサンプル：外部依存を持つスクリプト

### テスト対象

```powershell
# SystemMonitor.ps1

function Get-DiskAlert {
    param(
        [string]$DriveLetter = "C",
        [int]$ThresholdPercent = 80
    )

    $disk = Get-PSDrive -Name $DriveLetter -ErrorAction Stop
    $totalGB = [math]::Round(($disk.Used + $disk.Free) / 1GB, 1)
    $usedPercent = [math]::Round(($disk.Used / ($disk.Used + $disk.Free)) * 100, 1)

    if ($usedPercent -ge $ThresholdPercent) {
        $message = "ディスク使用率が ${usedPercent}% に達しました（閾値: ${ThresholdPercent}%）"
        Write-EventLog -LogName Application -Source "SystemMonitor" `
            -EventId 1001 -EntryType Warning -Message $message
        return [PSCustomObject]@{
            DriveLetter    = $DriveLetter
            UsedPercent    = $usedPercent
            TotalGB        = $totalGB
            AlertTriggered = $true
            Message        = $message
        }
    }

    return [PSCustomObject]@{
        DriveLetter    = $DriveLetter
        UsedPercent    = $usedPercent
        TotalGB        = $totalGB
        AlertTriggered = $false
        Message        = $null
    }
}
```

### テストファイル

```powershell
# SystemMonitor.Tests.ps1

BeforeAll {
    . $PSScriptRoot/SystemMonitor.ps1
}

Describe "Get-DiskAlert" {

    Context "使用率が閾値未満のとき" {
        BeforeAll {
            # Get-PSDrive を偽物に差し替え（使用率 50% 相当）
            Mock Get-PSDrive {
                return [PSCustomObject]@{
                    Name = "C"
                    Used = 50GB
                    Free = 50GB
                }
            }
            # Write-EventLog も偽物に（呼ばれないはずだが念のため）
            Mock Write-EventLog {}
        }

        It "AlertTriggered が false である" {
            $result = Get-DiskAlert -DriveLetter "C" -ThresholdPercent 80
            $result.AlertTriggered | Should -BeFalse
        }

        It "Message が null である" {
            $result = Get-DiskAlert -DriveLetter "C" -ThresholdPercent 80
            $result.Message | Should -BeNullOrEmpty
        }

        It "Write-EventLog は呼ばれない" {
            Get-DiskAlert -DriveLetter "C" -ThresholdPercent 80
            Should -Invoke Write-EventLog -Times 0
        }
    }

    Context "使用率が閾値以上のとき" {
        BeforeAll {
            # 使用率 90% 相当のディスクをシミュレート
            Mock Get-PSDrive {
                return [PSCustomObject]@{
                    Name = "C"
                    Used = 90GB
                    Free = 10GB
                }
            }
            Mock Write-EventLog {}
        }

        It "AlertTriggered が true である" {
            $result = Get-DiskAlert -DriveLetter "C" -ThresholdPercent 80
            $result.AlertTriggered | Should -BeTrue
        }

        It "Message にパーセンテージが含まれる" {
            $result = Get-DiskAlert -DriveLetter "C" -ThresholdPercent 80
            $result.Message | Should -Match "\d+(\.\d+)?%"
        }

        It "Write-EventLog が1回呼ばれる" {
            Get-DiskAlert -DriveLetter "C" -ThresholdPercent 80
            Should -Invoke Write-EventLog -Times 1
        }

        It "Write-EventLog に正しいイベント ID が渡される" {
            Get-DiskAlert -DriveLetter "C" -ThresholdPercent 80
            Should -Invoke Write-EventLog -Times 1 -ParameterFilter {
                $EventId -eq 1001
            }
        }

        It "Write-EventLog に Warning エントリタイプが渡される" {
            Get-DiskAlert -DriveLetter "C" -ThresholdPercent 80
            Should -Invoke Write-EventLog -Times 1 -ParameterFilter {
                $EntryType -eq "Warning"
            }
        }
    }

    Context "ドライブが存在しないとき" {
        BeforeAll {
            Mock Get-PSDrive { throw "ドライブ 'Z' が見つかりません" }
        }

        It "例外を投げる" {
            { Get-DiskAlert -DriveLetter "Z" } |
                Should -Throw "*ドライブ 'Z' が見つかりません*"
        }
    }
}
```

---

## 7. MockWith：$args を使った動的なモック

モック内で元のコマンドに渡された引数を参照できます。

```powershell
Mock Get-Item {
    # $Path は実際に渡された引数の値
    return [PSCustomObject]@{
        FullName  = $Path
        Extension = [System.IO.Path]::GetExtension($Path)
        Exists    = $true
    }
}
```

---

## 8. Verifiable Mock：必ず呼ばれることを強制する

`-Verifiable` を付けると、テスト後に `Should -InvokeVerifiable` で「モック化したコマンドが実際に呼ばれたか」を一括確認できます。

```powershell
It "バックアップと通知が両方実行される" {
    Mock Backup-Files    {} -Verifiable
    Mock Send-Notification {} -Verifiable

    Invoke-BackupPipeline -Source "C:\Data"

    # Verifiable モックが全て呼ばれたことを確認
    Should -InvokeVerifiable
}
```

---

## 9. よくある落とし穴

### 落とし穴 1：Mock は Describe/Context の中に書く

```powershell
# 悪い例：Describe の外に書くと動作しない
Mock Get-Date { return [DateTime]"2026-01-01" }

Describe "テスト" {
    It "日付が固定される" {
        (Get-Date).Year | Should -Be 2026   # 動かない
    }
}
```

```powershell
# 良い例：Describe の中（または BeforeAll の中）に書く
Describe "テスト" {
    BeforeAll {
        Mock Get-Date { return [DateTime]"2026-01-01" }
    }

    It "日付が固定される" {
        (Get-Date).Year | Should -Be 2026   # 成功
    }
}
```

### 落とし穴 2：ドットソーシングしないとモックが効かない

モックが効くのは**同じスコープに読み込まれた関数**だけです。

```powershell
BeforeAll {
    # ドットソーシングしていないと Write-Log がモック化されない
    Import-Module "./MyModule.psm1"   # モジュールとして読み込む場合は -ModuleName が必要
}

Mock Write-Log {}   # 効かない

# 正しい対応:
Mock Write-Log {} -ModuleName MyModule
```

### 落とし穴 3：Should -Invoke は It の中に書く

```powershell
# 悪い例：BeforeAll に書くと常に 0 回になる
BeforeAll {
    Mock Send-MailMessage {}
    Should -Invoke Send-MailMessage -Times 1   # ここではまだ呼ばれていない
}

# 良い例：It の中でテスト後に検証する
It "メールが送信される" {
    Send-AlertMail -Message "test"
    Should -Invoke Send-MailMessage -Times 1   # 呼ばれた後に検証
}
```

---

## 10. まとめ

```
Mock コマンド名 { 代替コード }
    → 指定したコマンドを偽物に差し替える

Mock コマンド名 { 代替コード } -ParameterFilter { 条件 }
    → 特定の引数のときだけ偽物に差し替える

Should -Invoke コマンド名 -Times N
    → N 回呼ばれたことを確認

Should -Invoke コマンド名 -Times N -ParameterFilter { 条件 }
    → 特定の引数で N 回呼ばれたことを確認

Mock コマンド名 { throw "エラー" }
    → 外部サービスのエラーをシミュレート

Mock コマンド名 {} -ModuleName モジュール名
    → モジュール内のコマンドをモック化
```

---

## 練習問題

### 問題 1
次のコードで `Should -Invoke` が正しい場所にありません。何が問題で、どう直せばよいですか？

```powershell
Describe "ファイル削除のテスト" {
    BeforeAll {
        . $PSScriptRoot/FileCleanup.ps1
        Mock Remove-Item {}
    }

    BeforeEach {
        Invoke-Cleanup -Path "C:\Temp"
        Should -Invoke Remove-Item -Times 1
    }

    It "削除が実行される" {
        $true | Should -BeTrue
    }
}
```

### 問題 2
以下の関数のテストを書いてください。`Invoke-RestMethod` をモックして、実際のHTTP通信なしにテストしてください。

```powershell
# WeatherService.ps1
function Get-WeatherReport {
    param([string]$City)

    try {
        $response = Invoke-RestMethod -Uri "https://api.weather.example.com/v1/$City"
        return [PSCustomObject]@{
            City        = $City
            Temperature = $response.temperature
            Condition   = $response.condition
            Success     = $true
        }
    }
    catch {
        return [PSCustomObject]@{
            City        = $City
            Temperature = $null
            Condition   = $null
            Success     = $false
            Error       = $_.Exception.Message
        }
    }
}
```

テストで確認すること：
1. 正常時：`City` プロパティが渡した都市名と一致する
2. 正常時：`Success` が `$true` である
3. 正常時：`Invoke-RestMethod` が1回だけ呼ばれる
4. 正常時：正しいURLで呼ばれる（`-ParameterFilter` を使う）
5. API エラー時：`Success` が `$false` である
6. API エラー時：`Error` プロパティにエラーメッセージが入る

### 問題 3（考察）
モックを使いすぎると「テストが通るのにスクリプトが実際には動かない」という問題が起きることがあります。どのような場面でモックを使い、どのような場面では使わないべきかを考えてみてください。

---

## 練習問題の解答

### 問題 1 の解答

**問題点：** `Should -Invoke` が `BeforeEach` の中にある。`BeforeEach` は各 `It` の**前**に実行されるため、`Invoke-Cleanup` を呼んだ直後に検証しているように見えるが、`It` が実行される前のタイミングであり、さらに次の `It` の `BeforeEach` では前回の呼び出し数が累積してしまう。`Should -Invoke` は `It` の中で書くべき。

```powershell
# 修正後
Describe "ファイル削除のテスト" {
    BeforeAll {
        . $PSScriptRoot/FileCleanup.ps1
        Mock Remove-Item {}
    }

    It "削除が実行される" {
        Invoke-Cleanup -Path "C:\Temp"
        Should -Invoke Remove-Item -Times 1   # It の中で検証する
    }
}
```

### 問題 2 の解答

```powershell
# WeatherService.Tests.ps1
BeforeAll {
    . $PSScriptRoot/WeatherService.ps1
}

Describe "Get-WeatherReport" {

    Context "API が正常に応答するとき" {
        BeforeAll {
            Mock Invoke-RestMethod {
                return [PSCustomObject]@{
                    temperature = 22.5
                    condition   = "Sunny"
                }
            }
        }

        It "City プロパティが引数と一致する" {
            $result = Get-WeatherReport -City "Tokyo"
            $result.City | Should -Be "Tokyo"
        }

        It "Success が true である" {
            $result = Get-WeatherReport -City "Tokyo"
            $result.Success | Should -BeTrue
        }

        It "Invoke-RestMethod が1回だけ呼ばれる" {
            Get-WeatherReport -City "Tokyo"
            Should -Invoke Invoke-RestMethod -Times 1
        }

        It "正しい URL で呼ばれる" {
            Get-WeatherReport -City "Tokyo"
            Should -Invoke Invoke-RestMethod -Times 1 -ParameterFilter {
                $Uri -eq "https://api.weather.example.com/v1/Tokyo"
            }
        }
    }

    Context "API がエラーを返すとき" {
        BeforeAll {
            Mock Invoke-RestMethod { throw "503 Service Unavailable" }
        }

        It "Success が false である" {
            $result = Get-WeatherReport -City "Tokyo"
            $result.Success | Should -BeFalse
        }

        It "Error プロパティにエラーメッセージが入る" {
            $result = Get-WeatherReport -City "Tokyo"
            $result.Error | Should -Match "503"
        }
    }
}
```

### 問題 3 の解答

**モックを使うべき場面：**
- メール送信・HTTP API・Active Directory など**外部システムへの副作用**がある処理
- テスト環境に存在しない**インフラリソース**（特定のサーバー、DB）への依存
- 時刻（`Get-Date`）やランダム値など**非決定的な値**を固定したいとき
- エラーや特定の状態を**意図的にシミュレート**したいとき

**モックを使わないべき場面：**
- テスト対象のロジックそのものをモックしてしまうと、何もテストしていないのと同じになる
- `$TestDrive` などで**実際のファイル操作をテスト環境内で実行できる**場合（モックより実際の動作をテストした方が信頼性が高い）
- モックが複雑になりすぎて**テスト自体のメンテナンスコストが上がる**場合

バランスの原則：「テスト対象の**ロジック**は本物で動かし、テスト対象の**外側の依存**をモックにする」

---

**次のステップ:** Phase 5では、ファイル操作・レジストリ・外部コマンドなどの実務パターンを総合的に学びます。

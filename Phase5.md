# Phase 5: 実務パターン

## 1. このフェーズの目的

Phase 0〜4で学んだ知識を統合し、実務でよく遭遇するシナリオに対応できるようにします。
扱うテーマは以下の3つです。

| テーマ | 実務での例 |
|---|---|
| ファイル操作 | ログローテーション、設定ファイルの読み書き、CSVの変換 |
| レジストリ | インストール確認、設定値の読み書き、環境チェック |
| 外部コマンド | ping、robocopy、外部CLIツールの呼び出し |

それぞれ「テスト対象のスクリプト」→「テスト設計の考え方」→「テストコード」の順で学びます。

---

## 2. パターン 1：ファイル操作

### 2-1. テスト対象

```powershell
# LogRotator.ps1

function Invoke-LogRotation {
    param(
        [string]$LogDirectory,
        [int]$RetentionDays = 30,
        [string]$ArchiveDirectory
    )

    if (-not (Test-Path $LogDirectory)) {
        throw "ログディレクトリが見つかりません: $LogDirectory"
    }

    $cutoffDate = (Get-Date).AddDays(-$RetentionDays)
    $oldLogs = Get-ChildItem -Path $LogDirectory -Filter "*.log" |
               Where-Object { $_.LastWriteTime -lt $cutoffDate }

    $result = [PSCustomObject]@{
        ArchivedCount = 0
        DeletedCount  = 0
        Errors        = @()
    }

    foreach ($log in $oldLogs) {
        try {
            if ($ArchiveDirectory) {
                if (-not (Test-Path $ArchiveDirectory)) {
                    New-Item -Path $ArchiveDirectory -ItemType Directory | Out-Null
                }
                Move-Item -Path $log.FullName -Destination $ArchiveDirectory
                $result.ArchivedCount++
            } else {
                Remove-Item -Path $log.FullName
                $result.DeletedCount++
            }
        }
        catch {
            $result.Errors += $log.Name
        }
    }

    return $result
}
```

### 2-2. テスト設計の考え方

ファイル操作テストでは **`$TestDrive`** を最大活用します。
モックを使わず実際のファイルシステム操作でテストすることで、より信頼性の高いテストになります。

```
テストすべき境界条件:
  ・ログディレクトリが存在しない → 例外
  ・古いログが 0 件のとき → 何もしない
  ・古いログがあるとき → アーカイブ or 削除される
  ・アーカイブディレクトリが存在しない → 自動作成される
  ・ファイルの日付による絞り込みが正しいか
```

### 2-3. テストコード

```powershell
# LogRotator.Tests.ps1

BeforeAll {
    . $PSScriptRoot/LogRotator.ps1
}

Describe "Invoke-LogRotation" {

    BeforeAll {
        $script:logDir     = Join-Path $TestDrive "logs"
        $script:archiveDir = Join-Path $TestDrive "archive"
        New-Item -Path $script:logDir -ItemType Directory
    }

    BeforeEach {
        # 各テストの前にログディレクトリをリセット
        Get-ChildItem $script:logDir | Remove-Item
        if (Test-Path $script:archiveDir) {
            Remove-Item $script:archiveDir -Recurse
        }
    }

    Context "ログディレクトリが存在しないとき" {
        It "例外を投げる" {
            { Invoke-LogRotation -LogDirectory (Join-Path $TestDrive "nonexistent") } |
                Should -Throw "*ログディレクトリが見つかりません*"
        }
    }

    Context "古いログが存在するとき（削除モード）" {
        BeforeEach {
            # 35日前のログファイルを作成し、LastWriteTime を書き換える
            $oldFile = Join-Path $script:logDir "old.log"
            Set-Content -Path $oldFile -Value "古いログ"
            (Get-Item $oldFile).LastWriteTime = (Get-Date).AddDays(-35)

            # 最新のログファイル（削除されないはず）
            $newFile = Join-Path $script:logDir "new.log"
            Set-Content -Path $newFile -Value "新しいログ"
        }

        It "古いログファイルが削除される" {
            Invoke-LogRotation -LogDirectory $script:logDir -RetentionDays 30
            Join-Path $script:logDir "old.log" | Should -Not -Exist
        }

        It "新しいログファイルは残る" {
            Invoke-LogRotation -LogDirectory $script:logDir -RetentionDays 30
            Join-Path $script:logDir "new.log" | Should -Exist
        }

        It "DeletedCount が 1 になる" {
            $result = Invoke-LogRotation -LogDirectory $script:logDir -RetentionDays 30
            $result.DeletedCount | Should -Be 1
        }

        It "ArchivedCount は 0 のまま" {
            $result = Invoke-LogRotation -LogDirectory $script:logDir -RetentionDays 30
            $result.ArchivedCount | Should -Be 0
        }
    }

    Context "古いログが存在するとき（アーカイブモード）" {
        BeforeEach {
            $oldFile = Join-Path $script:logDir "archive_target.log"
            Set-Content -Path $oldFile -Value "アーカイブ対象"
            (Get-Item $oldFile).LastWriteTime = (Get-Date).AddDays(-40)
        }

        It "アーカイブディレクトリが自動作成される" {
            Invoke-LogRotation -LogDirectory $script:logDir `
                               -RetentionDays 30 `
                               -ArchiveDirectory $script:archiveDir
            $script:archiveDir | Should -Exist
        }

        It "古いログがアーカイブに移動する" {
            Invoke-LogRotation -LogDirectory $script:logDir `
                               -RetentionDays 30 `
                               -ArchiveDirectory $script:archiveDir
            Join-Path $script:archiveDir "archive_target.log" | Should -Exist
        }

        It "元のディレクトリからは消える" {
            Invoke-LogRotation -LogDirectory $script:logDir `
                               -RetentionDays 30 `
                               -ArchiveDirectory $script:archiveDir
            Join-Path $script:logDir "archive_target.log" | Should -Not -Exist
        }

        It "ArchivedCount が 1 になる" {
            $result = Invoke-LogRotation -LogDirectory $script:logDir `
                                         -RetentionDays 30 `
                                         -ArchiveDirectory $script:archiveDir
            $result.ArchivedCount | Should -Be 1
        }
    }

    Context "保持期間内のログしかないとき" {
        BeforeEach {
            $recentFile = Join-Path $script:logDir "recent.log"
            Set-Content -Path $recentFile -Value "最近のログ"
            # LastWriteTime は今日のまま（デフォルト）
        }

        It "何も削除されない" {
            $result = Invoke-LogRotation -LogDirectory $script:logDir -RetentionDays 30
            $result.DeletedCount  | Should -Be 0
            $result.ArchivedCount | Should -Be 0
        }

        It "ファイルが残る" {
            Invoke-LogRotation -LogDirectory $script:logDir -RetentionDays 30
            Join-Path $script:logDir "recent.log" | Should -Exist
        }
    }
}
```

---

## 3. パターン 2：レジストリ

### 3-1. テスト対象

```powershell
# AppInstallChecker.ps1

function Get-InstalledAppInfo {
    param([string]$AppName)

    $registryPaths = @(
        "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall",
        "HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall",
        "HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall"
    )

    foreach ($path in $registryPaths) {
        $app = Get-ChildItem -Path $path -ErrorAction SilentlyContinue |
               Get-ItemProperty |
               Where-Object { $_.DisplayName -like "*$AppName*" } |
               Select-Object -First 1

        if ($app) {
            return [PSCustomObject]@{
                DisplayName    = $app.DisplayName
                Version        = $app.DisplayVersion
                Publisher      = $app.Publisher
                InstallDate    = $app.InstallDate
                UninstallString = $app.UninstallString
                Found          = $true
            }
        }
    }

    return [PSCustomObject]@{ Found = $false }
}

function Set-AppRegistrySetting {
    param(
        [string]$KeyPath,
        [string]$ValueName,
        [string]$Value
    )

    if (-not (Test-Path $KeyPath)) {
        New-Item -Path $KeyPath -Force | Out-Null
    }
    Set-ItemProperty -Path $KeyPath -Name $ValueName -Value $Value
}
```

### 3-2. テスト設計の考え方

レジストリ操作テストの戦略は2つあります。

**戦略 A：モックで隔離する**（本番レジストリに触らない）
```
利点: 高速・安全・環境非依存
欠点: 実際のレジストリ動作を検証できない
適用: Get-ChildItem/Get-ItemProperty など読み取り系
```

**戦略 B：$TestRegistry を使う**（Pesterの仮想レジストリ）
```
利点: 実際のレジストリ操作をテストできる
欠点: TestRegistry は HKCU:\Software\Pester 配下にマップされる
適用: Set-ItemProperty など書き込み系
```

### 3-3. テストコード（モック戦略）

```powershell
# AppInstallChecker.Tests.ps1

BeforeAll {
    . $PSScriptRoot/AppInstallChecker.ps1
}

Describe "Get-InstalledAppInfo" {

    Context "アプリが見つかるとき" {
        BeforeAll {
            # Get-ChildItem と Get-ItemProperty をモック化
            Mock Get-ChildItem {
                # レジストリキーオブジェクトの簡易モック
                return [PSCustomObject]@{ PSPath = "HKLM:\...\{FAKE-GUID}" }
            }

            Mock Get-ItemProperty {
                return [PSCustomObject]@{
                    DisplayName     = "MyApp 2.0.1"
                    DisplayVersion  = "2.0.1"
                    Publisher       = "My Company"
                    InstallDate     = "20260101"
                    UninstallString = "C:\Program Files\MyApp\uninstall.exe"
                }
            }
        }

        It "Found が true である" {
            $result = Get-InstalledAppInfo -AppName "MyApp"
            $result.Found | Should -BeTrue
        }

        It "DisplayName にアプリ名が含まれる" {
            $result = Get-InstalledAppInfo -AppName "MyApp"
            $result.DisplayName | Should -BeLike "*MyApp*"
        }

        It "Version が返る" {
            $result = Get-InstalledAppInfo -AppName "MyApp"
            $result.Version | Should -Not -BeNullOrEmpty
        }
    }

    Context "アプリが見つからないとき" {
        BeforeAll {
            Mock Get-ChildItem { return @() }
            Mock Get-ItemProperty { return @() }
        }

        It "Found が false である" {
            $result = Get-InstalledAppInfo -AppName "存在しないアプリ"
            $result.Found | Should -BeFalse
        }
    }
}

Describe "Set-AppRegistrySetting" {

    BeforeAll {
        # レジストリ書き込みをモック化（実際のレジストリを変更しない）
        Mock Test-Path      { return $false }
        Mock New-Item       {}
        Mock Set-ItemProperty {}
    }

    It "キーが存在しない場合 New-Item が呼ばれる" {
        Set-AppRegistrySetting -KeyPath "HKCU:\Test\Key" `
                               -ValueName "Setting" -Value "On"
        Should -Invoke New-Item -Times 1
    }

    It "Set-ItemProperty が正しい引数で呼ばれる" {
        Set-AppRegistrySetting -KeyPath "HKCU:\Test\Key" `
                               -ValueName "Setting" -Value "On"
        Should -Invoke Set-ItemProperty -Times 1 -ParameterFilter {
            $Name -eq "Setting" -and $Value -eq "On"
        }
    }

    Context "キーが既に存在するとき" {
        BeforeAll {
            Mock Test-Path { return $true }
        }

        It "New-Item は呼ばれない" {
            Set-AppRegistrySetting -KeyPath "HKCU:\Test\Key" `
                                   -ValueName "Setting" -Value "On"
            Should -Invoke New-Item -Times 0
        }
    }
}
```

---

## 4. パターン 3：外部コマンド

### 4-1. テスト対象

```powershell
# NetworkDiagnostics.ps1

function Test-NetworkConnectivity {
    param(
        [string[]]$Targets,
        [int]$TimeoutMs = 1000
    )

    $results = @()

    foreach ($target in $Targets) {
        try {
            $output = & ping -n 1 -w $TimeoutMs $target 2>&1
            $success = $LASTEXITCODE -eq 0

            $results += [PSCustomObject]@{
                Target  = $target
                Success = $success
                Output  = $output -join "`n"
            }
        }
        catch {
            $results += [PSCustomObject]@{
                Target  = $target
                Success = $false
                Output  = $_.Exception.Message
            }
        }
    }

    return $results
}

function Invoke-RobocopyBackup {
    param(
        [string]$Source,
        [string]$Destination,
        [string[]]$Options = @("/MIR", "/R:3", "/W:5")
    )

    if (-not (Test-Path $Source)) {
        throw "コピー元が存在しません: $Source"
    }

    $args = @($Source, $Destination) + $Options
    $output = & robocopy @args 2>&1
    $exitCode = $LASTEXITCODE

    # robocopy の終了コード: 0-7 は成功、8以上はエラー
    return [PSCustomObject]@{
        ExitCode = $exitCode
        Success  = $exitCode -lt 8
        Output   = $output -join "`n"
    }
}
```

### 4-2. テスト設計の考え方

外部コマンドのテストには **`&` 演算子の呼び出しをモック化する**のが定石です。
PowerShellでは外部コマンドの実行を関数でラップし、その関数をモックします。

```
直接外部コマンドをモックする方法はない（ping や robocopy はモックできない）
→ 解決策: & 演算子の呼び出しを内部ヘルパー関数でラップし、それをモックする
```

### 4-3. 外部コマンドをラップして分離する

```powershell
# NetworkDiagnostics.ps1 （改良版）

function Invoke-PingCommand {
    param([string]$Target, [int]$TimeoutMs)
    & ping -n 1 -w $TimeoutMs $Target 2>&1
    return $LASTEXITCODE
}

function Test-NetworkConnectivity {
    param(
        [string[]]$Targets,
        [int]$TimeoutMs = 1000
    )

    $results = @()

    foreach ($target in $Targets) {
        $output   = Invoke-PingCommand -Target $target -TimeoutMs $TimeoutMs
        $exitCode = $output[-1]   # Invoke-PingCommand の最後の戻り値が終了コード

        # 実際の実装では LASTEXITCODE を使う
        $success = ($LASTEXITCODE -eq 0)

        $results += [PSCustomObject]@{
            Target  = $target
            Success = $success
            Output  = ($output | Where-Object { $_ -isnot [int] }) -join "`n"
        }
    }

    return $results
}
```

### 4-4. テストコード

```powershell
# NetworkDiagnostics.Tests.ps1

BeforeAll {
    . $PSScriptRoot/NetworkDiagnostics.ps1
}

Describe "Test-NetworkConnectivity" {

    Context "全ターゲットに疎通できるとき" {
        BeforeAll {
            # ping をモック化（成功応答）
            Mock Invoke-PingCommand {
                $script:LASTEXITCODE = 0
                return @("Pinging $Target", "Reply from 192.168.1.1")
            }
        }

        It "全ターゲットの Success が true" {
            $results = Test-NetworkConnectivity -Targets @("192.168.1.1", "192.168.1.2")
            $results | ForEach-Object {
                $_.Success | Should -BeTrue
            }
        }

        It "返ってくる結果の数がターゲット数と一致する" {
            $results = Test-NetworkConnectivity -Targets @("host1", "host2", "host3")
            $results | Should -HaveCount 3
        }

        It "Target プロパティに渡したホスト名が入る" {
            $results = Test-NetworkConnectivity -Targets @("myserver")
            $results[0].Target | Should -Be "myserver"
        }
    }

    Context "疎通できないターゲットがあるとき" {
        BeforeAll {
            Mock Invoke-PingCommand {
                $script:LASTEXITCODE = 1
                return @("Request timed out.")
            }
        }

        It "Success が false になる" {
            $results = Test-NetworkConnectivity -Targets @("10.0.0.99")
            $results[0].Success | Should -BeFalse
        }
    }
}

Describe "Invoke-RobocopyBackup" {

    BeforeAll {
        $script:srcDir  = Join-Path $TestDrive "source"
        $script:dstDir  = Join-Path $TestDrive "dest"
        New-Item -Path $script:srcDir -ItemType Directory
    }

    Context "コピー元が存在するとき" {
        BeforeAll {
            # robocopy をモック化（終了コード 1 = ファイルコピー成功）
            Mock robocopy {
                $global:LASTEXITCODE = 1
                return "robocopy の出力（モック）"
            }
        }

        It "Success が true である（終了コード 1）" {
            $result = Invoke-RobocopyBackup -Source $script:srcDir `
                                            -Destination $script:dstDir
            $result.Success | Should -BeTrue
        }

        It "ExitCode が返る" {
            $result = Invoke-RobocopyBackup -Source $script:srcDir `
                                            -Destination $script:dstDir
            $result.ExitCode | Should -BeOfType [int]
        }
    }

    Context "コピー元が存在しないとき" {
        It "例外を投げる" {
            { Invoke-RobocopyBackup -Source (Join-Path $TestDrive "ghost") `
                                    -Destination $script:dstDir } |
                Should -Throw "*コピー元が存在しません*"
        }
    }

    Context "robocopy がエラーを返すとき（終了コード 8以上）" {
        BeforeAll {
            Mock robocopy {
                $global:LASTEXITCODE = 8
                return "ERROR: アクセス拒否"
            }
        }

        It "Success が false である" {
            $result = Invoke-RobocopyBackup -Source $script:srcDir `
                                            -Destination $script:dstDir
            $result.Success | Should -BeFalse
        }
    }
}
```

---

## 5. 実務で使えるテクニック集

### 5-1. TestCases：パラメータ化テスト

同じロジックを複数のパラメータで検証するときに使います。重複した `It` を削減できます。

```powershell
Describe "ConvertTo-FileSizeString" {
    BeforeAll {
        function ConvertTo-FileSizeString {
            param([long]$Bytes)
            if ($Bytes -ge 1GB) { return "{0:N1} GB" -f ($Bytes / 1GB) }
            if ($Bytes -ge 1MB) { return "{0:N1} MB" -f ($Bytes / 1MB) }
            if ($Bytes -ge 1KB) { return "{0:N1} KB" -f ($Bytes / 1KB) }
            return "$Bytes B"
        }
    }

    $testCases = @(
        @{ Bytes = 500;         Expected = "500 B"   }
        @{ Bytes = 2048;        Expected = "2.0 KB"  }
        @{ Bytes = 3145728;     Expected = "3.0 MB"  }
        @{ Bytes = 1073741824;  Expected = "1.0 GB"  }
    )

    It "<Bytes> バイトは <Expected> になる" -TestCases $testCases {
        $result = ConvertTo-FileSizeString -Bytes $Bytes
        $result | Should -Be $Expected
    }
}
```

**出力例：**
```
[+] 500 バイトは 500 B になる
[+] 2048 バイトは 2.0 KB になる
[+] 3145728 バイトは 3.0 MB になる
[+] 1073741824 バイトは 1.0 GB になる
```

### 5-2. -Tag：テストのタグ分類と選択実行

```powershell
Describe "ユーザー管理" {
    It "ユーザーを作成できる" -Tag "unit" {
        # ...
    }

    It "Active Directory に同期される" -Tag "integration", "ad" {
        # ...
    }
}
```

```powershell
# タグを指定して実行
Invoke-Pester -Path "." -Tag "unit"          # unit タグのみ実行
Invoke-Pester -Path "." -ExcludeTag "ad"     # ad タグを除外して実行
```

### 5-3. Invoke-Pester の設定：PesterConfiguration

細かい実行設定をオブジェクトで管理できます。CI/CD環境での利用に適しています。

```powershell
$config = New-PesterConfiguration
$config.Run.Path         = "./tests"
$config.Output.Verbosity = "Detailed"
$config.TestResult.Enabled    = $true
$config.TestResult.OutputPath = "./TestResults.xml"   # JUnit形式で出力
$config.Filter.Tag       = @("unit")

Invoke-Pester -Configuration $config
```

### 5-4. $PSBoundParameters を使ったテスト

関数に渡されたパラメータを確認するパターンです。

```powershell
function Backup-Database {
    param(
        [string]$Server,
        [string]$Database,
        [switch]$Compress
    )

    $backupArgs = @{
        Server   = $Server
        Database = $Database
    }
    if ($Compress) { $backupArgs['Compress'] = $true }

    Invoke-DatabaseBackup @backupArgs
}
```

```powershell
It "Compress スイッチが渡されると圧縮バックアップが実行される" {
    Mock Invoke-DatabaseBackup {}

    Backup-Database -Server "sql01" -Database "MyDB" -Compress

    Should -Invoke Invoke-DatabaseBackup -Times 1 -ParameterFilter {
        $Compress -eq $true
    }
}
```

---

## 6. 総合サンプル：実務スクリプト一式

### 6-1. テスト対象（サーバー環境チェックスクリプト）

```powershell
# ServerHealthCheck.ps1

function Get-ServerHealth {
    param(
        [string]$ServerName,
        [int]$DiskThresholdPercent = 85,
        [int]$CpuThresholdPercent  = 90
    )

    $issues = @()

    # ディスク使用率チェック
    $disk = Get-PSDrive -Name C -ErrorAction SilentlyContinue
    if ($disk) {
        $diskUsed = [math]::Round($disk.Used / ($disk.Used + $disk.Free) * 100, 1)
        if ($diskUsed -ge $DiskThresholdPercent) {
            $issues += "ディスク使用率が ${diskUsed}% です（閾値: ${DiskThresholdPercent}%）"
        }
    }

    # CPU使用率チェック
    $cpu = Get-CimInstance -ClassName Win32_Processor -ErrorAction SilentlyContinue |
           Measure-Object -Property LoadPercentage -Average |
           Select-Object -ExpandProperty Average

    if ($null -ne $cpu -and $cpu -ge $CpuThresholdPercent) {
        $issues += "CPU使用率が ${cpu}% です（閾値: ${CpuThresholdPercent}%）"
    }

    # 重要サービスの稼働確認
    $criticalServices = @("Spooler", "W32Time", "wuauserv")
    foreach ($svcName in $criticalServices) {
        $svc = Get-Service -Name $svcName -ErrorAction SilentlyContinue
        if ($svc -and $svc.Status -ne "Running") {
            $issues += "サービス '$svcName' が停止しています"
        }
    }

    return [PSCustomObject]@{
        ServerName = $ServerName
        Healthy    = $issues.Count -eq 0
        Issues     = $issues
        CheckedAt  = Get-Date
    }
}
```

### 6-2. テストコード

```powershell
# ServerHealthCheck.Tests.ps1

BeforeAll {
    . $PSScriptRoot/ServerHealthCheck.ps1
}

Describe "Get-ServerHealth" {

    BeforeAll {
        # 共通モック：Get-Date を固定
        Mock Get-Date { return [DateTime]"2026-01-15 09:00:00" }
    }

    Context "全て正常なとき" {
        BeforeAll {
            Mock Get-PSDrive {
                [PSCustomObject]@{ Name = "C"; Used = 50GB; Free = 50GB }
            }
            Mock Get-CimInstance {
                [PSCustomObject]@{ LoadPercentage = 30 }
            }
            Mock Get-Service {
                [PSCustomObject]@{ Name = $Name; Status = "Running" }
            }
        }

        It "Healthy が true である" {
            $result = Get-ServerHealth -ServerName "WebServer01"
            $result.Healthy | Should -BeTrue
        }

        It "Issues が空である" {
            $result = Get-ServerHealth -ServerName "WebServer01"
            $result.Issues | Should -HaveCount 0
        }

        It "ServerName が正しい" {
            $result = Get-ServerHealth -ServerName "WebServer01"
            $result.ServerName | Should -Be "WebServer01"
        }

        It "CheckedAt が固定日時である" {
            $result = Get-ServerHealth -ServerName "WebServer01"
            $result.CheckedAt | Should -Be ([DateTime]"2026-01-15 09:00:00")
        }
    }

    Context "ディスク使用率が閾値を超えているとき" {
        BeforeAll {
            Mock Get-PSDrive {
                [PSCustomObject]@{ Name = "C"; Used = 90GB; Free = 10GB }
            }
            Mock Get-CimInstance {
                [PSCustomObject]@{ LoadPercentage = 20 }
            }
            Mock Get-Service {
                [PSCustomObject]@{ Name = $Name; Status = "Running" }
            }
        }

        It "Healthy が false である" {
            $result = Get-ServerHealth -ServerName "DBServer01" -DiskThresholdPercent 85
            $result.Healthy | Should -BeFalse
        }

        It "Issues にディスク警告が含まれる" {
            $result = Get-ServerHealth -ServerName "DBServer01" -DiskThresholdPercent 85
            $result.Issues | Should -HaveCount 1
            $result.Issues[0] | Should -Match "ディスク使用率"
        }
    }

    Context "サービスが停止しているとき" {
        BeforeAll {
            Mock Get-PSDrive {
                [PSCustomObject]@{ Name = "C"; Used = 40GB; Free = 60GB }
            }
            Mock Get-CimInstance {
                [PSCustomObject]@{ LoadPercentage = 10 }
            }
            Mock Get-Service {
                # Spooler だけ停止している
                if ($Name -eq "Spooler") {
                    return [PSCustomObject]@{ Name = "Spooler"; Status = "Stopped" }
                }
                return [PSCustomObject]@{ Name = $Name; Status = "Running" }
            }
        }

        It "Healthy が false である" {
            $result = Get-ServerHealth -ServerName "PrintServer"
            $result.Healthy | Should -BeFalse
        }

        It "Issues にサービス停止の警告が含まれる" {
            $result = Get-ServerHealth -ServerName "PrintServer"
            $result.Issues | Should -HaveCount 1
            $result.Issues[0] | Should -Match "Spooler"
        }
    }

    Context "複数の問題があるとき" {
        BeforeAll {
            Mock Get-PSDrive {
                [PSCustomObject]@{ Name = "C"; Used = 95GB; Free = 5GB }
            }
            Mock Get-CimInstance {
                [PSCustomObject]@{ LoadPercentage = 95 }
            }
            Mock Get-Service {
                [PSCustomObject]@{ Name = $Name; Status = "Stopped" }
            }
        }

        It "全ての問題が Issues に含まれる" {
            $result = Get-ServerHealth -ServerName "TroubledServer" `
                                       -DiskThresholdPercent 85 `
                                       -CpuThresholdPercent 90
            # ディスク1件 + CPU1件 + サービス3件 = 計5件
            $result.Issues | Should -HaveCount 5
        }

        It "Healthy が false である" {
            $result = Get-ServerHealth -ServerName "TroubledServer"
            $result.Healthy | Should -BeFalse
        }
    }
}
```

---

## 7. テストの品質を上げるチェックリスト

実務でテストを書くときに確認したい観点です。

```
[ ] 正常系（ハッピーパス）のテストが書かれているか
[ ] 異常系（エラー、例外）のテストが書かれているか
[ ] 境界値（0件、1件、最大値、最小値）のテストが書かれているか
[ ] 外部依存（ファイル・レジストリ・外部コマンド）がモック化されているか
[ ] $TestDrive を使って実際のファイルシステムを汚染していないか
[ ] BeforeAll/BeforeEach の使い分けが適切か
[ ] It の説明文を読むだけで何を確認しているか分かるか
[ ] テストが互いに独立していて、実行順序に依存していないか
[ ] AfterAll/AfterEach で後片付けが行われているか
```

---

## 8. まとめ：フェーズ全体の振り返り

```
Phase 0: テストとは何か、Pesterの位置づけ
Phase 1: Describe / It / Should の骨格
Phase 2: Should マッチャーの全体像と使い分け
Phase 3: BeforeAll/AfterAll/BeforeEach/AfterEach によるセットアップ・ティアダウン
Phase 4: Mock による外部依存の切り離し、Should -Invoke による呼び出し検証
Phase 5: ファイル操作・レジストリ・外部コマンドの実務パターン
         TestCases によるパラメータ化、Tag による選択実行、PesterConfiguration
```

**次のアクション：**
- 手元にある実務スクリプトに対してテストを書いてみる
- CI/CDパイプラインにPesterを組み込む（GitHub Actions等）
- `Invoke-Pester -Configuration $config` でXMLレポートを出力し、テスト結果を可視化する

---

## 練習問題

### 問題 1：TestCases を使ったパラメータ化

以下の関数に対して `TestCases` を使ったテストを書いてください。

```powershell
function ConvertTo-NetworkClass {
    param([string]$IpAddress)

    $firstOctet = [int]($IpAddress -split "\.")[0]

    if ($firstOctet -le 127) { return "A" }
    if ($firstOctet -le 191) { return "B" }
    if ($firstOctet -le 223) { return "C" }
    return "D以上"
}
```

テストケース：
- `10.0.0.1` → `"A"`
- `127.0.0.1` → `"A"`
- `128.0.0.1` → `"B"`
- `191.255.255.0` → `"B"`
- `192.168.1.1` → `"C"`
- `223.255.255.0` → `"C"`
- `224.0.0.1` → `"D以上"`

### 問題 2：ファイル操作のテスト

以下の関数のテストを `$TestDrive` を使って書いてください。

```powershell
# ConfigManager.ps1
function Export-Config {
    param([hashtable]$Config, [string]$Path)

    $Config | ConvertTo-Json -Depth 5 | Set-Content -Path $Path -Encoding UTF8
}

function Import-Config {
    param([string]$Path)

    if (-not (Test-Path $Path)) {
        throw "設定ファイルが見つかりません: $Path"
    }
    return Get-Content -Path $Path -Raw | ConvertFrom-Json
}
```

テストで確認すること：
1. `Export-Config` を実行するとファイルが作成される
2. 作成されたファイルを `Import-Config` で読み込むと元のデータが復元できる
3. 存在しないパスを `Import-Config` に渡すと例外が投げられる

### 問題 3：モックと実ファイルの組み合わせ

実務では「一部はモック・一部は実ファイル」という組み合わせがよく出てきます。
Phase 4 と Phase 5 の知識を組み合わせて、以下の関数をテストしてください。

```powershell
# ReportGenerator.ps1
function New-DailyReport {
    param([string]$OutputDirectory)

    $today      = Get-Date
    $fileName   = "report_{0}.txt" -f $today.ToString("yyyyMMdd")
    $outputPath = Join-Path $OutputDirectory $fileName

    $content = "日次レポート`n生成日時: $today`nサーバー: $env:COMPUTERNAME"
    Set-Content -Path $outputPath -Value $content -Encoding UTF8

    return $outputPath
}
```

テストで確認すること：
1. 返り値のパスにファイルが存在する（`$TestDrive` を使う）
2. ファイル名に今日の日付が含まれる（`Get-Date` をモックして固定する）
3. ファイルの内容に "日次レポート" が含まれる

---

## 練習問題の解答

### 問題 1 の解答

```powershell
BeforeAll {
    . $PSScriptRoot/NetworkUtils.ps1
}

Describe "ConvertTo-NetworkClass" {

    $testCases = @(
        @{ IP = "10.0.0.1";       Expected = "A"    }
        @{ IP = "127.0.0.1";      Expected = "A"    }
        @{ IP = "128.0.0.1";      Expected = "B"    }
        @{ IP = "191.255.255.0";  Expected = "B"    }
        @{ IP = "192.168.1.1";    Expected = "C"    }
        @{ IP = "223.255.255.0";  Expected = "C"    }
        @{ IP = "224.0.0.1";      Expected = "D以上" }
    )

    It "<IP> はクラス <Expected> に分類される" -TestCases $testCases {
        $result = ConvertTo-NetworkClass -IpAddress $IP
        $result | Should -Be $Expected
    }
}
```

### 問題 2 の解答

```powershell
# ConfigManager.Tests.ps1
BeforeAll {
    . $PSScriptRoot/ConfigManager.ps1
}

Describe "Export-Config と Import-Config" {

    BeforeEach {
        $script:configPath = Join-Path $TestDrive "config.json"
        $script:testConfig = @{
            AppName = "MyApp"
            Version = "2.0"
            Debug   = $true
        }
    }

    It "Export-Config を実行するとファイルが作成される" {
        Export-Config -Config $script:testConfig -Path $script:configPath
        $script:configPath | Should -Exist
    }

    It "Export 後に Import すると元のデータが復元される" {
        Export-Config -Config $script:testConfig -Path $script:configPath
        $loaded = Import-Config -Path $script:configPath
        $loaded.AppName | Should -Be "MyApp"
        $loaded.Version | Should -Be "2.0"
    }

    It "存在しないパスを Import-Config に渡すと例外が投げられる" {
        { Import-Config -Path (Join-Path $TestDrive "ghost.json") } |
            Should -Throw "*設定ファイルが見つかりません*"
    }
}
```

### 問題 3 の解答

```powershell
# ReportGenerator.Tests.ps1
BeforeAll {
    . $PSScriptRoot/ReportGenerator.ps1

    # Get-Date をモックして日付を固定
    Mock Get-Date { return [DateTime]"2026-03-06 10:00:00" }
}

Describe "New-DailyReport" {

    BeforeEach {
        $script:outputDir = Join-Path $TestDrive "reports"
        New-Item -Path $script:outputDir -ItemType Directory -Force
    }

    It "返り値のパスにファイルが存在する" {
        $path = New-DailyReport -OutputDirectory $script:outputDir
        $path | Should -Exist
    }

    It "ファイル名に今日の日付 (20260306) が含まれる" {
        $path = New-DailyReport -OutputDirectory $script:outputDir
        $path | Should -BeLike "*20260306*"
    }

    It "ファイルの内容に '日次レポート' が含まれる" {
        $path = New-DailyReport -OutputDirectory $script:outputDir
        $path | Should -FileContentMatch "日次レポート"
    }
}
```

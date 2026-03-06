# Pester 学習教材

PowerShell未経験者がPesterを基礎から学び、実務スクリプトにテストを書けるようになることを目的とした教材です。

## 対象読者

- PowerShellは書けるが、テストを書いたことがない方
- Pesterを使ってみたいが何から始めればよいかわからない方

## 前提環境

- PowerShell 5.1 または PowerShell 7.x
- Pester 5.x

```powershell
# Pester のインストール
Install-Module -Name Pester -Force -SkipPublisherCheck

# バージョン確認
Import-Module Pester -PassThru | Select-Object -ExpandProperty Version
```

## フェーズ構成

| ファイル | テーマ | 内容 |
|---|---|---|
| [Phase0.md](./Phase0.md) | 概念・Why Pester？ | テストとは何か、Pesterの位置づけ、初めての実行 |
| [Phase1.md](./Phase1.md) | 基礎構文 | `Describe` / `It` / `Should`、テストの骨格 |
| [Phase2.md](./Phase2.md) | アサーション | `-Be` / `-Match` / `-Contain` など全マッチャー |
| [Phase3.md](./Phase3.md) | セットアップ／ティアダウン | `BeforeAll` / `AfterAll` / `BeforeEach` / `AfterEach` |
| [Phase4.md](./Phase4.md) | モック | `Mock` / `Should -Invoke`、外部依存の切り離し |
| [Phase5.md](./Phase5.md) | 実務パターン | ファイル操作・レジストリ・外部コマンド、TestCases |

## 教材の構成

各フェーズは以下の順で構成されています。

```
説明 → コード例 → 解説 → 練習問題 → 解答
```

## 推奨する進め方

1. Phase0 から順番に読む
2. コード例は実際に手元で動かす
3. 練習問題を自力で解いてから解答を確認する
4. Phase5 まで終えたら、手元にある実務スクリプトにテストを書いてみる

## テストの実行方法

```powershell
# 特定のファイルを実行
Invoke-Pester -Path "./MyScript.Tests.ps1" -Output Detailed

# ディレクトリ内の全テストを実行
Invoke-Pester -Path "./tests" -Output Detailed
```

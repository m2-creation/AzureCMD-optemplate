# Azure Monitor Agent ログ収集障害の包括的調査報告書

## 最も可能性の高い原因と即座に実施すべき対策

本番環境でのみAzure Monitor Agent (AMA) のログ収集が失敗している問題について、環境情報と調査結果から、**文字エンコーディングの不一致**と**Symantec SEPによるブロック**が最も可能性の高い原因と判明しました。

### 重要な発見事項

**Azure Monitor Agentは UTF-8またはASCIIエンコーディングのみサポート**しており[1][2]、日本語環境で一般的なShift-JIS（CP932）には対応していません[3]。SEPでの日本語文字化けは、システム全体のエンコーディング問題を示唆しており、これがログ収集失敗の根本原因である可能性が極めて高いです。

## 1. 文字エンコーディング問題の解決

### エンコーディング不一致の診断と修正

日本語環境でのLocale設定変更後、システムがShift-JISエンコーディングを使用している可能性があります。AMAは**UTF-8/ASCIIのみ対応**のため、以下の確認と修正が必要です：

**診断コマンド：**
```powershell
# 現在のシステムロケール確認
Get-WinSystemLocale
[System.Text.Encoding]::Default

# AMA設定ファイルのエンコーディング確認
$configPath = "C:\WindowsAzure\Resources\AMADataStore.$env:COMPUTERNAME\mcs\mcsconfig.latest.xml"
$bytes = [System.IO.File]::ReadAllBytes($configPath)
[System.Text.Encoding]::ASCII.GetString($bytes[0..2])
```

**修正手順：**
```powershell
# システム全体でUTF-8を使用する設定
# レジストリ編集（管理者権限必要）
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Nls\CodePage" -Name "ACP" -Value "65001"
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Nls\CodePage" -Name "OEMCP" -Value "65001"

# 環境変数の設定
[System.Environment]::SetEnvironmentVariable("PYTHONUTF8", "1", "Machine")

# サービス再起動
Restart-Service WindowsAzureGuestAgent
```

## 2. Symantec SEP による干渉の解決

### SEP除外設定の即時実施

SEPがAMAコンポーネントをブロックしている可能性が高いため[4]、以下の除外設定を**AMA再インストール前に**実施する必要があります：

**ディレクトリ除外：**
```
C:\WindowsAzure\Resources\AMADataStore.*\
C:\WindowsAzure\Logs\Plugins\Microsoft.Azure.Monitor.AzureMonitorWindowsAgent\
C:\Packages\Plugins\Microsoft.Azure.Monitor.AzureMonitorWindowsAgent\
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\AzureMonitorAgent
```

**プロセス除外：**
```
MonAgentCore.exe
MonAgentHost.exe
MonAgentManager.exe
MonAgentLauncher.exe
MetricsExtension.Native.exe
AMAExtHealthMonitor.exe
```

**ネットワーク通信許可：**
```
global.handler.control.monitor.azure.com:443
*.handler.control.monitor.azure.com:443
*.ods.opinsights.azure.com:443
*.oms.opinsights.azure.com:443
*.ingest.monitor.azure.com:443
management.azure.com:443
```[5]

### SEPファイアウォール設定
```powershell
# SEPでHTTPS検査を無効化（Azure Monitor通信用）
# SEP管理コンソールで以下のドメインをHTTPS検査除外リストに追加
# - *.monitor.azure.com
# - *.opinsights.azure.com
# - *.azure-automation.net
```

## 3. ネットワークとDNS設定の検証

### 本番環境固有のネットワーク問題診断

**接続性テスト実施：**
```powershell
# 必須エンドポイントへの接続確認
$endpoints = @(
    "global.handler.control.monitor.azure.com",
    "japaneast.handler.control.monitor.azure.com",  # リージョンに応じて変更
    "management.azure.com"
)

foreach ($endpoint in $endpoints) {
    $result = Test-NetConnection -ComputerName $endpoint -Port 443
    Write-Host "$endpoint : $($result.TcpTestSucceeded)"
}

# IMDS接続確認（重要）
$imdsTest = Invoke-RestMethod -Uri "http://169.254.169.254/metadata/instance?api-version=2021-01-01" -Headers @{Metadata="true"} -TimeoutSec 5
if ($imdsTest) { Write-Host "IMDS Access: OK" } else { Write-Host "IMDS Access: FAILED" }
```

### NSGルール確認
```powershell
# Azure PowerShellで本番環境のNSG確認
Get-AzNetworkSecurityGroup -ResourceGroupName <prod-rg> | 
    Select-Object Name, @{Name="OutboundRules";Expression={$_.SecurityRules | Where-Object Direction -eq "Outbound"}}
```

**必須のアウトバウンドルール：**
- ServiceTag: AzureMonitor, Port: 443, Protocol: TCP
- ServiceTag: AzureResourceManager, Port: 443, Protocol: TCP
- Destination: 169.254.169.254, Port: 80, Protocol: HTTP (IMDS用)

## 4. JP1との競合解決

### JP1とAMAの共存設定

JP1のシステム監視機能とAMAが競合する可能性があるため[7]、以下の調整を推奨：

**リソース競合の回避：**
```powershell
# サービス起動順序の調整
Set-Service -Name "JP1_Base" -StartupType Manual
Set-Service -Name "WindowsAzureGuestAgent" -StartupType Automatic -StartupDelayed

# JP1のポーリング間隔調整（AMAと重複しないよう）
# JP1設定ファイルで監視間隔を調整
```

**ポート競合の確認：**
```powershell
# 使用ポート確認
netstat -ano | findstr :443
netstat -ano | findstr :161  # JP1 SNMP
```

## 5. DCE/DCR設定の完全検証

### 本番環境のDCR再構成

**現在のDCR状態確認：**
```powershell
# DCR関連付け確認
Get-AzDataCollectionRuleAssociation -ResourceUri "/subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.Compute/virtualMachines/<vm-name>"

# DCR設定ファイル確認
$configPath = "C:\WindowsAzure\Resources\AMADataStore.$env:COMPUTERNAME\mcs\configchunks"
Get-ChildItem $configPath -Recurse
```

**DCR再作成手順：**
```bash
# Azure CLIでDCR再作成
az monitor data-collection rule delete --name <old-dcr> --resource-group <rg>

# 新規DCR作成（UTF-8対応設定）
az monitor data-collection rule create \
    --name "dcr-prod-utf8" \
    --resource-group <rg> \
    --location japaneast \
    --rule-file dcr-utf8-config.json
```

## 6. 段階的トラブルシューティング手順

### Phase 1: 基礎診断（5分）

```powershell
# AMA Troubleshooterの実行[6]
$troubleshooterPath = "C:\Packages\Plugins\Microsoft.Azure.Monitor.AzureMonitorWindowsAgent\*\Troubleshooter"
& "$troubleshooterPath\AgentTroubleshooter.exe" --ama

# プロセス確認
Get-Process -Name "MonAgentCore" -ErrorAction SilentlyContinue
if ($?) { "AMA Core: 実行中" } else { "AMA Core: 停止" }

# 設定ファイル存在確認
Test-Path "C:\WindowsAzure\Resources\AMADataStore.$env:COMPUTERNAME\mcs\mcsconfig.latest.xml"
```

### Phase 2: エンコーディング修正（10分）

```powershell
# ログファイルのエンコーディング変換
Get-ChildItem -Path "C:\Logs" -Filter "*.log" | ForEach-Object {
    $content = Get-Content $_.FullName -Encoding Default
    Set-Content -Path $_.FullName -Value $content -Encoding UTF8
}

# システムロケール設定確認と修正
$currentACP = (Get-ItemProperty "HKLM:\SYSTEM\CurrentControlSet\Control\Nls\CodePage").ACP
if ($currentACP -eq "932") {  # Shift-JIS
    Write-Warning "Shift-JIS detected. UTF-8 conversion required."
}
```

### Phase 3: SEP除外設定（15分）

SEP管理コンソールにアクセスし、前述の除外設定を全て適用。設定後、SEPサービスを再起動。

### Phase 4: AMA完全再インストール（20分）

```powershell
# 1. 既存AMA削除
Remove-AzVMExtension -ResourceGroupName <rg> -VMName <vm-name> -Name "AzureMonitorWindowsAgent" -Force

# 2. 残存ファイル削除
Remove-Item "C:\WindowsAzure\Resources\AMADataStore*" -Recurse -Force
Remove-Item "HKLM:\SOFTWARE\Microsoft\AzureMonitorAgent" -Recurse -Force

# 3. VM再起動
Restart-Computer -Force

# 4. AMA再インストール（システム割り当てマネージドID使用）[8]
Set-AzVMExtension -Name AzureMonitorWindowsAgent `
    -ExtensionType AzureMonitorWindowsAgent `
    -Publisher Microsoft.Azure.Monitor `
    -ResourceGroupName <rg> `
    -VMName <vm-name> `
    -Location japaneast `
    -EnableAutomaticUpgrade $true
```

### Phase 5: 検証（5分）

```kusto
// Log Analyticsで実行[9]
Heartbeat 
| where Category == "Azure Monitor Agent"
| where Computer == "<computer-name>"
| where TimeGenerated > ago(10m)
| project TimeGenerated, Computer, Version, Category
```

## 7. 開発環境と本番環境の差分確認

### 環境比較チェックリスト

```powershell
# 自動比較スクリプト
function Compare-AMAEnvironments {
    param($DevVM, $ProdVM)
    
    $comparison = @{}
    
    # エンコーディング確認
    $comparison["Encoding_Dev"] = Invoke-Command -ComputerName $DevVM -ScriptBlock { [System.Text.Encoding]::Default.EncodingName }
    $comparison["Encoding_Prod"] = Invoke-Command -ComputerName $ProdVM -ScriptBlock { [System.Text.Encoding]::Default.EncodingName }
    
    # SEP状態確認
    $comparison["SEP_Dev"] = Invoke-Command -ComputerName $DevVM -ScriptBlock { Get-Service -Name "SepMasterService" -ErrorAction SilentlyContinue }
    $comparison["SEP_Prod"] = Invoke-Command -ComputerName $ProdVM -ScriptBlock { Get-Service -Name "SepMasterService" -ErrorAction SilentlyContinue }
    
    # JP1状態確認
    $comparison["JP1_Dev"] = Invoke-Command -ComputerName $DevVM -ScriptBlock { Get-Service -Name "JP1*" -ErrorAction SilentlyContinue }
    $comparison["JP1_Prod"] = Invoke-Command -ComputerName $ProdVM -ScriptBlock { Get-Service -Name "JP1*" -ErrorAction SilentlyContinue }
    
    return $comparison
}
```

## 8. 本番稼働前の安定化対策

### 予防的措置の実装

**監視アラート設定：**
```kusto
// AMA停止検知アラート
Heartbeat
| where Category == "Azure Monitor Agent"
| summarize LastHeartbeat = max(TimeGenerated) by Computer
| where LastHeartbeat < ago(5m)
```

**定期的な健全性チェック：**
```powershell
# 日次実行スケジュールタスク
$action = New-ScheduledTaskAction -Execute "PowerShell.exe" -Argument "-File C:\Scripts\AMA-HealthCheck.ps1"
$trigger = New-ScheduledTaskTrigger -Daily -At "06:00"
Register-ScheduledTask -TaskName "AMA-HealthCheck" -Action $action -Trigger $trigger
```

**バックアップ構成の保持：**
```powershell
# DCR構成のバックアップ
$backupPath = "C:\AMA-Backup\$(Get-Date -Format 'yyyyMMdd')"
New-Item -ItemType Directory -Path $backupPath -Force
Copy-Item "C:\WindowsAzure\Resources\AMADataStore.$env:COMPUTERNAME\mcs\*" -Destination $backupPath -Recurse
```

## 9. エスカレーション基準

以下の場合は直ちにMicrosoftサポートへエスカレーション：
- 上記手順実施後も48時間以上ログ収集が開始されない
- AMAプロセスが頻繁にクラッシュする
- システムパフォーマンスに重大な影響が発生

## 結論

本番環境でのログ収集失敗は、**日本語環境特有の文字エンコーディング問題**と**Symantec SEPによる干渉**が主要因と考えられます。提示した手順を順番に実施することで、問題の解決が期待できます。特に重要なのは、SEP除外設定を**AMA再インストール前**に完了させること、およびシステム全体でUTF-8エンコーディングを使用するよう設定することです。

最優先で実施すべきアクション：
1. SEP除外設定の即時適用
2. 文字エンコーディングのUTF-8への統一
3. IMDS (169.254.169.254) への接続性確認
4. AMAの完全再インストール

これらの対策により、本番環境でも開発環境と同様にログ収集が正常に機能することが期待されます。

---

## 参考文献

[1] Microsoft Learn. "Collect text logs with the Log Analytics agent in Azure Monitor - Azure Monitor". https://learn.microsoft.com/en-us/azure/azure-monitor/agents/data-sources-custom-logs

[2] Microsoft Learn. "Collect text file from virtual machine with Azure Monitor - Azure Monitor". https://learn.microsoft.com/en-us/azure/azure-monitor/vm/data-collection-log-text

[3] Wikipedia. "Code page 932 (Microsoft Windows)". https://en.wikipedia.org/wiki/Code_page_932_(Microsoft_Windows)

[4] SoftActivity. "Setup SoftActivity with Symantec Endpoint Protection". https://www.softactivity.com/howto/activity-monitor-symantec-endpoint-protection/

[5] Microsoft Learn. "Azure Monitor Agent Network Configuration - Azure Monitor". https://learn.microsoft.com/en-us/azure/azure-monitor/agents/azure-monitor-agent-network-configuration

[6] Microsoft Learn. "Troubleshoot the Azure Monitor agent on Windows virtual machines and scale sets - Azure Monitor". https://learn.microsoft.com/en-us/azure/azure-monitor/agents/azure-monitor-agent-troubleshoot-windows-vm

[7] Hitachi. "JP1 Version 11 Network Management: Getting Started". https://itpfdoc.hitachi.co.jp/manuals/3021/30213A7110e/NNMK0004.HTM

[8] Microsoft Learn. "Install and Manage the Azure Monitor Agent - Azure Monitor". https://learn.microsoft.com/en-us/azure/azure-monitor/agents/azure-monitor-agent-manage

[9] Microsoft Learn. "Example log table queries for Heartbeat - Azure Monitor". https://learn.microsoft.com/en-us/azure/azure-monitor/reference/queries/heartbeat

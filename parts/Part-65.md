# Part 65: Cloud Security Automation

> **ระดับ**: 🔴 Advanced | **เวลา**: ~5 ชั่วโมง

---

## 1. Azure Security Center / Defender

```powershell
Install-Module Az.Security -Force

# Security score
Connect-AzAccount -ServicePrincipal -Credential $cred -TenantId $env:AZURE_TENANT_ID

$score = Get-AzSecuritySecureScore
$score | Select-Object DisplayName, Score, MaxScore |
    ForEach-Object { Write-Host "$($_.DisplayName): $($_.Score.Current)/$($_.MaxScore.Max)" }

# Security recommendations
$recs = Get-AzSecurityTask | Sort-Object RecommendationType
$high = $recs | Where-Object { $_.State -eq 'Active' } | Select-Object -First 20
$high | Select-Object RecommendationType, ResourceId | Format-Table -AutoSize

# Alerts
$alerts = Get-AzSecurityAlert | Where-Object { $_.AlertSeverity -in @('High','Critical') }
$alerts | ForEach-Object {
    Write-Warning "[$($_.AlertSeverity)] $($_.AlertDisplayName) - $($_.CompromisedEntity)"
}

# Auto-remediate open storage accounts
$openStorage = Get-AzStorageAccount | Where-Object {
    (Get-AzStorageAccountNetworkRuleSet -ResourceGroupName $_.ResourceGroupName -Name $_.StorageAccountName).DefaultAction -eq 'Allow'
}

foreach ($sa in $openStorage) {
    Write-Host "Restricting: $($sa.StorageAccountName)" -ForegroundColor Yellow
    Update-AzStorageAccountNetworkRuleSet `
        -ResourceGroupName $sa.ResourceGroupName `
        -Name $sa.StorageAccountName `
        -DefaultAction Deny
}
Write-Host "Secured $($openStorage.Count) storage accounts" -ForegroundColor Green
```

---

## 2. AWS Security Hub Automation

```powershell
Install-Module AWS.Tools.SecurityHub -Force
Install-Module AWS.Tools.IAM -Force

# Security Hub findings
$findings = Get-SHUBFinding -Filter @{
    RecordState  = @(@{Value='ACTIVE'; Comparison='EQUALS'})
    SeverityLabel= @(@{Value='CRITICAL'; Comparison='EQUALS'})
} -MaxResult 50

$findings.Findings | Select-Object `
    @{n='Title';      e={$_.Title.Substring(0,[Math]::Min(60,$_.Title.Length))}},
    @{n='Resource';   e={$_.Resources[0].Id.Split('/')[-1]}},
    @{n='Severity';   e={$_.Severity.Label}},
    @{n='Updated';    e={$_.UpdatedAt}} | Format-Table

# IAM security audit
function Get-IAMSecurityReport {
    # Users with no MFA
    $users    = Get-IAMUserList
    $noMFA    = $users | Where-Object {
        -not (Get-IAMMFADevice -UserName $_.UserName)
    }
    
    # Users with old access keys (>90 days)
    $oldKeys  = foreach ($u in $users) {
        Get-IAMAccessKey -UserName $u.UserName | Where-Object {
            ([datetime]::Now - $_.CreateDate).Days -gt 90
        } | ForEach-Object { 
            [PSCustomObject]@{ User=$u.UserName; KeyId=$_.AccessKeyId; AgeDays=([datetime]::Now - $_.CreateDate).Days }
        }
    }
    
    # Root account last activity
    $credReport = Get-IAMCredentialReport
    $root       = $credReport | Where-Object { $_.user -eq '<root_account>' }
    
    [PSCustomObject]@{
        UsersWithoutMFA  = $noMFA.Count
        OldAccessKeys    = $oldKeys.Count
        NoMFAUsers       = $noMFA.UserName
        OldKeyDetails    = $oldKeys
    }
}

$report = Get-IAMSecurityReport
Write-Host "Users without MFA: $($report.UsersWithoutMFA)" -ForegroundColor $(if ($report.UsersWithoutMFA -gt 0) {'Red'} else {'Green'})
$report.OldKeyDetails | Format-Table
```

---

## 3. CIS Benchmark Checks

```powershell
function Invoke-CISAzureChecks {
    param([string]$SubscriptionId)
    
    Set-AzContext -SubscriptionId $SubscriptionId
    $results = @()
    
    # CIS 2.1: Ensure that Microsoft Defender for Servers is 'On'
    $defender = Get-AzSecurityPricing -Name 'VirtualMachines'
    $results += [PSCustomObject]@{
        Control  = 'CIS 2.1'
        Check    = 'Defender for Servers'
        Status   = if ($defender.PricingTier -eq 'Standard') {'PASS'} else {'FAIL'}
        Value    = $defender.PricingTier
    }
    
    # CIS 3.1: Ensure Storage logging enabled
    $storageAccounts = Get-AzStorageAccount
    foreach ($sa in $storageAccounts) {
        $ctx     = $sa.Context
        $logging = Get-AzStorageServiceLoggingProperty -ServiceType Blob -Context $ctx
        $results += [PSCustomObject]@{
            Control  = 'CIS 3.7'
            Check    = "Storage logging: $($sa.StorageAccountName)"
            Status   = if ($logging.LoggingOperations -match 'Read|Write|Delete') {'PASS'} else {'FAIL'}
            Value    = $logging.LoggingOperations
        }
    }
    
    # CIS 4.1: Require SQL TDE
    $sqlServers = Get-AzSqlServer
    foreach ($srv in $sqlServers) {
        $dbs = Get-AzSqlDatabase -ServerName $srv.ServerName -ResourceGroupName $srv.ResourceGroupName
        foreach ($db in $dbs | Where-Object { $_.DatabaseName -ne 'master' }) {
            $tde = Get-AzSqlDatabaseTransparentDataEncryption `
                -ServerName $srv.ServerName -ResourceGroupName $srv.ResourceGroupName `
                -DatabaseName $db.DatabaseName
            $results += [PSCustomObject]@{
                Control  = 'CIS 4.1'
                Check    = "SQL TDE: $($db.DatabaseName)"
                Status   = if ($tde.State -eq 'Enabled') {'PASS'} else {'FAIL'}
                Value    = $tde.State
            }
        }
    }
    
    $pass = @($results | Where-Object { $_.Status -eq 'PASS' }).Count
    $fail = @($results | Where-Object { $_.Status -eq 'FAIL' }).Count
    Write-Host "CIS Score: $pass/$($results.Count) passed" -ForegroundColor $(if ($fail -eq 0) {'Green'} else {'Yellow'})
    return $results
}

$cisResults = Invoke-CISAzureChecks -SubscriptionId $env:AZURE_SUBSCRIPTION_ID
$cisResults | Where-Object { $_.Status -eq 'FAIL' } | Format-Table
```

---

**ก่อนหน้า ← [Part 64](Part-64.md) | ต่อไป → [Part 66: Compliance Automation](Part-66.md)**

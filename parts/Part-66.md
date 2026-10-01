# Part 66: Compliance Automation

> **ระดับ**: 🔴 Advanced | **เวลา**: ~4 ชั่วโมง

---

## 1. GDPR / PDPA Data Discovery

```powershell
# Scan files for PII patterns
function Find-PIIInFiles {
    param(
        [string]$Path,
        [string[]]$Extensions = @('*.txt','*.csv','*.json','*.xml','*.log'),
        [switch]$Recurse
    )
    
    $patterns = @{
        Email     = '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}'
        Phone     = '(\+66|0)[0-9]{8,9}'
        ThaiID    = '[0-9]{13}'
        CreditCard= '[0-9]{4}[\s-]?[0-9]{4}[\s-]?[0-9]{4}[\s-]?[0-9]{4}'
        IPAddress = '\b(?:(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.){3}(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\b'
    }
    
    $findings = @()
    $files = Get-ChildItem -Path $Path -Include $Extensions -Recurse:$Recurse
    
    foreach ($file in $files) {
        $content = Get-Content $file.FullName -Raw -ErrorAction SilentlyContinue
        if (-not $content) { continue }
        
        foreach ($patternName in $patterns.Keys) {
            $matches = [regex]::Matches($content, $patterns[$patternName])
            if ($matches.Count -gt 0) {
                $findings += [PSCustomObject]@{
                    File      = $file.FullName
                    PIIType   = $patternName
                    Count     = $matches.Count
                    Sample    = ($matches[0].Value).Substring(0, [Math]::Min(20, $matches[0].Value.Length)) + '...'
                    SizeKB    = [math]::Round($file.Length / 1KB, 1)
                    Modified  = $file.LastWriteTime
                }
            }
        }
    }
    
    Write-Host "Scanned $($files.Count) files, found PII in $(@($findings | Select-Object -Unique File).Count) files" -ForegroundColor Yellow
    return $findings
}

$pii = Find-PIIInFiles -Path 'C:\DataStore' -Recurse
$pii | Group-Object PIIType | Select-Object Name, Count | Format-Table
$pii | Sort-Object Count -Descending | Select-Object -First 10 | Format-Table
```

---

## 2. Audit Log Compliance Report

```powershell
function New-ComplianceReport {
    param(
        [string]$ReportTitle,
        [datetime]$StartDate = [datetime]::Now.AddDays(-30),
        [datetime]$EndDate   = [datetime]::Now,
        [string]$OutputPath  = "./compliance-$(Get-Date -Format yyyyMMdd).html"
    )
    
    # Gather audit data
    $logonEvents = Get-WinEvent -FilterHashtable @{
        LogName   = 'Security'
        Id        = @(4624, 4625, 4634)  # Logon, Failed logon, Logoff
        StartTime = $StartDate
        EndTime   = $EndDate
    } -ErrorAction SilentlyContinue
    
    $failedLogons = $logonEvents | Where-Object { $_.Id -eq 4625 } |
        ForEach-Object {
            $xml = [xml]$_.ToXml()
            $data = $xml.Event.EventData.Data
            [PSCustomObject]@{
                Time     = $_.TimeCreated
                User     = ($data | Where-Object { $_.Name -eq 'TargetUserName' }).'#text'
                IP       = ($data | Where-Object { $_.Name -eq 'IpAddress' }).'#text'
                Workst   = ($data | Where-Object { $_.Name -eq 'WorkstationName' }).'#text'
            }
        }
    
    # Policy changes
    $policyChanges = Get-WinEvent -FilterHashtable @{
        LogName   = 'Security'
        Id        = @(4719, 4817)  # Audit policy change, Object audit change
        StartTime = $StartDate
        EndTime   = $EndDate
    } -ErrorAction SilentlyContinue
    
    # Privileged access
    $privAccess = Get-WinEvent -FilterHashtable @{
        LogName   = 'Security'
        Id        = @(4672, 4673)  # Special privileges, Sensitive privilege use
        StartTime = $StartDate
        EndTime   = $EndDate
    } -ErrorAction SilentlyContinue
    
    # Generate HTML report
    $html = @"
<!DOCTYPE html>
<html>
<head>
    <title>$ReportTitle</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; }
        h1   { color: #2c3e50; }
        h2   { color: #34495e; border-bottom: 2px solid #3498db; }
        table{ border-collapse: collapse; width: 100%; margin: 10px 0; }
        th   { background: #3498db; color: white; padding: 8px; }
        td   { border: 1px solid #ddd; padding: 6px; }
        tr:nth-child(even) { background: #f2f2f2; }
        .metric { display: inline-block; padding: 10px 20px; margin: 5px;
                  background: #ecf0f1; border-radius: 5px; }
        .bad { color: #e74c3c; font-weight: bold; }
    </style>
</head>
<body>
<h1>$ReportTitle</h1>
<p>Period: $StartDate to $EndDate | Generated: $(Get-Date)</p>

<h2>Summary</h2>
<div class='metric'>Failed Logons: <span class='$(if ($failedLogons.Count -gt 50) {"bad"} else {""})'>$($failedLogons.Count)</span></div>
<div class='metric'>Policy Changes: $($policyChanges.Count)</div>
<div class='metric'>Privileged Access Events: $($privAccess.Count)</div>

<h2>Top Failed Logon Sources</h2>
<table>
<tr><th>IP Address</th><th>Count</th><th>Usernames Tried</th></tr>
$($failedLogons | Group-Object IP | Sort-Object Count -Desc | Select-Object -First 10 |
    ForEach-Object { "<tr><td>$($_.Name)</td><td>$($_.Count)</td><td>$(($_.Group.User | Sort-Object -Unique) -join ', ')</td></tr>" })
</table>
</body>
</html>
"@
    
    $html | Set-Content $OutputPath -Encoding UTF8
    Write-Host "Report saved: $OutputPath" -ForegroundColor Green
    return $OutputPath
}

New-ComplianceReport -ReportTitle 'Security Compliance Report Q4 2024' -OutputPath './reports/compliance.html'
```

---

## 3. Data Retention Policy Automation

```powershell
function Invoke-DataRetentionPolicy {
    param(
        [string]$DataPath,
        [hashtable]$RetentionDays = @{
            '*.log'   = 90
            '*.bak'   = 30
            '*.tmp'   = 7
            '*.csv'   = 365
        },
        [switch]$WhatIf
    )
    
    $deleted = 0
    $freed   = 0
    
    foreach ($pattern in $RetentionDays.Keys) {
        $maxAge  = $RetentionDays[$pattern]
        $cutoff  = [datetime]::Now.AddDays(-$maxAge)
        
        $files = Get-ChildItem -Path $DataPath -Filter $pattern -Recurse |
            Where-Object { $_.LastWriteTime -lt $cutoff -and -not $_.PSIsContainer }
        
        foreach ($file in $files) {
            if ($WhatIf) {
                Write-Host "[WhatIf] Would delete: $($file.FullName) (age: $(([datetime]::Now - $file.LastWriteTime).Days) days)" -ForegroundColor Yellow
            } else {
                $freed += $file.Length
                Remove-Item $file.FullName -Force
                $deleted++
            }
        }
    }
    
    $result = [PSCustomObject]@{
        FilesDeleted = $deleted
        SpaceFreedMB = [math]::Round($freed / 1MB, 2)
        WhatIf       = $WhatIf.IsPresent
    }
    Write-Host "Retention policy: $($result.FilesDeleted) files deleted, $($result.SpaceFreedMB) MB freed" -ForegroundColor Cyan
    return $result
}

# Preview first
Invoke-DataRetentionPolicy -DataPath 'D:\Data' -WhatIf
# Execute
Invoke-DataRetentionPolicy -DataPath 'D:\Data'
```

---

**ก่อนหน้า ← [Part 65](Part-65.md) | ต่อไป → [Part 67: System Hardening](Part-67.md)**

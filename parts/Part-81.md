# Part 81: Capstone 2 - SecOps Dashboard

> **ระดับ**: 🟠 World-Class | **เวลา**: ~6 ชั่วโมง
>
> ⚠️ ใช้สำหรับการตรวจสอบความปลอดภัย Blue Team และการป้องกันเท่านั้น

---

## โครงการ: Security Operations Center Dashboard

```powershell
# Full SOC Dashboard collecting real-time security metrics
function Start-SOCDashboard {
    param(
        [int]$RefreshInterval = 60,  # seconds
        [string]$LogPath      = './soc-dashboard.log',
        [string]$AlertWebhook = $env:TEAMS_WEBHOOK
    )
    
    Write-Host @"
=============================================
  SOC DASHBOARD - PowerShell Security Center
  Refresh: ${RefreshInterval}s | $(Get-Date)
=============================================
"@ -ForegroundColor Cyan
    
    while ($true) {
        $dashboard = @{}
        
        # --- Threat Indicators ---
        $dashboard.FailedLogins = @(Get-WinEvent -FilterHashtable @{
            LogName='Security'; Id=4625
            StartTime=(Get-Date).AddMinutes(-15)
        } -ErrorAction SilentlyContinue).Count
        
        $dashboard.NewServices = @(Get-WinEvent -FilterHashtable @{
            LogName='System'; Id=7045
            StartTime=(Get-Date).AddHours(-1)
        } -ErrorAction SilentlyContinue).Count
        
        $dashboard.PowerShellScriptBlocks = @(Get-WinEvent -FilterHashtable @{
            LogName='Microsoft-Windows-PowerShell/Operational'; Id=4104
            StartTime=(Get-Date).AddMinutes(-15)
        } -ErrorAction SilentlyContinue).Count
        
        # --- Active Threats ---
        $suspiciousConns = Get-NetTCPConnection -State Established |
            Where-Object { $_.RemotePort -notin @(80,443,22,53,3389,5985) }
        $dashboard.SuspiciousConnections = $suspiciousConns.Count
        
        # --- System Health ---
        $dashboard.AV = (Get-MpComputerStatus).AntivirusEnabled
        $dashboard.AVUpToDate = (Get-MpComputerStatus).AntivirusSignatureAge -lt 2
        $dashboard.Firewall = (Get-NetFirewallProfile | Where-Object { -not $_.Enabled }).Count -eq 0
        
        # --- Top Processes ---
        $top5Procs = Get-Process | Sort-Object CPU -Descending | Select-Object -First 5
        $dashboard.TopProcesses = $top5Procs | Select-Object Name, Id, @{n='CPU';e={[math]::Round($_.CPU,1)}}
        
        # --- Network Summary ---
        $connsByProcess = Get-NetTCPConnection -State Established |
            Group-Object OwningProcess | Sort-Object Count -Descending | Select-Object -First 5 |
            ForEach-Object {
                $proc = Get-Process -Id $_.Name -ErrorAction SilentlyContinue
                [PSCustomObject]@{ Process=$proc.Name; PID=$_.Name; Connections=$_.Count }
            }
        $dashboard.NetworkTopTalkers = $connsByProcess
        
        # Display
        Clear-Host
        Write-Host "=== SOC Dashboard: $(Get-Date -Format 'yyyy-MM-dd HH:mm:ss') ===" -ForegroundColor Cyan
        
        # Threat Level
        $threatScore = 0
        if ($dashboard.FailedLogins -gt 10)          { $threatScore += 30 }
        if ($dashboard.SuspiciousConnections -gt 5)  { $threatScore += 20 }
        if ($dashboard.NewServices -gt 2)            { $threatScore += 25 }
        if ($dashboard.PowerShellScriptBlocks -gt 50){ $threatScore += 25 }
        
        $threatLevel = if ($threatScore -ge 70) { 'CRITICAL' }
                       elseif ($threatScore -ge 40) { 'HIGH' }
                       elseif ($threatScore -ge 20) { 'MEDIUM' }
                       else { 'LOW' }
        
        $threatColor = @{LOW='Green';MEDIUM='Yellow';HIGH='DarkYellow';CRITICAL='Red'}[$threatLevel]
        Write-Host "Threat Level: $threatLevel (score: $threatScore)" -ForegroundColor $threatColor
        
        # Metrics
        Write-Host "`n[Authentication]" -ForegroundColor White
        Write-Host "  Failed logons (15m): $($dashboard.FailedLogins)" `
            -ForegroundColor $(if ($dashboard.FailedLogins -gt 10) {'Red'} else {'Green'})
        
        Write-Host "`n[System Changes]" -ForegroundColor White
        Write-Host "  New services (1h): $($dashboard.NewServices)" `
            -ForegroundColor $(if ($dashboard.NewServices -gt 0) {'Yellow'} else {'Green'})
        Write-Host "  PS script blocks (15m): $($dashboard.PowerShellScriptBlocks)"
        
        Write-Host "`n[Network]" -ForegroundColor White
        Write-Host "  Suspicious connections: $($dashboard.SuspiciousConnections)" `
            -ForegroundColor $(if ($dashboard.SuspiciousConnections -gt 5) {'Red'} else {'Green'})
        $dashboard.NetworkTopTalkers | Format-Table -AutoSize
        
        Write-Host "`n[Security Posture]" -ForegroundColor White
        Write-Host "  AV: $(if ($dashboard.AV) {'ENABLED'} else {'DISABLED'})" `
            -ForegroundColor $(if ($dashboard.AV) {'Green'} else {'Red'})
        Write-Host "  AV Signatures: $(if ($dashboard.AVUpToDate) {'CURRENT'} else {'OUTDATED'})" `
            -ForegroundColor $(if ($dashboard.AVUpToDate) {'Green'} else {'Red'})
        Write-Host "  Firewall: $(if ($dashboard.Firewall) {'ALL PROFILES ON'} else {'PROFILE(S) OFF'})" `
            -ForegroundColor $(if ($dashboard.Firewall) {'Green'} else {'Red'})
        
        # Alert on critical events
        if ($threatLevel -in @('HIGH','CRITICAL') -and $AlertWebhook) {
            $alert = @{
                attachments = @(@{
                    color   = 'danger'
                    title   = "[SOC ALERT] Threat Level: $threatLevel"
                    text    = "Score: $threatScore | Failed logins: $($dashboard.FailedLogins) | Suspicious conns: $($dashboard.SuspiciousConnections)"
                })
            } | ConvertTo-Json -Depth 5
            Invoke-RestMethod -Uri $AlertWebhook -Method POST -Body $alert -ContentType 'application/json' | Out-Null
        }
        
        # Log to file
        $logEntry = @{
            ts          = [datetime]::UtcNow.ToString('o')
            threatLevel = $threatLevel
            score       = $threatScore
            metrics     = @{
                failedLogins          = $dashboard.FailedLogins
                suspiciousConnections = $dashboard.SuspiciousConnections
                newServices           = $dashboard.NewServices
            }
        } | ConvertTo-Json -Compress
        $logEntry | Add-Content $LogPath
        
        Write-Host "`n[Next refresh in ${RefreshInterval}s - Ctrl+C to exit]"
        Start-Sleep $RefreshInterval
    }
}

Start-SOCDashboard -RefreshInterval 30
```

---

**ก่อนหน้า ← [Part 80](Part-80.md) | ต่อไป → [Part 82: Capstone 3 - Enterprise Framework](Part-82.md)**

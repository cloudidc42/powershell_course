# Part 67: Windows System Hardening

> **ระดับ**: 🔴 Advanced | **เวลา**: ~4 ชั่วโมง
>
> ⚠️ ทดสอบใน Lab หรือสภาพแวดล้อม Development เท่านั้น ดูแล Change Management ก่อน Production

---

## 1. CIS Benchmark Hardening Script

```powershell
#Requires -RunAsAdministrator

function Invoke-WindowsHardening {
    param([switch]$WhatIf, [string]$ReportPath = './hardening-report.json')
    
    $applied = @()
    $skipped = @()
    
    function Apply {
        param([string]$Name, [string]$CISControl, [scriptblock]$Action)
        try {
            if (-not $WhatIf) { & $Action }
            $script:applied += [PSCustomObject]@{ Control=$CISControl; Name=$Name; Status='Applied' }
            Write-Host "[OK] $Name" -ForegroundColor Green
        } catch {
            $script:skipped += [PSCustomObject]@{ Control=$CISControl; Name=$Name; Status="Failed: $_" }
            Write-Warning "[SKIP] $Name`: $_"
        }
    }
    
    # CIS 2.2.1 - Disable Guest account
    Apply 'Disable Guest Account' 'CIS 2.2.1' {
        Disable-LocalUser -Name 'Guest' -ErrorAction SilentlyContinue
    }
    
    # CIS 1.1.1 - Set minimum password length
    Apply 'Minimum Password Length 14' 'CIS 1.1.1' {
        net accounts /MINPWLEN:14 | Out-Null
    }
    
    # CIS 1.1.5 - Password complexity
    Apply 'Enable Password Complexity' 'CIS 1.1.5' {
        secedit /export /cfg "$env:TEMP\secpol.cfg" /quiet
        (Get-Content "$env:TEMP\secpol.cfg") -replace 'PasswordComplexity = 0', 'PasswordComplexity = 1' |
            Set-Content "$env:TEMP\secpol.cfg"
        secedit /configure /db secedit.sdb /cfg "$env:TEMP\secpol.cfg" /quiet
    }
    
    # CIS 2.3.1 - Audit account logon
    Apply 'Enable Logon Auditing' 'CIS 2.3.1' {
        auditpol /set /category:"Logon/Logoff" /success:enable /failure:enable | Out-Null
    }
    
    # CIS 2.3.2 - Enable object access auditing
    Apply 'Enable Object Access Auditing' 'CIS 2.3.2' {
        auditpol /set /subcategory:"File System" /success:enable /failure:enable | Out-Null
    }
    
    # Disable SMBv1
    Apply 'Disable SMBv1' 'CIS Network' {
        Set-SmbServerConfiguration -EnableSMB1Protocol $false -Force
    }
    
    # Disable NetBIOS
    Apply 'Disable NetBIOS over TCP/IP' 'CIS Network' {
        Get-WmiObject Win32_NetworkAdapterConfiguration | Where-Object { $_.TcpipNetbiosOptions -ne 2 } |
            ForEach-Object { $_.SetTcpipNetbios(2) | Out-Null }
    }
    
    # Enable Windows Defender
    Apply 'Enable Realtime Protection' 'CIS AV' {
        Set-MpPreference -DisableRealtimeMonitoring $false
        Set-MpPreference -DisableBehaviorMonitoring $false
        Set-MpPreference -DisableScriptScanning     $false
    }
    
    # Disable autorun
    Apply 'Disable AutoRun' 'CIS 18.8.3' {
        Set-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\Explorer' `
            -Name 'NoDriveTypeAutoRun' -Value 0xFF -Type DWord
    }
    
    # Restrict RDP access
    Apply 'Require NLA for RDP' 'CIS RDP' {
        Set-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' `
            -Name 'UserAuthentication' -Value 1 -Type DWord
    }
    
    # Save report
    @{ Applied=$applied; Skipped=$skipped; WhatIf=$WhatIf.IsPresent; Timestamp=[datetime]::Now } |
        ConvertTo-Json -Depth 5 | Set-Content $ReportPath
    
    Write-Host "`nHardening complete: $($applied.Count) applied, $($skipped.Count) skipped" -ForegroundColor Cyan
    return @{ Applied=$applied; Skipped=$skipped }
}

# Preview
Invoke-WindowsHardening -WhatIf

# Execute
Invoke-WindowsHardening -ReportPath './hardening-$(Get-Date -Format yyyyMMdd).json'
```

---

## 2. Service & Port Hardening

```powershell
# Disable unnecessary services
$servicesToDisable = @(
    'Fax',          # Fax
    'TabletInputService',  # Tablet Input
    'WMPNetworkSvc',       # Windows Media Player Network
    'XblAuthManager',      # Xbox Live
    'XblGameSave',         # Xbox Game Save
    'XboxNetApiSvc',       # Xbox Network
    'RemoteRegistry'       # Remote Registry
)

foreach ($svc in $servicesToDisable) {
    $service = Get-Service -Name $svc -ErrorAction SilentlyContinue
    if ($service) {
        Stop-Service   -Name $svc -Force -ErrorAction SilentlyContinue
        Set-Service    -Name $svc -StartupType Disabled
        Write-Host "Disabled: $svc" -ForegroundColor Yellow
    }
}

# Audit listening ports
$listeningPorts = Get-NetTCPConnection -State Listen |
    ForEach-Object {
        $proc = Get-Process -Id $_.OwningProcess -ErrorAction SilentlyContinue
        [PSCustomObject]@{
            Port    = $_.LocalPort
            Address = $_.LocalAddress
            PID     = $_.OwningProcess
            Process = $proc.Name
            Path    = $proc.Path
        }
    } | Sort-Object Port

# Highlight unexpected listeners
$knownPorts = @(80, 443, 135, 445, 3389, 5985, 5986)
$unexpected = $listeningPorts | Where-Object { $_.Port -notin $knownPorts }
if ($unexpected) {
    Write-Warning "Unexpected listening ports:"
    $unexpected | Format-Table Port, Process, Path
}
```

---

## 3. Baseline Configuration Snapshot

```powershell
function Save-SecurityBaseline {
    param([string]$OutputPath = './baseline.json')
    
    $baseline = @{
        Timestamp       = [datetime]::Now
        ComputerName    = $env:COMPUTERNAME
        OSVersion       = [System.Environment]::OSVersion.Version.ToString()
        InstalledPatches= (Get-HotFix | Sort-Object InstalledOn -Descending | Select-Object -First 10) |
                            Select-Object HotFixID, InstalledOn, Description
        Services        = Get-Service | Where-Object { $_.Status -eq 'Running' } | Select-Object Name, DisplayName, StartType
        ListeningPorts  = Get-NetTCPConnection -State Listen | Select-Object LocalAddress, LocalPort, OwningProcess
        LocalAdmins     = (Get-LocalGroupMember -Group 'Administrators').Name
        AutoStartItems  = Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run' |
                            Select-Object * -ExcludeProperty PS*
        DefenderStatus  = Get-MpComputerStatus | Select-Object AntivirusEnabled, RealTimeProtectionEnabled,
                            AntivirusSignatureLastUpdated, QuickScanAge
        WindowsFeatures = Get-WindowsOptionalFeature -Online | Where-Object { $_.State -eq 'Enabled' } |
                            Select-Object FeatureName
    }
    
    $baseline | ConvertTo-Json -Depth 8 | Set-Content $OutputPath
    Write-Host "Baseline saved to $OutputPath" -ForegroundColor Green
    return $baseline
}

function Compare-SecurityBaseline {
    param([string]$BaselinePath, [string]$OutputPath = './baseline-diff.json')
    
    $old     = Get-Content $BaselinePath | ConvertFrom-Json
    $current = Save-SecurityBaseline -OutputPath $null  # in-memory
    
    $diffs = @()
    
    # Check new admins
    $newAdmins = $current.LocalAdmins | Where-Object { $_ -notin $old.LocalAdmins }
    if ($newAdmins) { $diffs += @{ Type='NewLocalAdmin'; Items=$newAdmins } }
    
    # Check new services
    $oldSvcs = $old.Services.Name
    $newSvcs = $current.Services | Where-Object { $_.Name -notin $oldSvcs }
    if ($newSvcs) { $diffs += @{ Type='NewService'; Items=$newSvcs.Name } }
    
    # Check new autostart
    $oldAuto = $old.AutoStartItems.PSObject.Properties.Name
    $newAuto = $current.AutoStartItems.PSObject.Properties | Where-Object { $_.Name -notin $oldAuto }
    if ($newAuto) { $diffs += @{ Type='NewAutostart'; Items=$newAuto.Name } }
    
    if ($diffs) {
        Write-Warning "$($diffs.Count) security changes detected since baseline!"
        $diffs | ForEach-Object { Write-Warning "[$($_.Type)] $($_.Items -join ', ')" }
    } else {
        Write-Host "No security changes detected" -ForegroundColor Green
    }
    
    return $diffs
}

Save-SecurityBaseline -OutputPath './baseline-initial.json'
# Later...
Compare-SecurityBaseline -BaselinePath './baseline-initial.json'
```

---

**ก่อนหน้า ← [Part 66](Part-66.md) | ต่อไป → [Part 68: PowerShell Remoting Advanced](Part-68.md)**

# Part 93: Configuration Management ระดับ Enterprise

> **ระดับ**: 🟠 World-Class | **เวลา**: ~5 ชั่วโมง

---

## 1. PowerShell DSC (Desired State Configuration)

```powershell
# DSC Configuration for web server
Configuration WebServerConfig {
    param(
        [string[]]$Nodes = 'localhost',
        [string]$SiteName = 'DefaultWebSite',
        [string]$AppPoolName = 'MyAppPool'
    )
    
    Import-DscResource -ModuleName PSDesiredStateConfiguration
    Import-DscResource -ModuleName xWebAdministration
    
    Node $Nodes {
        # Ensure IIS is installed
        WindowsFeature IIS {
            Ensure = 'Present'
            Name   = 'Web-Server'
        }
        
        WindowsFeature IISManagement {
            Ensure    = 'Present'
            Name      = 'Web-Mgmt-Tools'
            DependsOn = '[WindowsFeature]IIS'
        }
        
        # Create app pool
        xWebAppPool MyAppPool {
            Ensure       = 'Present'
            Name         = $AppPoolName
            managedRuntimeVersion = 'v4.0'
            processModel = MSFT_xWebApplicationPoolProcessModel {
                userName = 'CORP\svc-webapp'
                password = (Get-Credential 'CORP\svc-webapp').Password
                logonType = 'SpecificUser'
            }
            DependsOn    = '[WindowsFeature]IIS'
        }
        
        # Create website
        xWebsite MySite {
            Ensure          = 'Present'
            Name            = $SiteName
            State           = 'Started'
            PhysicalPath    = 'C:\inetpub\myapp'
            ApplicationPool = $AppPoolName
            BindingInfo     = @(
                MSFT_xWebBindingInformation {
                    Protocol  = 'HTTPS'
                    Port      = 443
                    CertificateThumbprint = $env:SSL_THUMBPRINT
                    SslFlags  = 1
                }
            )
            DependsOn = '[xWebAppPool]MyAppPool'
        }
        
        # Firewall rules
        Script OpenPort443 {
            GetScript  = { @{ Result=(Get-NetFirewallRule -Name 'HTTPS-In' -ErrorAction SilentlyContinue)?.Enabled } }
            TestScript = { (Get-NetFirewallRule -Name 'HTTPS-In' -ErrorAction SilentlyContinue) -ne $null }
            SetScript  = { New-NetFirewallRule -Name 'HTTPS-In' -DisplayName 'HTTPS Inbound' -Direction Inbound -Protocol TCP -LocalPort 443 -Action Allow }
        }
    }
}

# Generate MOF files
WebServerConfig -Nodes @('WEB-01','WEB-02','WEB-03') -OutputPath './DSC'

# Apply
Get-ChildItem './DSC/*.mof' | ForEach-Object {
    $node = $_.BaseName
    Write-Host "Applying DSC to $node"
    Start-DscConfiguration -Path './DSC' -ComputerName $node -Wait -Verbose -Force
}

# Test compliance
$results = @('WEB-01','WEB-02','WEB-03') | ForEach-Object {
    $status = Test-DscConfiguration -ComputerName $_ -Detailed
    [PSCustomObject]@{
        Node        = $_
        Compliant   = $status.InDesiredState
        Violations  = @($status.ResourcesNotInDesiredState).Count
    }
}
$results | Format-Table
```

---

## 2. Configuration Drift Detection

```powershell
class DriftDetector {
    [hashtable]$Baseline
    [string]$BaselineFile
    
    DriftDetector([string]$baselineFile) {
        $this.BaselineFile = $baselineFile
    }
    
    [void] CaptureBaseline([string[]]$Targets) {
        $baseline = @{}
        
        foreach ($target in $Targets) {
            $baseline[$target] = $this.GetServerState($target)
        }
        
        $this.Baseline = $baseline
        $baseline | ConvertTo-Json -Depth 6 | Set-Content $this.BaselineFile
        Write-Host "Baseline captured for $($Targets.Count) servers" -ForegroundColor Green
    }
    
    [PSCustomObject[]] DetectDrift([string[]]$Targets) {
        if (-not $this.Baseline) {
            if (Test-Path $this.BaselineFile) {
                $this.Baseline = Get-Content $this.BaselineFile | ConvertFrom-Json -AsHashtable
            } else {
                throw 'No baseline found. Run CaptureBaseline first.'
            }
        }
        
        $drifts = @()
        foreach ($target in $Targets) {
            $current  = $this.GetServerState($target)
            $expected = $this.Baseline[$target]
            if (-not $expected) { continue }
            
            $changes = @()
            
            # Compare services
            $addedSvcs   = $current.Services.Keys | Where-Object { $_ -notin $expected.Services.Keys }
            $removedSvcs = $expected.Services.Keys | Where-Object { $_ -notin $current.Services.Keys }
            $changedSvcs = $current.Services.Keys | Where-Object {
                $_ -in $expected.Services.Keys -and
                $current.Services[$_].Status -ne $expected.Services[$_].Status
            }
            
            foreach ($s in $addedSvcs)   { $changes += "Service ADDED: $s" }
            foreach ($s in $removedSvcs) { $changes += "Service REMOVED: $s" }
            foreach ($s in $changedSvcs) {
                $changes += "Service CHANGED: $s ($($expected.Services[$s].Status) -> $($current.Services[$s].Status))"
            }
            
            # Compare registry settings
            foreach ($key in $expected.Registry.Keys) {
                if ($current.Registry[$key] -ne $expected.Registry[$key]) {
                    $changes += "Registry CHANGED: $key ($($expected.Registry[$key]) -> $($current.Registry[$key]))"
                }
            }
            
            if ($changes) {
                $drifts += [PSCustomObject]@{
                    Server  = $target
                    Changes = $changes
                    Count   = $changes.Count
                    Time    = [datetime]::Now
                }
            }
        }
        
        return $drifts
    }
    
    hidden [hashtable] GetServerState([string]$target) {
        $session = New-PSSession -ComputerName $target
        $state   = Invoke-Command -Session $session {
            $svcs = Get-Service | Where-Object { $_.StartType -eq 'Automatic' } |
                Group-Object Name -AsHashTable -AsString
            $reg  = @{
                'UAC'        = (Get-ItemPropertyValue 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System' EnableLUA)
                'RDP'        = (Get-ItemPropertyValue 'HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server' fDenyTSConnections)
                'WindowsUpdate'= (Get-ItemPropertyValue 'HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU' AUOptions -ErrorAction SilentlyContinue)
            }
            @{ Services=$svcs; Registry=$reg }
        }
        Remove-PSSession $session
        return $state
    }
}

$detector = [DriftDetector]::new('./server-baseline.json')
$servers  = @('SRV-APP-01','SRV-APP-02','SRV-WEB-01')

# Capture baseline
$detector.CaptureBaseline($servers)

# Later, detect drift
$drifts = $detector.DetectDrift($servers)
if ($drifts) {
    Write-Host "`n=== DRIFT DETECTED ===" -ForegroundColor Red
    $drifts | ForEach-Object {
        Write-Host "`nServer: $($_.Server)" -ForegroundColor Yellow
        $_.Changes | ForEach-Object { Write-Host "  - $_" }
    }
} else {
    Write-Host 'No drift detected - all servers in desired state' -ForegroundColor Green
}
```

---

**ก่อนหน้า ← [Part 92](Part-92.md) | ต่อไป → [Part 94: Final Capstone Projects](Part-94.md)**

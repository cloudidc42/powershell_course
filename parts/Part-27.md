# Part 27: DSC - Desired State Configuration

> **ระดับ**: 🟠 Advanced | **เวลา**: ~4 ชั่วโมง

---

## 1. DSC คืออะไร

```powershell
# DSC = "ระบบควบคุมว่า server ต้องอยู่ในสถานะที่ต้องการ"
# เหมือน Infrastructure as Code

# Configuration = function ที่คำนวณ MOF file
Configuration WebServerSetup {
    param(
        [string[]]$ComputerName = 'localhost'
    )
    
    # Import resources
    Import-DscResource -ModuleName PSDesiredStateConfiguration
    Import-DscResource -ModuleName xWebAdministration  # Install-Module xWebAdministration
    
    Node $ComputerName {
        # ติดตั้ง Windows features
        WindowsFeature WebServer {
            Name   = 'Web-Server'
            Ensure = 'Present'  # Absent = remove
        }
        
        WindowsFeature WebAsp {
            Name      = 'Web-Asp-Net45'
            Ensure    = 'Present'
            DependsOn = '[WindowsFeature]WebServer'
        }
        
        # สร้าง directory
        File WebRoot {
            DestinationPath = 'C:\inetpub\myapp'
            Type            = 'Directory'
            Ensure          = 'Present'
        }
        
        # เขียน config file
        File AppConfig {
            DestinationPath = 'C:\inetpub\myapp\web.config'
            Contents        = @'
<?xml version="1.0"?>
<configuration>
  <system.webServer></system.webServer>
</configuration>
'@
            Ensure    = 'Present'
            DependsOn = '[File]WebRoot'
        }
        
        # Service management
        Service Spooler {
            Name        = 'Spooler'
            State       = 'Running'
            StartupType = 'Automatic'
        }
        
        # Registry
        Registry MaxConnections {
            Key       = 'HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters'
            ValueName = 'TcpNumConnections'
            ValueData = '16777214'
            ValueType = 'DWord'
            Ensure    = 'Present'
        }
    }
}
```

---

## 2. ใช้ DSC

```powershell
# 1. สร้าง MOF file
WebServerSetup -ComputerName 'WebServer01'
# สร้างไฟล์: .\WebServerSetup\WebServer01.mof

# 2. Apply configuration (Push mode)
Start-DscConfiguration -Path '.\WebServerSetup' -Wait -Verbose

# Push ไปยัง remote server
Start-DscConfiguration -Path '.\WebServerSetup' \
    -ComputerName 'WebServer01' \
    -Credential (Get-Credential) \
    -Wait -Verbose

# 3. ตรวจสอบสถานะ
Get-DscConfiguration
Get-DscConfigurationStatus

# Test: จริงไหม?
Test-DscConfiguration
Test-DscConfiguration -ComputerName 'WebServer01'

# 4. Restoreน configuration ถ้าเปลี่ยนไป
Restore-DscConfiguration
```

---

## 3. Custom DSC Resource

```powershell
# สร้าง custom resource ด้วย class

[DscResource()]
class HostsFileEntry {
    [DscProperty(Key)]
    [string]$Hostname
    
    [DscProperty(Mandatory)]
    [string]$IPAddress
    
    [DscProperty()]
    [string]$Ensure = 'Present'
    
    [DscProperty()]
    [string]$Comment = ''
    
    hidden [string]$HostsFile = 'C:\Windows\System32\drivers\etc\hosts'
    
    [void] Set() {
        $lines = Get-Content $this.HostsFile
        $pattern = "^[^#]*\s+$([regex]::Escape($this.Hostname))\s*$"
        
        if ($this.Ensure -eq 'Present') {
            $entry = "$($this.IPAddress)    $($this.Hostname)"
            if ($this.Comment) { $entry += "    # $($this.Comment)" }
            
            # Remove existing, add new
            $newLines = $lines | Where-Object { $_ -notmatch $pattern }
            $newLines + $entry | Set-Content $this.HostsFile
        } else {
            $lines | Where-Object { $_ -notmatch $pattern } | Set-Content $this.HostsFile
        }
    }
    
    [bool] Test() {
        $lines = Get-Content $this.HostsFile
        $pattern = "^\s*$([regex]::Escape($this.IPAddress))\s+$([regex]::Escape($this.Hostname))"
        $exists  = $lines | Where-Object { $_ -match $pattern }
        
        if ($this.Ensure -eq 'Present') { return [bool]$exists }
        else { return ![bool]$exists }
    }
    
    [HostsFileEntry] Get() {
        $result = [HostsFileEntry]::new()
        $result.Hostname = $this.Hostname
        $lines   = Get-Content $this.HostsFile
        $pattern = "\s+$([regex]::Escape($this.Hostname))\s*$"
        $line    = $lines | Where-Object { $_ -match $pattern } | Select-Object -First 1
        
        if ($line) {
            $result.IPAddress = ($line -split '\s+')[0]
            $result.Ensure   = 'Present'
        } else {
            $result.IPAddress = ''
            $result.Ensure   = 'Absent'
        }
        return $result
    }
}
```

---

## 4. LCM Configuration

```powershell
# Local Configuration Manager
[DSCLocalConfigurationManager()]
Configuration LcmSettings {
    Node 'localhost' {
        Settings {
            RefreshMode           = 'Push'     # Push or Pull
            ConfigurationMode    = 'ApplyAndAutoCorrect'  # Apply, ApplyAndMonitor
            RebootNodeIfNeeded    = $false
            ActionAfterReboot     = 'ContinueConfiguration'
            AllowModuleOverwrite  = $true
            
            # Pull mode settings
            # RefreshFrequencyMins     = 30
            # ConfigurationModeFrequencyMins = 15
        }
    }
}

LcmSettings
Set-DscLocalConfigurationManager -Path '.\LcmSettings' -Verbose
Get-DscLocalConfigurationManager   # view current LCM
```

---

**ก่อนหน้า ← [Part 26](Part-26.md) | ต่อไป → [Part 28: Jobs & Scheduling](Part-28.md)**
